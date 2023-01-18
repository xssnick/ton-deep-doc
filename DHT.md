# DHT

DHT stands for Distributed Hash Table and is essentially a distributed key-value database,
where each member of the network can store something, for example, information about themselves.

The implementation of DHT in TON is inherently similar to the implementation of [Kademlia](https://codethechange.stanford.edu/guides/guide_kademlia.html), which is used in IPFS.
Any network member can run a DHT node, generate keys and store data.
To do this, he needs to generate a random ID and inform other nodes about himself.

To determine which node to store the data on, an algorithm is used to determine the "distance" between the node and the key.
The algorithm is simple: we take the ID of the node and the ID of the key, we perform the XOR operation. The smaller the value, the closer the node.
The task is to store the key on the nodes as close as possible to the key, so that other network participants can, using
the same algorithm, find a node that can give data on this key.

### Finding a value by key
Let's look at an example with a search for a key, [connect to any DHT node and establish a connection via ADNL UDP](/ADNL-UDP-Internal.md#устройство-пакетов-и-обмен-информацией).

For example, we want to find the address and data for connecting to the RLDP node TON of the foundation.ton site. Let's say we have already obtained the ADNL address of this site by executing the Get method of the DNS contract. The ADNL address in hex representation is `516618cf6cbe9004f6883e742c9a2e3ca53ed02e3e36f4cef62a98ee1e449174`. Now our task is to find the ip, port and public key of the node that has this address.

To do this, we need to get the ID of the DHT key, first we will collect the DHT key itself:
```
dht.key id:int256 name:bytes idx:int = dht.Key
```
`name` is the type of key, for ADNL addresses the word `address` is used, and, for example, to search for shardchain nodes - `nodes`. But the key type can be any array of bytes, depending on the value you are looking for.

Filling in this diagram, we get:
```
8fde67f6                                                           -- TL ID dht.key
516618cf6cbe9004f6883e742c9a2e3ca53ed02e3e36f4cef62a98ee1e449174   -- our searched ADNL address
07 61646472657373                                                  -- key type, the word "address" as an array of bytes
00000000                                                           -- index 0 because there is only 1 key
```
Next - get the key ID, sha256 hash from the bytes serialized above. It will be `b30af0538916421b46df4ce580bf3a29316831e0c3323a7f156df0236c5b2f75`

Now we can start searching. To do this, we need to execute a query that has [schema](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/ton_api.tl#L197):
```
dht.findValue key:int256 k:int = dht.ValueResult
```
`key` is the id of our DHT key, and `k` is the "width" of the search, the smaller it is, the more accurate, but fewer potential nodes for polling. The maximum k for nodes in a TON is 10, we can use 6.

Let's populate this structure, serialize and send the request using the `adnl.message.query` scheme. [You can read more about this in another article.](/ADNL-UDP-Internal.md#устройство-пакетов-и-обмен-информацией).

In response, we can get:
* `dht.valueNotFound` - if the value is not found.
* `dht.valueFound` - if the value is found on this node.

##### dht.valueNotFound
If we get `dht.valueNotFound`, the response will contain a list of nodes that are known to the node we requested and are as close as possible to the key we requested from the list of nodes known to it. In this case, we need to connect and add the received nodes to the list known to us. After that, from the list of all nodes known to us, select the closest, accessible and not yet polled, and make the same request to it. And so on until we try all the nodes in the range we have chosen or until we stop receiving new nodes.

Let's analyze the response fields in more detail, the schemes used:
```
adnl.address.udp ip:int port:int = adnl.Address;
adnl.addressList addrs:(vector adnl.Address) version:int reinit_date:int priority:int expire_at:int = adnl.AddressList;

dht.node id:PublicKey addr_list:adnl.addressList version:int signature:bytes = dht.Node;
dht.nodes nodes:(vector dht.node) = dht.Nodes;

dht.valueNotFound nodes:dht.nodes = dht.ValueResult;
```
`dht.nodes -> nodes` -  list of DHT nodes (array).

Each node has an `id` which is its public key, usually [pub.ed25519](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/ton_api.tl#L47), used as a server key to connect to the node via ADNL. Also, each node has a list of addresses `addr_list:adnl.addressList`, version and signature.

We need to check the signature of each node, for this we read the value of `signature` and set the field to zero (we make it an empty bytes array). After - we serialize the TL structure `dht.node` with the nulled signature and check the `signature` that was before nulling. We check on the received serialized bytes, using the key from `id`. [[Implementation example]](https://github.com/xssnick/tonutils-go/blob/udp-rldp-2/adnl/dht/client.go#L91)

From the list `addrs:(vector adnl.Address)` we take the address and try to establish an ADNL UDP connection, as the server key we use `id`, which is the public key.

To find out the "distance" to this node - we need to take [key id](/ADNL-TCP-Liteserver.md#получение-айди-ключа) from the key from the `id` field and check the distance by the XOR operation from the node's key id and the desired key. If the distance is small enough, we can make the same request to this node. And so on, until we find a value or there are no more new nodes.

##### dht.valueFound
The response will contain the value itself, the full key information, and optionally a signature.

Let's analyze the response fields in more detail, the schemes used:
```
adnl.address.udp ip:int port:int = adnl.Address;
adnl.addressList addrs:(vector adnl.Address) version:int reinit_date:int priority:int expire_at:int = adnl.AddressList;

dht.key id:int256 name:bytes idx:int = dht.Key;

dht.updateRule.signature = dht.UpdateRule;
dht.updateRule.anybody = dht.UpdateRule;
dht.updateRule.overlayNodes = dht.UpdateRule;

dht.keyDescription key:dht.key id:PublicKey update_rule:dht.UpdateRule signature:bytes = dht.KeyDescription;

dht.value key:dht.keyDescription value:bytes ttl:int signature:bytes = dht.Value; 

dht.valueFound value:dht.Value = dht.ValueResult;
```
First, let's analyze `key:dht.keyDescription`, it is a complete description of the key, the key itself, information about who and how can update the value.

* `key:dht.key` - ключ, должен полностью совпадать с тем, от чего мы брали айди ключа для поиска. 
* `id:PublicKey` - публичный ключ владельца записи. 
* `update_rule:dht.UpdateRule` - правило обновления записи
* * `dht.updateRule.signature` - обновлять запись может только владелец приватного ключа, `signature` как ключа, так и значения должна быть валидной
* * `dht.updateRule.anybody` - обновлять запись могут все, `signature` пуста и не проверяется
* * `dht.updateRule.overlayNodes` - обновлять ключ могут только ноды из одного оверлея (шарды воркчеина)

###### dht.updateRule.signature
После чтения описания ключа, мы действуем в зависимости от `updateRule`, для кейса с поиском ADNL адреса тип всегда - `dht.updateRule.signature`. Проверяем подпись ключа тем же способом, что и в прошлый раз, делаем подпись пустым массивом байтов, сериализуем и проверяем. После - повторяем то же самое для значения, т.е для всего объекта `dht.value` (подпись ключа при этом возвращаем на место).

[[Пример реализации]](https://github.com/xssnick/tonutils-go/blob/udp-rldp-2/adnl/dht/client.go#L331)

###### dht.updateRule.overlayNodes
Используется для ключей, содержащих информацию о других нодах-шардах воркчеина в сети, в значении всегда имеет TL структуру `overlay.nodes`. Подпись значения должна быть пустой.

```
overlay.node id:PublicKey overlay:int256 version:int signature:bytes = overlay.Node;
overlay.nodes nodes:(vector overlay.node) = overlay.Nodes;
```
Для проверки валидности мы должны проверить все `nodes` и для каждой проверить `signature` на соответствие ее `id`, сериализовав TL структуру:
```
overlay.node.toSign id:adnl.id.short overlay:int256 version:int = overlay.node.ToSign;
```
Как мы видим, id надо заменить на adnl.id.short, который является айди ключа `id` из оригинальной структуры. После сериализации - сверяем подпись с данными.

В результате мы получаем валидный список нод, которые способны отдать нам информацию о нужной нам шарде воркчеина.
###### dht.updateRule.anybody
Подписей нет, обновлять может любой, но реального использования я не видел.

##### Использование значения

Когда все верифицировано и время жизни значения `ttl:int` не просрочено, мы можем начинать работать с самим значением, т.е `value:bytes`. Для ADNL адреса там внутри должна быть структура `adnl.addressList`. В ней будут ip адреса и порты серверов, соответствующих запрошенному ADNL адресу. В нашем случае там скорее всего будет 1 адрес RLDP-HTTP сервиса `foundation.ton`. В качестве ключа сервера мы будем использовать публичный ключ `id:PublicKey` из информации о ключе DHT.

После установки соединения - мы можем запрашивать страницы сайта, используя протокол RLDP. Задача DHT на этом этапе выполнена.


### Поиск нод хранящих состояние блокчеина

DHT так же используется для поиска информации о нодах, хранящих данные воркчеинов и их шардов. Процесс такой же, как и при поиске любого ключа, разница лишь в самой сериализации ключа и валидации ответа, эти моменты мы разберем в этом разделе.

Для того, чтобы получить данные, например, мастерчеина и его шарды, нам нужно заполнить TL структуру:
```
tonNode.shardPublicOverlayId workchain:int shard:long zero_state_file_hash:int256 = tonNode.ShardPublicOverlayId;
```
Где `workchain` в случае мастерчеина будет равен -1, его shard будет равен -922337203685477580, и zero state - хэш нулевого стейта чеина (file_hash), его, как и другие данные, можно взять в глобальном конфиге сети, в поле `"validator"`
```json
"zero_state": {
  "workchain": -1,
  "shard": -9223372036854775808, 
  "seqno": 0,
  "root_hash": "F6OpKZKqvqeFp6CQmFomXNMfMj2EnaUSOXN+Mh+wVWk=",
  "file_hash": "XplPz01CXAps5qeSWUtxcyBfdAo5zVb1N979KLSKD24="
}
```
После того, как мы заполнили `tonNode.shardPublicOverlayId`, мы сериализуем его и получаем из него айди ключа, путем хеширования (как и всегда).

Полученный айди ключа нам нужно использовать, как `name` для заполнения структуры `pub.overlay name:bytes = PublicKey`, завернув его в `bytes`. Далее сериализуем его, и получаем айди ключа теперь уже из него. 

Полученный айди будет ключом для использования в `dht.findValue`, а в качестве `name` будет слово `nodes`. Повторяем основной процесс из предыдущего раздела, все как и в прошлый раз, но `updateRule` будет [dht.updateRule.overlayNodes](#dhtupdateruleoverlaynodes).

После валидации - мы получим публичные ключи (`id`) нод имеющих информацию о нашем воркчеине и шарде. Чтобы получить ADNL адреса нод - нам нужно сделать из ключей айди (методом хеширования) и для каждого из ADNL адресов повторить процедуру описанную выше, как и с ADNL адресом домена `foundation.ton`.

В результате мы получим адреса нод, у которых, если захотим, сможем узнать адреса других нод этого чеина c помощью [overlay.getRandomPeers](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/ton_api.tl#L237). Также мы сможем получать от этих нод всю информацию о блоках. 
