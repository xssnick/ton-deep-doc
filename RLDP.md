## RLDP

RLDP - Reliable Large Datagram Protocol - это протокол, работающий поверх ADNL UDP, который служит для передачи больших данных и 
включает в себя Forward Error Correction (FEC) алгоритмы для замещения подтверждений получения пакетов другой стороной. Это позволяет более эффективно, хоть и с бОльшим потреблением трафика передавать данные между компонентами сети. 

RLDP используется практически повсеместно в инфраструктуре TON. Например, для скачивания блоков с других нод и передачи им данных,
для запросов к TON сайтам и в TON Storage.

#### Протокол

Для обмена данными RLDP использует следующие TL структуры:
```
fec.raptorQ data_size:int symbol_size:int symbols_count:int = fec.Type;
fec.roundRobin data_size:int symbol_size:int symbols_count:int = fec.Type;
fec.online data_size:int symbol_size:int symbols_count:int = fec.Type;

rldp.messagePart transfer_id:int256 fec_type:fec.Type part:int total_size:long seqno:int data:bytes = rldp.MessagePart;
rldp.confirm transfer_id:int256 part:int seqno:int = rldp.MessagePart;
rldp.complete transfer_id:int256 part:int = rldp.MessagePart;

rldp.message id:int256 data:bytes = rldp.Message;
rldp.query query_id:int256 max_answer_size:long timeout:int data:bytes = rldp.Message;
rldp.answer query_id:int256 data:bytes = rldp.Message;
```
Сериализованная структура оборачивается в TL схему `adnl.message.custom` и отправляется по ADNL UDP. Для передачи больших данных используются трансферы, генерируется случайный `transfer_id`, а сами данные обрабатываются FEC алгоритмом. Полученные кусочки заворачиваются в структуру `rldp.messagePart` и отправляются получателю до тех пор пока получатель не отправит нам `rldp.complete`. Когда получатель собрал кусочки `rldp.messagePart` необходимые для сборки полного сообщения, он соединяет их все вместе, декодирует с помощью FEC и полученный массив байтов десериализует в одну из стркутур `rldp.query` или `rldp.answer`, в зависимости от типа (tl id префикса).

### FEC

Допустимые Forward Error Correction алгоритмы для использования с RLDP - это RoundRobin, Online, и RaptorQ. 
Сейчас для обмена данными используется именно [RaptorQ](https://www.qualcomm.com/media/documents/files/raptorq-technical-overview.pdf).

##### RaptorQ
Суть RaptorQ в том, что данные делятся на так называемые символы - блоки, одинакового, заранее определенного размера. 

Из блоков создаются матрицы, и к ним применяются дискретные математические операции. Это позволяет создать практически бесконечное количество символов 
из одних и тех же данных. Все символы смешиваются, и, благодаря этому, получается восстановить потерянные пакеты без запроса дополнительных данных от сервера, при этом используя меньшее количество пакетов, чем это было бы, если бы мы отправляли одни и те же кусочки по кругу.

Сгенерированные символы отправляются получателю, пока он не скажет, что все данные получены и восстановлены путем применения тех же дискретных операций.

[[Моя реализация RaptorQ на Golang]](https://github.com/xssnick/tonutils-go/tree/udp-rldp-2/adnl/rldp/raptorq)

### RLDP-HTTP

Для взаимодействия с сайтами на TON используется HTTP, обернутый в RLDP. Хостер размещает свой сайт на любом HTTP вебсервере и рядом поднимает rldp-http-proxy. Все запросы из сети TON приходят по протоколу RLDP в прокси, а прокси уже пересобирает запрос в обычный HTTP и вызывает вебсервер локально. 

Пользователь на своей стороне локально (в идеале) поднимает прокси, например [Tonutils Proxy](https://github.com/xssnick/TonUtils-Proxy), и пользуется сайтами `.ton`, весь трафик заворачивается в обратном порядке, запросы идут в локальный прокси, а он отправляет их по RLDP на удаленный TON сайт.

HTTP внутри RLDP реализован с использованием TL структур:
```
http.header name:string value:string = http.Header;
http.payloadPart data:bytes trailer:(vector http.header) last:Bool = http.PayloadPart;
http.response http_version:string status_code:int reason:string headers:(vector http.header) no_payload:Bool = http.Response;

http.request id:int256 method:string url:string http_version:string headers:(vector http.header) = http.Response;
http.getNextPayloadPart id:int256 seqno:int max_chunk_size:int = http.PayloadPart;
```
Это не чистый HTTP в текстовом виде, все завернуто в бинарный TL и разворачивается обратно для отправки в вебсервер или браузер самим прокси.

Схема работы выглядит следующим образом:
* Клиент отправляет запрос `http.request`
* Сервер при получении запроса проверяет заголовок `Content-Length`
* * Если в нем не 0, отправляет клиенту запрос `http.getNextPayloadPart`
* * Клиент при получении запроса отправляет `http.payloadPart` - запрошенный кусок боди в зависимости от `seqno` и `max_chunk_size`.
* * Сервер повторяет запросы, увеличивая `seqno`, пока не получит все куски от клиента, т.е пока поле `last:Bool` последнего полученного кусочка не будет true.
* После обработки запроса, сервер отправляет `http.response`, клиент проверяет заголовок `Content-Length`
* * Если в нем не 0, то отправляет серверу запрос `http.getNextPayloadPart`, и повторяются операции, как в случае с клиентом.

#### Обращаемся к TON сайту

Чтобы понять как устроен RLDP, разберем пример с получением данных с TON сайта `foundation.ton`. Допустим, мы уже узнали его ADNL адрес, вызвав Get метод  NFT-DNS контракта, [определили адрес и порт RLDP сервиса используя DHT](https://github.com/xssnick/ton-deep-doc/blob/master/DHT.md), и [подключились к нему по ADNL UDP](https://github.com/xssnick/ton-deep-doc/blob/master/ADNL-UDP-Internal.md).

##### Отправим GET запрос к `foundation.ton`
Для этого заполним структуру:
```
http.request id:int256 method:string url:string http_version:string headers:(vector http.header) = http.Response;
```

Сериализуем `http.request`, заполнив поля:
```
e191b161                                                           -- TL ID http.request      
116505dac8a9a3cdb464f9b5dd9af78594f23f1c295099a9b50c8245de471194   -- id           = {random}
03 474554                                                          -- method       = GET
16 687474703a2f2f666f756e646174696f6e2e746f6e2f 00                 -- url          = http://foundation.ton/
08 485454502f312e31 000000                                         -- http_version = HTTP/1.1
01000000                                                           -- headers (1)
   04 486f7374 000000                                              -- name         = Host
   0e 666f756e646174696f6e2e746f6e 00                              -- value        = foundation.ton
```

Теперь упакуем наш сериализованный `http.request` в `rldp.query` и тоже сериализуем:
```
694d798a                                                              -- TL ID rldp.query
184c01cb1a1e4dc9322e5cabe8aa2d2a0a4dd82011edaf59eb66f3d4d15b1c5c      -- query_id        = {random}
0004040000000000                                                      -- max_answer_size = 257 KB, может быть любой достаточный
258f9063                                                              -- timeout (unix)  = 1670418213
34 e191b161116505dac8a9a3cdb464f9b5dd9af78594f23f1c295099a9b50c8245   -- data (http.request)
   de4711940347455416687474703a2f2f666f756e646174696f6e2e746f6e2f00
   08485454502f312e310000000100000004486f73740000000e666f756e646174
   696f6e2e746f6e00 000000
```

##### Кодируем и отправляем пакеты

Теперь наша задача применить к этим данным FEC алгоритм, а конкретно - RaptorQ.

Создадим энкодер, для этого нам нужно превратить полученный массив байт в символы фиксированного размера. В тоне размер символов равен 768 байт. 
Чтобы это сделать, разобьем массив на кусочки по 768 байт. В последнем кусочке, если он выйдет по размеру меньше 768, его нужно будет дополнить нулевыми байтами до необходимого размера. 

Наш массив размером 156 байт, значит будет всего 1 кусок, и нам нужно его дополнить 612ю нулевыми байтами до размера 768.

Так же для энкодера подбираются константы, зависящие от размера данных и символа, подробнее об этом можно узнать в документации самого RaptorQ, но чтобы не лезть в математические дебри, - я рекомендую использовать уже готовую библиотеку, реализующую такое кодирование. [[Пример создания энкодера]](https://github.com/xssnick/tonutils-go/blob/udp-rldp-2/adnl/rldp/raptorq/encoder.go#L15) и [[Пример кодирования символов]](https://github.com/xssnick/tonutils-go/blob/be3411cf412f23e6889bf0b648904306a15936e7/adnl/rldp/raptorq/solver.go#L26).

Кодирование и отправка символов осуществляются по кругу: изначально мы определяем `seqno`, который равен 0 и является параметром для кодирования, и увеличиваем его на 1 для каждого последующего кодированного пакета. Например, если у нас 2 символа, то мы кодируем и отправляем сначала первый, увеличиваем seqno на 1, потом второй и увеличиваем seqno на 1, потом опять первый и увеличиваем seqno, который на этот момент уже равен 2, ещё на 1. И так, пока мы не получим сообщение о том, что получатель принял данные.

Теперь, когда мы создали энкодер, мы готовы отправлять данные, для этого мы заполним TL схему:
```
fec.raptorQ data_size:int symbol_size:int symbols_count:int = fec.Type;

rldp.messagePart transfer_id:int256 fec_type:fec.Type part:int total_size:long seqno:int data:bytes = rldp.MessagePart;
```
* `transfer_id` - случайный int256, одинаковый для всех messagePart в рамках одной передачи данных.
*  `fec_type` будет `fec.raptorQ`.
*  * `data_size` = 156
*  * `symbol_size` = 768
*  * `symbols_count` = 1
*  `part` в нашем случае всегда 0, может использоваться для трансферов, которые упираются в лимит.
*  `total_size` = 156. Размер данных нашего трансфера. 
*  `seqno` - для первого пакета будет равен 0, и для каждого последующего будет увеличиваться на 1, с помощью него производится кодирование. 
*  `data` - наш закодированный символ, размером 768 байт.

После сериализации `rldp.messagePart`, заворачиваем его в `adnl.message.custom` и отправляем по ADNL UDP.

Мы отправляем пакеты по кругу, все время увеличивая seqno, пока не дождемся от получателя сообщения `rldp.complete`, либо останавливаемся по таймауту. После того, как мы отправили количество пакетов равное количеству наших символов, мы можем сбавить темп и слать по дополнительному пакету, например, раз в 10 миллисекунд или реже. Дополнительные пакеты служат для восстановления на случай потери данных, так как UDP быстрый, но ненадежный протокол.

[[Пример реализации]](https://github.com/xssnick/tonutils-go/blob/be3411cf412f23e6889bf0b648904306a15936e7/adnl/rldp/rldp.go#L249)

##### Обрабатываем ответ от `foundation.ton`

После отправки (или во время), мы уже можем ожидать ответ от сервера, в нашем случае мы ждем `rldp.answer` с `http.response` внутри. Он придет нам так же, в виде RLDP трансфера, как и отправлялся во время запроса. Мы получим `adnl.message.custom` сообщения, содержащие `rldp.messagePart`. 

Сначала нам нужно получить информацию о FEC из первого полученного сообщения транфсера, а конкретно параметры `data_size`, `symbol_size` и `symbols_count` из `fec.raptorQ` структуры messagePart. Они нам нужны для инициализации RaptorQ декодера. [[Пример]](https://github.com/xssnick/tonutils-go/blob/be3411cf412f23e6889bf0b648904306a15936e7/adnl/rldp/rldp.go#L137)

После инициализации, мы добавляем полученные символы с их `seqno` в наш декодер, и, как только накопили минимально необходимое количество равное `symbols_count`, можем пробовать декодировать полное сообщение. При успехе, мы отправим `rldp.complete`. [[Пример]](https://github.com/xssnick/tonutils-go/blob/be3411cf412f23e6889bf0b648904306a15936e7/adnl/rldp/rldp.go#L168)

В результате мы получим сообщение `rldp.answer`, с тем же query_id, что и в отправленном нами `rldp.query`. В данных должен быть `http.response`.
```
http.response http_version:string status_code:int reason:string headers:(vector http.header) no_payload:Bool = http.Response;
```
С основными полями, думаю, все понятно, суть, как и в HTTP. Интересный флаг тут `no_payload`, если там true - значит в ответе нет тела, (`Content-Length` = 0). Ответ от сервера можно считать полученным.

Если `no_payload` = false, значит в ответе есть контент, и нам нужно его получить. Для этого нам нужно отправить запрос c TL схемой `http.getNextPayloadPart`, обернутой в `rldp.query`.
```
http.getNextPayloadPart id:int256 seqno:int max_chunk_size:int = http.PayloadPart;
```
`id` - должен быть тот же, что мы отправляли в `http.request`, `seqno` - 0, и +1 для каждой следующей части. `max_chunk_size` - максимальный размер части, обычно используется 128 КБ (131072 байт).

В ответ мы получим:
```
http.payloadPart data:bytes trailer:(vector http.header) last:Bool = http.PayloadPart;
```
Если `last` = true, значит мы достигли конца, мы можем соединить все части воедино и получить полный боди ответа, например, html.


