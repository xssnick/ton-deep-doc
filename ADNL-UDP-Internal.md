## ADNL UDP

ADNL over UDP is used by nodes and tone components to communicate with each other. It is a low-level protocol on top of which other, higher-level TON protocols such as DHT and RLDP operate. In this article, we will analyze how ADNL over UDP works for basic data exchange between nodes, tunneling and anonymizing traffic will be discussed in a separate article.

Unlike ADNL over TCP, in the UDP implementation, the handshake takes place in a different form, and an additional layer is used in the form of channels, but other principles are similar: keys are also generated based on our private key and the server's public key, which is known in advance from the config or received from other network nodes.

In the UDP version of ADNL, the connection is established simultaneously with the receipt of initial data from the client, the key is calculated and the creation of the channel is confirmed, if the initiator sent a message about the desire to create a channel.

Within one connection, several channels can be opened, they are needed for data isolation. Each channel has its own ID and encryption key. But usually, for basic interaction, only one channel is used, which is created along with the first request.

### Package arrangement and information exchange

##### First exchange of data
Let's analyze the initialization of the connection with the DHT node and obtaining a signed list of its addresses in order to understand how the data exchange works.

Find the node you like in [mainnet config](https://ton-blockchain.github.io/global.config.json), in the `dht.nodes` section. For example:
```json
{
  "@type": "dht.node",
  "id": {
    "@type": "pub.ed25519",
    "key": "fZnkoIAxrTd4xeBgVpZFRm5SvVvSx7eN3Vbe8c83YMk="
  },
  "addr_list": {
    "@type": "adnl.addressList",
    "addrs": [
      {
        "@type": "adnl.address.udp",
        "ip": 1091897261,
        "port": 15813
      }
    ],
    "version": 0,
    "reinit_date": 0,
    "priority": 0,
    "expire_at": 0
  },
  "version": -1,
  "signature": "cmaMrV/9wuaHOOyXYjoxBnckJktJqrQZ2i+YaY3ehIyiL3LkW81OQ91vm8zzsx1kwwadGZNzgq4hI4PCB/U5Dw=="
}
```

1. Let's take its key ED25519, `fZnkoIAxrTd4xeBgVpZFRm5SvVvSx7eN3Vbe8c83YMk`, decode from base64
2. Take its IP address `1091897261` and translate it into an understandable format using [service](https://www.browserling.com/tools/dec-to-ip), get `65.21.7.173`
3. Combine with the port, get `65.21.7.173:15813` and establish a UDP connection.

We want to open a channel for constant exchange of information with the node, and as the main task - to receive a list of subscribed addresses from it. To do this, we will generate 2 messages, the first - [create a channel](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/ton_api.tl#L129):
```
adnl.message.createChannel key:int256 date:int = adnl.Message
```
Here we have 2 parameters - key and date. As a date, we will specify the current unix timestamp. And for the key - we need to generate a new ED25519 private / public key pair, specifically for the channel, they will be used for initialization of [public encryption key](/ADNL-TCP-Liteserver.md#%D0%BF%D0%BE%D0%BB%D1%83%D1%87%D0%B5%D0%BD%D0%B8%D0%B5-%D0%BE%D0%B1%D1%89%D0%B5%D0%B3%D0%BE-%D0%BA%D0%BB%D1%8E%D1%87%D0%B0-%D0%BF%D0%BE-ecdh). We will specify our generated public key in the `key` parameter of the message, and just remember the private one for now.

Serialize the filled TL structure and get:
```
bbc373e6                                                         -- TL ID adnl.message.createChannel 
d59d8e3991be20b54dde8b78b3af18b379a62fa30e64af361c75452f6af019d7 -- key
555c8763                                                         -- date
```

Далее перейдем к нашему основному запросу - [получить список адресов](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/ton_api.tl#L198). Чтобы его выполнить, нам надо сначала сериализовать его TL структуру:
```
dht.getSignedAddressList = dht.Node
```
В нем нет параметров, поэтому просто сериализуем его. Получим `ed4879a9`

Далее, так как это запрос более высокого уровня, протокола DHT, нам нужно сначала обернуть его в `adnl.message.query` TL структуру:
```
adnl.message.query query_id:int256 query:bytes = adnl.Message
```
В качестве `query_id` генерируем случайные 32 байта, в качестве `query` используем наш основной запрос, [обернутый как массив байтов](/TL.md#кодирование-bytes-в-tl). Получаем:
```
7af98bb4                                                         -- TL ID adnl.message.query
d7be82afbc80516ebca39784b8e2209886a69601251571444514b7f17fcd8875 -- query_id
04 ed4879a9 000000                                               -- query
```
 
###### Собираем пакет

Весь обмен данными осуществляется с помощью пакетов, контент которых представляет из себя [TL структуру](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/ton_api.tl#L81):
```
adnl.packetContents 
  rand1:bytes                                     -- случайные 7 или 15 байт
  flags:#                                         -- битовые флаги, используются для определения наличия полей далее
  from:flags.0?PublicKey                          -- публичный ключ отправителя
  from_short:flags.1?adnl.id.short                -- айди отправителя
  message:flags.2?adnl.Message                    -- сообщение (используется, если оно одно)
  messages:flags.3?(vector adnl.Message)          -- сообщения (если их > 1)
  address:flags.4?adnl.addressList                -- список наших адресов
  priority_address:flags.5?adnl.addressList       -- приоритетный список наших адресов
  seqno:flags.6?long                              -- порядковый номер пакета
  confirm_seqno:flags.7?long                      -- порядковый номер последнего полученного пакета
  recv_addr_list_version:flags.8?int              -- версия адресов 
  recv_priority_addr_list_version:flags.9?int     -- версия приоритетных адресов
  reinit_date:flags.10?int                        -- дата реинициализации соединения (сброса счетчиков)
  dst_reinit_date:flags.10?int                    -- дата реинициализации соединения из последнего полученного пакета
  signature:flags.11?bytes                        -- подпись
  rand2:bytes                                     -- случайные 7 или 15 байт
        = adnl.PacketContents
        
```

После того, как мы сериализовали все сообщения, которые хотим отправить, мы можем приступать к сборке пакета. Пакеты для отправки в канал отличаются по содержанию от пакетов, которые отправляются до инициализации канала. Сначала разберем основной пакет, который используется для инициализации.

При первоначальном обмене информацией, вне канала, в качестве префикса к сериализованой структуре контента пакета идет публичный ключ сервера - 32 байта, наш публичный ключ - 32 байта, и sha256 хеш сериализованной TL структуры контента пакета - 32 байта. Контент пакета шифруется с помощью [общего ключа](/ADNL-TCP-Liteserver.md#%D0%BF%D0%BE%D0%BB%D1%83%D1%87%D0%B5%D0%BD%D0%B8%D0%B5-%D0%BE%D0%B1%D1%89%D0%B5%D0%B3%D0%BE-%D0%BA%D0%BB%D1%8E%D1%87%D0%B0-%D0%BF%D0%BE-ecdh), полученного из нашего приватного ключа и публичного ключа сервера.

Сериализуем нашу структуру контента пакета, и разберем побайтово:
```
89cd42d1                                                               -- TL ID adnl.packetContents
0f 4e0e7dd6d0c5646c204573bc47e567                                      -- rand1, 15 (0f) случайных байт
d9050000                                                               -- flags (0x05d9) -> 0b0000010111011001
                                                                       -- from (присутствует т.к 0й бит флага = 1)
c6b41348                                                                  -- TL ID pub.ed25519
   afc46336dd352049b366c7fd3fc1b143a518f0d02d9faef896cb0155488915d6       -- key:int256
                                                                       -- messages (присутствует т.к 3й бит флага = 1)
02000000                                                                  -- vector adnl.Message, размер 2 сообщения   
   bbc373e6                                                                  -- TL ID adnl.message.createChannel
   d59d8e3991be20b54dde8b78b3af18b379a62fa30e64af361c75452f6af019d7          -- key
   555c8763                                                                  -- date (дата создания)
   
   7af98bb4                                                                  -- TL ID [adnl.message.query](/)
   d7be82afbc80516ebca39784b8e2209886a69601251571444514b7f17fcd8875          -- query_id
   04 ed4879a9 000000                                                        -- query (bytes размер 4, паддинг 3)
                                                                       -- address (присутствует т.к 4й бит флага = 1), без TL ID т.к указан явно
00000000                                                                  -- addrs (пустой vector, т.к мы в режиме клиента и не имеем адреса на прослушке)
555c8763                                                                  -- version (обычно дата инициализации)
555c8763                                                                  -- reinit_date (обычно дата инициализации)
00000000                                                                  -- priority
00000000                                                                  -- expire_at

0100000000000000                                                       -- seqno (присутствует т.к 6й бит флага = 1)
0000000000000000                                                       -- confirm_seqno (присутствует т.к 7й бит флага = 1)
555c8763                                                               -- recv_addr_list_version (присутствует т.к 8й бит = 1, обычно дата инициализации)
555c8763                                                               -- reinit_date (присутствует т.к 10й бит флага = 1, обычно дата инициализации)
00000000                                                               -- dst_reinit_date (присутствует т.к 10й бит флага = 1)
0f 2b6a8c0509f85da9f3c7e11c86ba22                                      -- rand2, 15 (0f) случайных байт
```
После сериализации - нам нужно подписать полученный массив байтов нашим приватным ED25519 ключом клиента (не канала), который мы сгенерировали и запомнили до этого. После того, как мы получили подпись (размером 64 байта), нам нужно добавить ее в пакет, сериализуем его еще раз, но добавляем в флаг 11й бит, значащий наличие подписи:
```
89cd42d1                                                               -- TL ID adnl.packetContents
0f 4e0e7dd6d0c5646c204573bc47e567                                      -- rand1, 15 (0f) случайных байт
d90d0000                                                               -- flags (0x0dd9) -> 0b0000110111011001
                                                                       -- from (присутствует т.к 0й бит флага = 1)
c6b41348                                                                  -- TL ID pub.ed25519
   afc46336dd352049b366c7fd3fc1b143a518f0d02d9faef896cb0155488915d6       -- key:int256
                                                                       -- messages (присутствует т.к 3й бит флага = 1)
02000000                                                                  -- vector adnl.Message, размер 2 сообщения   
   bbc373e6                                                                  -- TL ID adnl.message.createChannel
   d59d8e3991be20b54dde8b78b3af18b379a62fa30e64af361c75452f6af019d7          -- key
   555c8763                                                                  -- date (дата создания)
   
   7af98bb4                                                                  -- TL ID adnl.message.query
   d7be82afbc80516ebca39784b8e2209886a69601251571444514b7f17fcd8875          -- query_id
   04 ed4879a9 000000                                                        -- query (bytes размер 4, паддинг 3)
                                                                       -- address (присутствует т.к 4й бит флага = 1), без TL ID т.к указан явно
00000000                                                                  -- addrs (пустой vector, т.к мы в режиме клиента и не имеем адреса на прослушке)
555c8763                                                                  -- version (обычно дата инициализации)
555c8763                                                                  -- reinit_date (обычно дата инициализации)
00000000                                                                  -- priority
00000000                                                                  -- expire_at

0100000000000000                                                       -- seqno (присутствует т.к 6й бит флага = 1)
0000000000000000                                                       -- confirm_seqno (присутствует т.к 7й бит флага = 1)
555c8763                                                               -- recv_addr_list_version (присутствует т.к 8й бит = 1, обычно дата инициализации)
555c8763                                                               -- reinit_date (присутствует т.к 10й бит флага = 1, обычно дата инициализации)
00000000                                                               -- dst_reinit_date (присутствует т.к 10й бит флага = 1)
40 b453fbcbd8e884586b464290fe07475ee0da9df0b8d191e41e44f8f42a63a710    -- signature (присутствует т.к 11й бит флага = 1), (bytes размер 64, падинг 3)
   341eefe8ffdc56de73db50a25989816dda17a4ac6c2f72f49804a97ff41df502    --
   000000                                                              --
0f 2b6a8c0509f85da9f3c7e11c86ba22                                      -- rand2, 15 (0f) случайных байт
```
Теперь у нас есть собранный, подписанный и сериализованный пакет, представляющий из себя массив байтов. Для последующей проверки его целостности получателем, нам нужно посчитать его sha256 хеш. Пусть для примера это будет `408a2a4ed623b25a2e2ba8bbe92d01a3b5dbd22c97525092ac3203ce4044dcd2`.

Теперь зашифруем контент нашего пакета шифром AES-CTR, с помощью [общего ключа](/ADNL-TCP-Liteserver.md#%D0%BF%D0%BE%D0%BB%D1%83%D1%87%D0%B5%D0%BD%D0%B8%D0%B5-%D0%BE%D0%B1%D1%89%D0%B5%D0%B3%D0%BE-%D0%BA%D0%BB%D1%8E%D1%87%D0%B0-%D0%BF%D0%BE-ecdh), полученного из нашего приватного ключа и публичного ключа сервера (не ключем канала).

Мы почти готовы к отправке, осталось [посчитать ID](/ADNL-TCP-Liteserver.md#получение-айди-ключа) ED25519 ключа сервера и соединить все вместе:
```
daa76538d99c79ea097a67086ec05acca12d1fefdbc9c96a76ab5a12e66c7ebb  -- ID ключа сервера
afc46336dd352049b366c7fd3fc1b143a518f0d02d9faef896cb0155488915d6  -- наш публичный ключ 
408a2a4ed623b25a2e2ba8bbe92d01a3b5dbd22c97525092ac3203ce4044dcd2  -- sha256 хеш контента (до шифрования)
...                                                               -- зашифрованное содержимое пакета
```
Теперь мы можем отправлять полученный пакет серверу по UDP, и ждать ответ. 

В ответ нам придет похожий по структуре пакет, но уже с другими сообщениями. Он будет состоять из:
```

68426d4906bafbd5fe25baf9e0608cf24fffa7eca0aece70765d64f61f82f005  -- ID нашего ключа
2d11e4a08031ad3778c5e060569645466e52bd1bd2c7b78ddd56def1cf3760c9  -- публичный ключ сервера, для общего ключа
f32fa6286d8ae61c0588b5a03873a220a3163cad2293a5dace5f03f06681e88a  -- sha256 хеш контента (до шифрования)
...                                                               -- зашифрованное содержимое пакета
```

Десериализация пакета от сервера происходит следующим образом:
1. Проверяем айди ключа из пакета, чтобы понять, что пакет для нас.
2. Используя публичный ключ сервера из пакета и наш приватный ключ, создаем общий ключ и дешифруем содержимое пакета
3. Сравниваем присланный нам sha256 хеш с хешом полученным от дешифрованых данных, должны совпасть
4. Начинаем десериализацию контента пакета, используя TL схему `adnl.packetContents`

Контент пакета будет выглядеть следующим образом:
```
89cd42d1                                                               -- TL ID adnl.packetContents
0f 985558683d58c9847b4013ec93ea28                                      -- rand1, 15 (0f) случайных байт
ca0d0000                                                               -- flags (0x0dca) -> 0b0000110111001010
daa76538d99c79ea097a67086ec05acca12d1fefdbc9c96a76ab5a12e66c7ebb       -- from_short (т.к 1й бит флага равен 1)
02000000                                                               -- messages (присутствует т.к 3й бит флага = 1)
   691ddd60                                                               -- TL ID adnl.message.confirmChannel 
   db19d5f297b2b0d76ef79be91ad3ae01d8d9f80fab8981d8ed0c9d67b92be4e3       -- key (публичный ключ канала сервера)
   d59d8e3991be20b54dde8b78b3af18b379a62fa30e64af361c75452f6af019d7       -- peer_key (наш публичный ключ канала)
   94848863                                                               -- date
   
   1684ac0f                                                               -- TL ID adnl.message.answer 
   d7be82afbc80516ebca39784b8e2209886a69601251571444514b7f17fcd8875       -- query_id
   90 48325384c6b413487d99e4a08031ad3778c5e060569645466e52bd5bd2c7b       -- answer (ответ на наш запрос, его содержание разберем в статье про DHT)
      78ddd56def1cf3760c901000000e7a60d67ad071541c53d0000ee354563ee       --
      35456300000000000000009484886340d46cc50450661a205ad47bacd318c       --
      65c8fd8e8f797a87884c1bad09a11c36669babb88f75eb83781c6957bc976       --
      6a234f65b9f6e7cc9b53500fbe2c44f3b3790f000000                        --
      000000                                                              --
0100000000000000                                                       -- seqno (присутствует т.к 6й бит флага = 1)
0100000000000000                                                       -- confirm_seqno (присутствует т.к 7й бит флага = 1)
94848863                                                               -- recv_addr_list_version (присутствует т.к 8й бит = 1, обычно дата инициализации)
ee354563                                                               -- reinit_date (присутствует т.к 10й бит флага = 1, обычно дата инициализации)
94848863                                                               -- dst_reinit_date (присутствует т.к 10й бит флага = 1)
40 5c26a2a05e584e9d20d11fb17538692137d1f7c0a1a3c97e609ee853ea9360ab6   -- signature (присутствует т.к 11й бит флага = 1), (bytes размер 64, падинг 3)
   d84263630fe02dfd41efb5cd965ce6496ac57f0e51281ab0fdce06e809c7901     --
   000000                                                              --
0f c3354d35749ffd088411599101deb2                                      -- rand2, 15 (0f) случайных байт
```
Сервер ответил нам двумя сообщениями: `adnl.message.confirmChannel` и `adnl.message.answer`. С `adnl.message.answer` все просто, это ответ на наш запрос `dht.getSignedAddressList`, его мы разберем в статье про DHT. 

Сфокусируемся на `adnl.message.confirmChannel`, оно значит, что сервер подтвердил создание канала и прислал нам свой публичный ключ канала. Теперь, имея наш приватный ключ канала и публичный ключ канала сервера, мы можем вычислить [общий ключ](/ADNL-TCP-Liteserver.md#%D0%BF%D0%BE%D0%BB%D1%83%D1%87%D0%B5%D0%BD%D0%B8%D0%B5-%D0%BE%D0%B1%D1%89%D0%B5%D0%B3%D0%BE-%D0%BA%D0%BB%D1%8E%D1%87%D0%B0-%D0%BF%D0%BE-ecdh).

Теперь, когда мы вычислили общий ключ канала, нам нужно сделать из него 2 ключа - один для шифрования исходящих сообщений, другой для дешифрования входящих. Сделать из него 2 ключа довольно просто, второй ключ равен общему ключу записанному в обратном порядке. Пример:
```
Общий ключ : AABB2233

Первый ключ: AABB2233
Второй ключ: 3322BBAA
```
Осталось определить какой ключ для чего использовать, мы можем это сделать, сравнив айди нашего публичного ключа канала с айди публичного ключа канала сервера, переведя их в числовой вид - uint256. Такой подход используется для того, чтобы и сервер, и клиент определили, какой ключ для чего им использовать. Если сервер использует первый ключ для шифрования, то с таким подходом клиент всегда будет использовать его для дешифровки. 

Условия использования такие:
```
Айди сервера меньше, чем наш айди:
Шифрование: Первый ключ
Дешифровка: Второй ключ

Айди сервера больше, чем наш айди:
Шифрование: Второй ключ
Дешифровка: Первый ключ

Если айди равны (почти невозможно):
Шифрование: Первый ключ
Дешифровка: Первый ключ
```
[[Пример реализации]](https://github.com/xssnick/tonutils-go/blob/udp-rldp-2/adnl/adnl.go#L502)

##### Обмен данныме в канале

Весь последующий обмен пакетами будет происходить внутри канала, и для шифрования будут использоваться новые ключи. Отправим тот же самый запрос `dht.getSignedAddressList`, но уже внутри свежесозданного канала, чтобы увидеть разницу.

Соберем контент пакета для канала, используя ту же структуру `adnl.packetContents`:
```
89cd42d1                                                               -- TL ID adnl.packetContents
0f c1fbe8c4ab8f8e733de83abac17915                                      -- rand1, 15 (0f) случайных байт
c4000000                                                               -- flags (0x00c4) -> 0b0000000011000100
                                                                       -- message (т.к 2й бит = 1)
7af98bb4                                                                  -- TL ID adnl.message.query
fe3c0f39a89917b7f393533d1d06b605b673ffae8bbfab210150fe9d29083c35          -- query_id
04 ed4879a9 000000                                                        -- query (наш dht.getSignedAddressList упакованный в bytes с падингом 3)
0200000000000000                                                       -- seqno (т.к 6й бит флага = 1), 2 тк это второе наше сообщение
0100000000000000                                                       -- confirm_seqno (7й бит флага = 1), 1 тк это последний полученный от сервера seqno
07 e4092842a8ae18                                                      -- rand2, 7 (07) случайных байт
```
Пакеты в канале довольно просты и состоят по сути из сиквенсов (seqno) и самих сообщений. 

После сериализации, как и в прошлый раз, мы вычисляем sha256 хеш от контента. Потом шифруем контент пакета с помощью ключа, предназначенного для исходящих пакетов канала. [Посчитаем](/ADNL-TCP-Liteserver.md#дополнительные-технические-детали-хендшейка) `pub.aes` ID нашего ключа шифрования исходящих сообщений, И собираем наш пакет:
```
bcd1cf47b9e657200ba21d94b822052cf553a548f51f539423c8139a83162180 -- ID нашего ключа шифрования исходящих сообщений
6185385aeee5faae7992eb350f26ba253e8c7c5fa1e3e1879d9a0666b9bd6080 -- sha256 хеш контента (до шифрования)
...                                                              -- зашифрованное содержимое пакета
```
Отправляем пакет по UDP и ждем ответ. В ответе мы получим пакет того же вида, что и отправили (те же поля), но уже с ответом на наш запрос `dht.getSignedAddressList`.

#### Другие типы сообщений
Для основной коммуникации используются сообщения типа `adnl.message.query` и `adnl.message.answer` которые мы разобрали выше, но для некоторых ситуаций возможны и использование других типов сообщений, которые мы разберем в этом разделе.

##### adnl.message.part
Сообщение этого типа представляет из себя кусок одного из других возможных типов сообщений, например `adnl.message.answer`. Передача таким методом используется когда сообщение слишком велико для передаче его в одной UDP датаграме. 
```
adnl.message.part 
hash:int256            -- sha256 хеш оригинального сообщения
total_size:int         -- размер оригинального сообщения
offset:int             -- смещение относительно начала оригинального сообщения
data:bytes             -- кусок данных оригинального сообщения
   = adnl.Message;
```
Таким образом, чтобы собрать оригинальное сообщение, нам нужно получить несколько партов и согласно оффсетам сложить их в единый массив байтов. А далее уже обработать как сообщение (согласно префиксу в этом массиве).

##### adnl.message.custom
```
adnl.message.custom data:bytes = adnl.Message;
```
Такие сообщения используются, когда логика на уровне выше не соответствует формату запрос-ответ, сообщения такого типа позволяют полностью вынести обработку на уровень выше, так как сообщение несет только массив байтов, без query_id и прочих полей. Сообщения такого типа используются, например в RLDP, так как там на множество запросов может быть всего один ответ и эта логика контролируется самим RLDP.

##### Заключение

Дальнейший обмен данными происходит на основе разобранной в этой статье логики, 
но содержание пакетов зависит уже от более высокоуровневых протоколов, таких как DHT и RLDP.
