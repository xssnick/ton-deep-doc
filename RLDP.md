## RLDP

RLDP - Reliable Large Datagram Protocol, это протокол работающий поверх ADNL UDP, который служит для передачи больших данных и 
включает в себя Forward Error Correction (FEC) алгоритмы, для замещения подтверждений получения пакетов другой стороной, 
это позволяет более эффективно, хоть и с бОльшим потреблением трафика передавать данные между компонентами сети. 

RLDP используется практически повсеместно в инфраструктуре TON, например для скачивания блоков с других нод и передачи им данных,
для запросов к TON сайтам, и будет использоваться в TON Storage.

#### Протокол

Для обмена данными RLDP использует следующие TL структуры:
```
rldp.messagePart transfer_id:int256 fec_type:fec.Type part:int total_size:long seqno:int data:bytes = rldp.MessagePart;
rldp.confirm transfer_id:int256 part:int seqno:int = rldp.MessagePart;
rldp.complete transfer_id:int256 part:int = rldp.MessagePart;

rldp.message id:int256 data:bytes = rldp.Message;
rldp.query query_id:int256 max_answer_size:long timeout:int data:bytes = rldp.Message;
rldp.answer query_id:int256 data:bytes = rldp.Message;
```
Сериализованная структура оборачивается в TL схему `adnl.message.custom` и отправляется по ADNL UDP. Для передачи больших данных используются трансферы, генерируется случайный `transfer_id`, сами данные обрабатываются FEC алгоритмом, полученные кусочки заворачиваются в структуру `rldp.messagePart` и отправляются получателю до тех пор пока получатель не отправит нам `rldp.complete`. Когда получатель собрал кусочки `rldp.messagePart` необходимые для сборки полного сообщения, он соединяет их все вместе, декодирует с помощью FEC, и полученный массив байтов десериализует в одну из стркутур `rldp.query` или `rldp.answer`, в зависимости от типа (tl id префикса).

### FEC

Допустимые Forward Error Correction алгоритмы для использования с RLDP это RoundRobin, Online, и RaptorQ. 
Сейчас для обмена данными используется именно [RaptorQ](https://www.qualcomm.com/media/documents/files/raptorq-technical-overview.pdf).

##### RaptorQ
Суть RaptorQ в том что данные делятся на так называемые символы - блоки, одинакового, заранее определенного размера. 

Из блоков создаются матрицы и к ним применяются дискретные математические операции. Это позволяет создать практически бесконечное количество символов 
из одних и тех же данных, все символы смешиваются и это позволяет восстановить потеряные пакеты без запроса дополнительных данных от сервера, 
при этом используя меньшее количество пакетов, чем это было бы если мы отправляли бы одни и те же кусочки по кругу.

Сгенерированные, символы отправляются получателю, пока он не скажет что все данные полученны и восстановленны, путем применения тех же дискретных операций.

[[Моя реализация RaptorQ на Golang]](https://github.com/xssnick/tonutils-go/tree/udp-rldp-2/adnl/rldp/raptorq)

### RLDP-HTTP

Для взаимодействия с сайтами на TON используется обернутый HTTP в RLDP, хостер размещает свой сайт на любом HTTP вебсервере, и рядом поднимает rldp-http-proxy, все запросы из сети TON приходят по протоколу RLDP в прокси, а прокси уже пересобирает запрос в обычный HTTP и вызывает вебсервер локально. 

А пользователь на своей стороне - локально (в идеале) поднимает прокси, например [Tonutils Proxy](https://github.com/xssnick/TonUtils-Proxy), и пользуется сайтами `.ton`, весь трафик заворачивается в обратном порядке, запросы идут в локальный прокси, а он отправляет их по RLDP на удаленный TON сайт.

HTTP внутри RLDP реализован с использованием TL структур:
```
http.header name:string value:string = http.Header;
http.payloadPart data:bytes trailer:(vector http.header) last:Bool = http.PayloadPart;
http.response http_version:string status_code:int reason:string headers:(vector http.header) no_payload:Bool = http.Response;

http.request id:int256 method:string url:string http_version:string headers:(vector http.header) = http.Response;
http.getNextPayloadPart id:int256 seqno:int max_chunk_size:int = http.PayloadPart;
```
Это не чистый HTTP в текстовом виде, все завернуто в бинарный TL и развертывается обратно для отправки в вебсервер или браузер самим прокси.

Схема работы выглядит следующим образом:
* Клиент отправляет запрос `http.request`
* Сервер при получении запроса проверяет заголовок `Content-Length`
* * Если в нем не 0, отправляет клиенту запрос `http.getNextPayloadPart`
* * Клиент при получении запроса отправляет `http.payloadPart` - запрошеный кусок боди в зависимости от `seqno` и `max_chunk_size`.
* * Сервер повторяет запросы увеличивая `seqno` пока не получит все куски от клиента, т.е пока `last:Bool` кусочка не будет true.
* После обработки запроса сервер отправляет `http.response`, клиент проверяет заголовок `Content-Length`
* * Если в нем не 0, то отправлят серверу запрос `http.getNextPayloadPart` и повторяются операции как в случае с клиентом.

#### Обращаемся к TON сайту

Чтобы понять как устроен RLDP - разберем пример с получением данных с TON сайта `foundation.ton`. Допустим мы уже узнали его ADNL адрес вызвав Get метод  NFT-DNS контракта, [определили адрес и порт RLDP сервиса используя DHT](https://github.com/xssnick/ton-deep-doc/blob/master/DHT.md), и [подключились к нему по ADNL UDP](https://github.com/xssnick/ton-deep-doc/blob/master/ADNL-UDP-Internal.md).

###### Отправим GET запрос к `foundation.ton`.
Для этого заполним структуру:
```
http.request id:int256 method:string url:string http_version:string headers:(vector http.header) = http.Response;
```

```
e191b161                                                           -- TL ID http.request      
116505dac8a9a3cdb464f9b5dd9af78594f23f1c295099a9b50c8245de471194   -- id
03 474554                                                          -- method       = GET
16 687474703a2f2f666f756e646174696f6e2e746f6e2f 00                 -- url          = http://foundation.ton/
08 485454502f312e31 000000                                         -- http_version = HTTP/1.1
01000000                                                           -- headers (1)
   04 486f7374 000000                                              -- name         = Host
   0e 666f756e646174696f6e2e746f6e 00                              -- value        = foundation.ton
```

```
694d798a
184c01cb1a1e4dc9322e5cabe8aa2d2a0a4dd82011edaf59eb66f3d4d15b1c5c
0004040000000000 
258f9063 
c8 e191b161116505dac8a9a3cdb464f9b5dd9af78594f23f1c295099a9b50c8245
   de4711940347455416687474703a2f2f666f756e646174696f6e2e746f6e2f00
   08485454502f312e310000000100000004486f73740000000e666f756e646174
   696f6e2e746f6e00
```

TODO





