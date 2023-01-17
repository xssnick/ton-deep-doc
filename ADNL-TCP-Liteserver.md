
## ADNL TCP + Liteserver
This is the lower level protocol on which all interaction in the TON network is built, it can work on top of any protocol, but is most often used on top of TCP and UDP. UDP is used for communication between nodes, and TCP is used for communication with lite servers.

Now we will analyze ADNL running over TCP and learn how to interact with light servers directly.

In the TCP version of ADNL, network nodes use public keys ed25519 as addresses and establish a connection using a shared key obtained using the Elliptic Curve Diffie-Hellman procedure - ECDH.


### Package Structure
Each ADNL TCP packet, except for the handshake, has the following structure:
* 4 bytes of packet size in little endian (N)
* 32 bytes nonce [[?]](## "Random bytes to protect against checksum attacks")
* (N - 64) payload bytes
* 32 bytes SHA256 checksum from nonce and payload

The entire packet, including the size, is **AES-CTR** encrypted.
After decryption, it is necessary to check whether the checksum matches the data, to check, you just need to calculate the checksum yourself and compare the result with what we have in the package.

The handshake packet is an exception, it is transmitted in a partially clear form and is described in the next chapter.


### Establishing a connection
To establish a connection, we need to know the ip, port and public key of the server, and generate our own private and public key ed25519. 

Public server data such as ip, port and key can be obtained from the [config](https://ton-blockchain.github.io/global.config.json). IP in the config in numerical form, it can be brought to normal using, for example [this tool](https://www.browserling.com/tools/dec-to-ip). The public key in the config in base64 format.

The client generates 160 random bytes, some of which will be used by the parties as the basis for AES encryption.

Of these, 2 permanent AES-CTR ciphers are created, which will be used by the parties to encrypt / decrypt messages after the handshake.
* Cipher A - key 0 - 31 bytes, iv 64 - 79 bytes
* Cipher B - key 32 - 63 bytes, iv 80 - 95 bytes

The ciphers are applied in this order:
* Cipher A is used by the server to encrypt the messages it sends.
* Cipher A is used by the client to decrypt received messages.
* Cipher B is used by the client to encrypt the messages it sends.
* Cipher B is used by the server to decrypt received messages.

To establish a connection, the client must send a handshake packet containing:
* [32 bytes] **Server key ID** [[More]](#%D0%BF%D0%BE%D0%BB%D1%83%D1%87%D0%B5%D0%BD%D0%B8%D0%B5-%D0%B0%D0%B9%D0%B4%D0%B8-%D0%BA%D0%BB%D1%8E%D1%87%D0%B0)
* [32 bytes] **Our public key is ed25519**
* [32 bytes] **SHA256 hash from our 160 bytes**
* [160 bytes] **Our 160 bytes encrypted** [[More]](#%D1%88%D0%B8%D1%84%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5-%D0%B4%D0%B0%D0%BD%D0%BD%D1%8B%D1%85-handshake-%D0%BF%D0%B0%D0%BA%D0%B5%D1%82%D0%B0)


When receiving a handshake packet, the server will do the same actions on itself, receive an ECDH key, decrypt 160 bytes and create 2 permanent keys. If everything works out, the server will respond with an empty ADNL packet, without payload, to decrypt which (as well as subsequent ones) you need to use one of the permanent ciphers. 

From this point on, the connection can be considered established.

## Data exchange
After we have established a connection, we can start receiving information; the TL language is used to serialize data.

[More about TL](/TL.md)

### Ping&Pong
It is optimal to send a ping packet about once every 5 seconds. This is necessary to maintain the connection while no data is being exchanged, otherwise the server will terminate the connection.

The ping packet, like all the others, is built according to the standard scheme described [above](#package-structure), and carries the request ID and ping ID as payload data.

Let's find the desired scheme for the ping request [here](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/ton_api.tl#L35) and calculate the scheme id as 
`crc32_IEEEE("tcp.ping random_id:long = tcp.Pong")`. When converted to little endian bytes, we get **9a2b084d**.

Thus, our ADNL ping packet will look like this:
* 4 bytes of packet size in little endian -> 64 + (4+8) = **76**
* 32 bytes nonce -> random 32 bytes
* 4 bytes of ID TL schema -> **9a2b084d**
* 8 bytes of request id -> random number uint64 
* 32 bytes of SHA256 checksum from nonce and payload

We send our packet and wait for [tcp.pong](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/ton_api.tl#L23), `random_id` will be equal to the one we sent to ping.

### Receiving information from a light server
All requests that are aimed at obtaining information from the blockchain are wrapped in [LiteServer Query](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/lite_api.tl#L83) schema, which in turn is wrapped in [ADNL Query](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/lite_api.tl#L22) schema.

LiteQuery:
`liteServer.query data:bytes = Object`, id **df068c79**

ADNLQuery:
`adnl.message.query query_id:int256 query:bytes = adnl.Message`, id **7af98bb4**

LiteQuery is passed inside ADNLQuery, as `query:bytes`, and the final query is passed inside LiteQuery, as `data:bytes`.

[Parsing encoding bytes in TL](/TL.md)

#### Getting payloads
Now, since we already know how to generate TL packets for the Lite API, we can request information about the current TON masterchain block. The masterchain block is used in many further requests as an input parameter to indicate the state (moment) in which we need information.

##### getMasterchainInfo
We are looking for the [TL schema we need](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/lite_api.tl#L60), calculate its ID and build the package:

* 4 bytes of packet size in little endian -> 64 + (4+32+(1+4+(1+4+3)+3)) = **116**
* 32 bytes nonce -> random 32 bytes
* 4 bytes of ID ADNLQuery schema -> **7af98bb4**
* 32 bytes `query_id:int256` -> random 32 bytes
* * 1 byte array size -> **12**
* * 4 byte of ID LiteQuery schema -> **df068c79**
* * * 1 byte array size -> **4**
* * * 4 bytes of ID getMasterchainInfo schema -> **2ee6b589**
* * * 3 null bytes of padding (alignment to 8)
* * 3 null bytes of padding (alignment to 16)
* 32 bytes of checksum SHA256 from nonce and payload

Packet example in hex:
```
74000000                                                             -> packet size (116)
5fb13e11977cb5cff0fbf7f23f674d734cb7c4bf01322c5e6b928c5d8ea09cfd     -> nonce
  7af98bb4                                                           -> ADNLQuery
  77c1545b96fa136b8e01cc08338bec47e8a43215492dda6d4d7e286382bb00c4   -> query_id
    0c                                                               -> array size
    df068c79                                                         -> LiteQuery
      04                                                             -> array size
      2ee6b589                                                       -> getMasterchainInfo
      000000                                                         -> 3 bytes of padding
    000000                                                           -> 3 bytes of padding
ac2253594c86bd308ed631d57a63db4ab21279e9382e416128b58ee95897e164     -> sha256
```

In response, we expect to receive [liteServer.masterchainInfo](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/lite_api.tl#L30), consisting of  last:[ton.blockIdExt](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/tonlib_api.tl#L51) state_root_hash:int256 и init:[tonNode.zeroStateIdExt](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/ton_api.tl#L359).

The received packet is deserialized in the same way as the sent one - he same algorithm, but in the opposite direction, except that the response is wrapped only in [ADNLAnswer](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/lite_api.tl#L23).

After decoding the response, we get a packet of the form:
```
20010000                                                                  -> packet size (288)
5558b3227092e39782bd4ff9ef74bee875ab2b0661cf17efdfcd4da4e53e78e6          -> nonce
  1684ac0f                                                                -> ADNLAnswer
  77c1545b96fa136b8e01cc08338bec47e8a43215492dda6d4d7e286382bb00c4        -> query_id (identical to request)
    b8                                                                    -> array size
    81288385                                                              -> liteServer.masterchainInfo
                                                                          last:tonNode.blockIdExt
        ffffffff                                                          -> workchain:int
        0000000000000080                                                  -> shard:long
        27405801                                                          -> seqno:int   
        e585a47bd5978f6a4fb2b56aa2082ec9deac33aaae19e78241b97522e1fb43d4  -> root_hash:int256
        876851b60521311853f59c002d46b0bd80054af4bce340787a00bd04e0123517  -> file_hash:int256
      8b4d3b38b06bb484015faf9821c3ba1c609a25b74f30e1e585b8c8e820ef0976    -> state_root_hash:int256
                                                                          init:tonNode.zeroStateIdExt 
        ffffffff                                                          -> workchain:int
        17a3a92992aabea785a7a090985a265cd31f323d849da51239737e321fb05569  -> root_hash:int256      
        5e994fcf4d425c0a6ce6a792594b7173205f740a39cd56f537defd28b48a0f6e  -> file_hash:int256
    000000                                                                -> 3 bytes of padding
520c46d1ea4daccdf27ae21750ff4982d59a30672b3ce8674195e8a23e270d21          -> sha256
```

##### runSmcMethod
We already know how to get the masterchain block, so now we can call any light server methods.
Let's analyze **runSmcMethod** - this is a method that calls a function from a smart contract and returns a result.  Here we need to understand some new data types such as [TL-B](/TL-B.md), [Cell](https://ton.org/docs/learn/overviews/Cells) и [BoC](/Cells-BoC.md#bag-of-cells).

To execute the smart contract method, we need to send a request using the TL schema:
`liteServer.runSmcMethod mode:# id:tonNode.blockIdExt account:liteServer.accountId method_id:long params:bytes = liteServer.RunMethodResult`

And wait for a response like:
`liteServer.runMethodResult mode:# id:tonNode.blockIdExt shardblk:tonNode.blockIdExt shard_proof:mode.0?bytes proof:mode.0?bytes state_proof:mode.1?bytes init_c7:mode.3?bytes lib_extras:mode.4?bytes exit_code:int result:mode.2?bytes = liteServer.RunMethodResult;`

In the request, we see the following fields:
1. mode:# - uint32 bitmask of what we want to see in the response, for example, result:mode.2?bytes will only be present in the response if the bit with index 2 is one.
2. id:tonNode.blockIdExt - is our master block state that we got in the previous chapter.
3. account:[liteServer.accountId](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/lite_api.tl#L27) - workchain and smart contract address data.
4. method_id:long - 8 bytes, in which crc16 is written with the XMODEM table on behalf of the called method + bit 17 is set [[Calculation]](https://github.com/xssnick/tonutils-go/blob/88f83bc3554ca78453dd1a42e9e9ea82554e3dd2/ton/runmethod.go#L16)
5. params:bytes - [Stack](https://github.com/ton-blockchain/ton/blob/master/crypto/block/block.tlb#L783) serialized in [BoC](/Cells-BoC.md#bag-of-cells), containing arguments to call the method. [[Implementation example]](https://github.com/xssnick/tonutils-go/blob/88f83bc3554ca78453dd1a42e9e9ea82554e3dd2/tlb/stack.go)

For example, we only need `result:mode.2?bytes`, then our mode will be equal to 0b100, that is 4. In response, we will get:
1. mode:# -> what was sent - 4.
2. id:tonNode.blockIdExt -> our master block against which the method was executed
3. shardblk:tonNode.blockIdExt -> shard block where the contract account is located
4. exit_code:int -> 4 bytes which is the exit code when executing the method. If everything is successful, then = 0, if not, it is equal to the exception code.
5. result:mode.2?bytes -> [Stack](https://github.com/ton-blockchain/ton/blob/master/crypto/block/block.tlb#L783) serialized in [BoC](/Cells-BoC.md#bag-of-cells), containing the values returned by the method.

Let's analyze the call and getting the result from the `a2` method of the contract `EQBL2_3lMiyywU17g-or8N7v9hDmPCpttzBPE2isF2GTzpK4`:

Method code in FunC:
```
(cell, cell) a2() method_id {
  cell a = begin_cell().store_uint(0xAABBCC8, 32).end_cell();
  cell b = begin_cell().store_uint(0xCCFFCC1, 32).end_cell();
  return (a, b);
}
```

Fill out our request:
* `mode` = 4, we only need the result -> `04000000`
* `id` = result of execution getMasterchainInfo
* `account` = workchain 0 (4 bytes `00000000`), and int256 [obtained from our contract address](/Address.md#сериализация), i.e. 32 bytes `4bdbfde5322cb2c14d7b83ea2bf0deeff610e63c2a6db7304f1368ac176193ce`
* `method_id` = [computed](https://github.com/xssnick/tonutils-go/blob/88f83bc3554ca78453dd1a42e9e9ea82554e3dd2/ton/runmethod.go#L16) id from `a2` -> `0a2e010000000000`
* `params:bytes` = Our method does not accept input parameters, so we need to pass it an empty stack (`000000`, cell 3 bytes - stack depth 0) serialized in [BoC](/Cells-BoC.md#bag-of-cells) -> `b5ee9c72010101010005000006000000` -> serialize in bytes and get `10b5ee9c72410101010005000006000000000000` 0x10 - size, 3 bytes in the end - padding.

In response we get:
* `mode:#` -> не интересен
* `id:tonNode.blockIdExt` -> не интересен
* `shardblk:tonNode.blockIdExt` -> не интересен
* `exit_code:int` -> равен 0, если все успешно
* `result:mode.2?bytes` -> [Stack](https://github.com/ton-blockchain/ton/blob/master/crypto/block/block.tlb#L783) содержащий возвращенные методом данные в формате BoC, его мы распакуем.

Внутри `result` мы получили `b5ee9c7201010501001b000208000002030102020203030400080ccffcc1000000080aabbcc8`, это [BoC](/Cells-BoC.md#bag-of-cells) содержащий стек с данными. Когда мы десериализуем его, мы получим ячейку:
```
32[00000203] -> {
  8[03] -> {
    0[],
    32[0AABBCC8]
  },
  32[0CCFFCC1]
}
```
Если мы ее распарсим, то получим 2 значения типа cell, которые возвращает наш FunC метод. 
Первые 3 байта корневой ячейки `000002` - это глубина стека, то есть 2. Значит метод вернул нам 2 значения. 

Продолжаем парсинг, следующие 8 бит (1 байт) - это тип значения на текущем уровне стека. Для некоторых типов он может занимать 2 байта. Возможные варианты можно посмотреть в [схеме](https://github.com/ton-blockchain/ton/blob/master/crypto/block/block.tlb#L766). В нашем случае мы имеем `03`, что означает:
```
vm_stk_cell#03 cell:^Cell = VmStackValue;
```
Значит тип нашего значения - cell, и, судя по схеме, он хранит само значение, как ссылку. Но, если мы посмотрим на схему хранения элементов стека, -
```
vm_stk_cons#_ {n:#} rest:^(VmStackList n) tos:VmStackValue = VmStackList (n + 1);
```
То увидим, что первая ссылка `rest:^(VmStackList n)` - это ячейка следующего значения на стеке, а наше значение `tos:VmStackValue` идет вторым, значит для получения значения нам нужно читать вторую ссылку, то есть `32[0CCFFCC1]` - это наш первый cell, который вернул контракт.

Теперь мы можем залезть глубже и достать второй элемент стека, проходим по первой ссылке, теперь мы имеем:
```
8[03] -> {
    0[],
    32[0AABBCC8]
  }
```
Повторяем тот же процесс. Первые 8 бит = `03` - то есть опять cell. Вторая ссылка - это значение `32[0AABBCC8]` и, так как глубина нашего стека равна 2, мы завершаем проход. Итого, мы имеем 2 значения, возвращенные контрактом, - `32[0CCFFCC1]` и `32[0AABBCC8]`. 

Обратите внимание, что они идут в обратном порядке. Точно так же нужно передавать и аргументы при вызове функции - в обратном порядке от того, что мы видим в коде FunC. 

[Пример реализации](https://github.com/xssnick/tonutils-go/blob/master/ton/runmethod.go#L24)

##### getAccountState
Для получения данных о состоянии аккаунта, таких как баланс, код и хранимые данные мы, можем использовать [getAccountState](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/lite_api.tl#L68). Для запроса нам понадобится [свежий мастер блок](#) и адрес аккаунта. В ответ мы получим TL структуру [AccountState](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/lite_api.tl#L38).

Разберем AccountState TL схему:
```
liteServer.accountState id:tonNode.blockIdExt shardblk:tonNode.blockIdExt shard_proof:bytes proof:bytes state:bytes = liteServer.AccountState;
```
1. `id` - это наш мастер блок, относительно которого мы получили данные.
2. `shardblk` - блок шарды воркчеина, на котором находится наш аккаунт, относительно которого мы получили данные.
3. `shard_proof` - merkle пруф блока шарды.
4. `proof` - merkle пруф состояния аккаунта.
5. `state` - [BoC](/Cells-BoC.md#bag-of-cells) TLB [схемы состояния аккаунта](https://github.com/ton-blockchain/ton/blob/master/crypto/block/block.tlb#L232).

Из всех этих данных то, что нам нужно, находится в `state`, разберем его. 

Например, получим состояние аккаунта TF `EQAhE3sLxHZpsyZ_HecMuwzvXHKLjYx4kEUehhOy2JmCcHCT`, `state` в ответе будет:
```
b5ee9c720102350100051e000277c0021137b0bc47669b3267f1de70cbb0cef5c728b8d8c7890451e8613b2d899827026a886043179d3f6000006e233be8722201d7d239dba7d818134001020114ff00f4a413f4bcf2c80b0d021d0000000105036248628d00000000e003040201cb05060013a03128bb16000000002002012007080043d218d748bc4d4f4ff93481fd41c39945d5587b8e2aa2d8a35eaf99eee92d9ba96004020120090a0201200b0c00432c915453c736b7692b5b4c76f3a90e6aeec7a02de9876c8a5eee589c104723a18020004307776cd691fbe13e891ed6dbd15461c098b1b95c822af605be8dc331e7d45571002000433817dc8de305734b0c8a3ad05264e9765a04a39dbe03dd9973aa612a61f766d7c02000431f8c67147ceba1700d3503e54c0820f965f4f82e5210e9a3224a776c8f3fad1840200201200e0f020148101104daf220c7008e8330db3ce08308d71820f90101d307db3c22c00013a1537178f40e6fa1f29fdb3c541abaf910f2a006f40420f90101d31f5118baf2aad33f705301f00a01c20801830abcb1f26853158040f40e6fa120980ea420c20af2670edff823aa1f5340b9f2615423a3534e2a2d2b2c0202cc12130201201819020120141502016616170003d1840223f2980bc7a0737d0986d9e52ed9e013c7a21c2b2f002d00a908b5d244a824c8b5d2a5c0b5007404fc02ba1b04a0004f085ba44c78081ba44c3800740835d2b0c026b500bc02f21633c5b332781c75c8f20073c5bd0032600201201a1b02012020210115bbed96d5034705520db3c8340201481c1d0201201e1f0173b11d7420c235c6083e404074c1e08075313b50f614c81e3d039be87ca7f5c2ffd78c7e443ca82b807d01085ba4d6dc4cb83e405636cf0069006031003daeda80e800e800fa02017a0211fc8080fc80dd794ff805e47a0000e78b64c00015ae19574100d56676a1ec40020120222302014824250151b7255b678626466a4610081e81cdf431c24d845a4000331a61e62e005ae0261c0b6fee1c0b77746e102d0185b5599b6786abe06fedb1c68a2270081e8f8df4a411c4605a400031c34410021ae424bae064f613990039e2ca840090081e886052261c52261c52265c4036625ccd88302d02012026270203993828290111ac1a6d9e2f81b609402d0015adf94100cc9576a1ec1840010da936cf0557c1602d0015addc2ce0806ab33b50f6200220db3c02f265f8005043714313db3ced542d34000ad3ffd3073004a0db3c2fae5320b0f26212b102a425b3531cb9b0258100e1aa23a028bcb0f269820186a0f8010597021110023e3e308e8d11101fdb3c40d778f44310bd05e254165b5473e7561053dcdb3c54710a547abc2e2f32300020ed44d0d31fd307d307d33ff404f404d10048018e1a30d20001f2a3d307d3075003d70120f90105f90115baf2a45003e06c2170542013000c01c8cbffcb0704d6db3ced54f80f70256e5389beb198106e102d50c75f078f1b30542403504ddb3c5055a046501049103a4b0953b9db3c5054167fe2f800078325a18e2c268040f4966fa52094305303b9de208e1638393908d2000197d3073016f007059130e27f080705926c31e2b3e63006343132330060708e2903d08308d718d307f40430531678f40e6fa1f2a5d70bff544544f910f2a6ae5220b15203bd14a1236ee66c2232007e5230be8e205f03f8009322d74a9802d307d402fb0002e83270c8ca0040148040f44302f0078e1771c8cb0014cb0712cb0758cf0158cf1640138040f44301e201208e8a104510344300db3ced54925f06e234001cc8cb1fcb07cb07cb3ff400f400c9
```

[Распарсим этот BoC](/Cells-BoC.md#bag-of-cells) и получим 

<details>
  <summary>большую ячейку</summary>
  
  ```
  473[C0021137B0BC47669B3267F1DE70CBB0CEF5C728B8D8C7890451E8613B2D899827026A886043179D3F6000006E233BE8722201D7D239DBA7D818130_] -> {
    80[FF00F4A413F4BCF2C80B] -> {
      2[0_] -> {
        4[4_] -> {
          8[CC] -> {
            2[0_] -> {
              13[D180],
              141[F2980BC7A0737D0986D9E52ED9E013C7A218] -> {
                40[D3FFD30730],
                48[01C8CBFFCB07]
              }
            },
            6[64] -> {
              178[00A908B5D244A824C8B5D2A5C0B5007404FC02BA1B048_],
              314[085BA44C78081BA44C3800740835D2B0C026B500BC02F21633C5B332781C75C8F20073C5BD00324_]
            }
          },
          2[0_] -> {
            2[0_] -> {
              84[BBED96D5034705520DB3C_] -> {
                112[C8CB1FCB07CB07CB3FF400F400C9]
              },
              4[4_] -> {
                2[0_] -> {
                  241[AEDA80E800E800FA02017A0211FC8080FC80DD794FF805E47A0000E78B648_],
                  81[AE19574100D56676A1EC0_]
                },
                458[B11D7420C235C6083E404074C1E08075313B50F614C81E3D039BE87CA7F5C2FFD78C7E443CA82B807D01085BA4D6DC4CB83E405636CF0069004_] -> {
                  384[708E2903D08308D718D307F40430531678F40E6FA1F2A5D70BFF544544F910F2A6AE5220B15203BD14A1236EE66C2232]
                }
              }
            },
            2[0_] -> {
              2[0_] -> {
                323[B7255B678626466A4610081E81CDF431C24D845A4000331A61E62E005AE0261C0B6FEE1C0B77746E0_] -> {
                  128[ED44D0D31FD307D307D33FF404F404D1]
                },
                531[B5599B6786ABE06FEDB1C68A2270081E8F8DF4A411C4605A400031C34410021AE424BAE064F613990039E2CA840090081E886052261C52261C52265C4036625CCD882_] -> {
                  128[ED44D0D31FD307D307D33FF404F404D1]
                }
              },
              4[4_] -> {
                2[0_] -> {
                  65[AC1A6D9E2F81B6090_] -> {
                    128[ED44D0D31FD307D307D33FF404F404D1]
                  },
                  81[ADF94100CC9576A1EC180_]
                },
                12[993_] -> {
                  50[A936CF0557C14_] -> {
                    128[ED44D0D31FD307D307D33FF404F404D1]
                  },
                  82[ADDC2CE0806AB33B50F60_]
                }
              }
            }
          }
        },
        872[F220C7008E8330DB3CE08308D71820F90101D307DB3C22C00013A1537178F40E6FA1F29FDB3C541ABAF910F2A006F40420F90101D31F5118BAF2AAD33F705301F00A01C20801830ABCB1F26853158040F40E6FA120980EA420C20AF2670EDFF823AA1F5340B9F2615423A3534E] -> {
          128[DB3C02F265F8005043714313DB3CED54] -> {
            128[ED44D0D31FD307D307D33FF404F404D1],
            112[C8CB1FCB07CB07CB3FF400F400C9]
          },
          128[ED44D0D31FD307D307D33FF404F404D1],
          40[D3FFD30730],
          640[DB3C2FAE5320B0F26212B102A425B3531CB9B0258100E1AA23A028BCB0F269820186A0F8010597021110023E3E308E8D11101FDB3C40D778F44310BD05E254165B5473E7561053DCDB3C54710A547ABC] -> {
            288[018E1A30D20001F2A3D307D3075003D70120F90105F90115BAF2A45003E06C2170542013],
            48[01C8CBFFCB07],
            504[5230BE8E205F03F8009322D74A9802D307D402FB0002E83270C8CA0040148040F44302F0078E1771C8CB0014CB0712CB0758CF0158CF1640138040F44301E2],
            856[DB3CED54F80F70256E5389BEB198106E102D50C75F078F1B30542403504DDB3C5055A046501049103A4B0953B9DB3C5054167FE2F800078325A18E2C268040F4966FA52094305303B9DE208E1638393908D2000197D3073016F007059130E27F080705926C31E2B3E63006] -> {
              112[C8CB1FCB07CB07CB3FF400F400C9],
              384[708E2903D08308D718D307F40430531678F40E6FA1F2A5D70BFF544544F910F2A6AE5220B15203BD14A1236EE66C2232],
              504[5230BE8E205F03F8009322D74A9802D307D402FB0002E83270C8CA0040148040F44302F0078E1771C8CB0014CB0712CB0758CF0158CF1640138040F44301E2],
              128[8E8A104510344300DB3CED54925F06E2] -> {
                112[C8CB1FCB07CB07CB3FF400F400C9]
              }
            }
          }
        }
      }
    },
    114[0000000105036248628D00000000C_] -> {
      7[CA] -> {
        2[0_] -> {
          2[0_] -> {
            266[2C915453C736B7692B5B4C76F3A90E6AEEC7A02DE9876C8A5EEE589C104723A1800_],
            266[07776CD691FBE13E891ED6DBD15461C098B1B95C822AF605BE8DC331E7D45571000_]
          },
          2[0_] -> {
            266[3817DC8DE305734B0C8A3AD05264E9765A04A39DBE03DD9973AA612A61F766D7C00_],
            266[1F8C67147CEBA1700D3503E54C0820F965F4F82E5210E9A3224A776C8F3FAD18400_]
          }
        },
        269[D218D748BC4D4F4FF93481FD41C39945D5587B8E2AA2D8A35EAF99EEE92D9BA96000]
      },
      74[A03128BB16000000000_]
    }
  }
  ```

</details>

Теперь нам нужно распарсить ячейку в соответствии с TL-B структурой:
```
account_none$0 = Account;

account$1 addr:MsgAddressInt storage_stat:StorageInfo
          storage:AccountStorage = Account;
```
Наша структура ссылается на другие, такие как:
```
anycast_info$_ depth:(#<= 30) { depth >= 1 } rewrite_pfx:(bits depth) = Anycast;
addr_std$10 anycast:(Maybe Anycast) workchain_id:int8 address:bits256  = MsgAddressInt;
addr_var$11 anycast:(Maybe Anycast) addr_len:(## 9) workchain_id:int32 address:(bits addr_len) = MsgAddressInt;
   
storage_info$_ used:StorageUsed last_paid:uint32 due_payment:(Maybe Grams) = StorageInfo;
storage_used$_ cells:(VarUInteger 7) bits:(VarUInteger 7) public_cells:(VarUInteger 7) = StorageUsed;
  
account_storage$_ last_trans_lt:uint64 balance:CurrencyCollection state:AccountState = AccountStorage;

currencies$_ grams:Grams other:ExtraCurrencyCollection = CurrencyCollection;
           
var_uint$_ {n:#} len:(#< n) value:(uint (len * 8)) = VarUInteger n;
var_int$_ {n:#} len:(#< n) value:(int (len * 8)) = VarInteger n;
nanograms$_ amount:(VarUInteger 16) = Grams;  
           
account_uninit$00 = AccountState;
account_active$1 _:StateInit = AccountState;
account_frozen$01 state_hash:bits256 = AccountState;
```

Как мы видим, ячейка содержит очень много данных, но мы разберем основные кейсы и получение баланса, остальное вы сможете разрбрать аналогичным способом.

Начнем парсинг. В данных корневой ячейки мы имеем:
```
C0021137B0BC47669B3267F1DE70CBB0CEF5C728B8D8C7890451E8613B2D899827026A886043179D3F6000006E233BE8722201D7D239DBA7D818130_
```
Переведем в бинарный вид и получим:
```
11000000000000100001000100110111101100001011110001000111011001101001101100110010011001111111000111011110011100001100101110110000110011101111010111000111001010001011100011011000110001111000100100000100010100011110100001100001001110110010110110001001100110000010011100000010011010101000100001100000010000110001011110011101001111110110000000000000000000000110111000100011001110111110100001110010001000100000000111010111110100100011100111011011101001111101100000011000000100110
```
Посмотрим на нашу основную TL-B структуру, мы видим, что у нас есть 2 варианта того, что там может быть - `account_none$0` или `account$1`. Понять, какой вариант у нас, мы можем прочитав префикс, заявленный после символа $, в нашем случае это 1 бит. Если там 0, то у нас `account_none`, если 1, то `account`. 

Наш первый бит из данных выше = 1, значит мы работаем с `account$1` и будем использовать схему:
```
account$1 addr:MsgAddressInt storage_stat:StorageInfo
          storage:AccountStorage = Account;
```
Далее у нас идет `addr:MsgAddressInt`, мы видим, что для MsgAddressInt у нас тоже есть несколько вариантов:
```
addr_std$10 anycast:(Maybe Anycast) workchain_id:int8 address:bits256  = MsgAddressInt;
addr_var$11 anycast:(Maybe Anycast) addr_len:(## 9) workchain_id:int32 address:(bits addr_len) = MsgAddressInt;
```
Чтобы понять с каким именно работать, мы, как и в прошлый раз, читаем биты префикса, в этот раз мы читаем 2 бита. Отрезаем уже прочитанный бит, остается `1000000...`, читаем первые 2 бита и получаем `10`, значит мы работаем с `addr_std$10`.

Следующим мы должны распарсить `anycast:(Maybe Anycast)`, Maybe значит, что мы должны прочитать 1 бит, и если там единица - читать Anycast, иначе пропустить. Наши оставшиеся биты - это `00000...`, читаем 1 бит, это 0, значит пропускаем Anycast.

Далее у нас идет `workchain_id:int8`, тут все просто, читаем 8 бит, это будет айди воркчеина. Читаем следующие 8 бит, все нули, значит воркчеин равен 0.

Далее читаем `address:bits256`, это 256 бит адреса, по тому же принципу, что и с `workchain_id`. Прочитав, мы получим `21137B0BC47669B3267F1DE70CBB0CEF5C728B8D8C7890451E8613B2D8998270` в hex представлении.



Мы прочитали адрес `addr:MsgAddressInt`, далее у нас идет `storage_stat:StorageInfo` из основной структуры, его схема:
```
storage_info$_ used:StorageUsed last_paid:uint32 due_payment:(Maybe Grams) = StorageInfo;
```
Первым идет `used:StorageUsed`, со схемой:
```
storage_used$_ cells:(VarUInteger 7) bits:(VarUInteger 7) public_cells:(VarUInteger 7) = StorageUsed;
```
Это количество используемых ячеек и битов для хранения данных аккаунта. Каждое поле определено, как `VarUInteger 7`, что значит uint динамического размера, но максимум 7 бит. Понять, как он устроен, можно по схеме:
```
var_uint$_ {n:#} len:(#< n) value:(uint (len * 8)) = VarUInteger n;
```
В нашем случае n будет равен 7. В len у нас будет `(#< 7)` что значит количество бит, которое вмещает число до 7. Определить его можно, переведя 7-1=6 в бинарный вид - `110`, получаем 3 бита, значит длина len = 3 бита. А value - это `(uint (len * 8))`. Чтобы его определить, нам нужно прочитать 3 бита длины, получить число и умножить на 8, это будет размер `value`, то есть количество битов, которое нужно прочитать для получения значения VarUInteger.

Прочитаем `cells:(VarUInteger 7)`, возьмем наши следующие биты из корневой ячейки, посмотрим на следующие 16 бит для понимания, это `0010011010101000`. Читаем первые 3 бита len, это `001`, то есть 1, получим размер (uint (1 * 8)), получим uint 8, читаем 8 бит, это будет `cells`, `00110101`, то есть 53 в десятеричном виде. Делаем то же самое для `bits` и `public_cells`.

Мы успешно прочитали `used:StorageUsed`, следующим у нас идет `last_paid:uint32`, тут все просто, читаем 32 бита. Так же все просто с `due_payment:(Maybe Grams)` тут мейби, который будет 0, соответственно Grams мы пропускаем. Но, если мейби равен 1, мы можем взглянуть на схему Grams `amount:(VarUInteger 16) = Grams` и сразу понять, что мы уже умеем с таким работать. Как в прошлый раз, только вместо 7 - у нас 16.

Далее у нас `storage:AccountStorage` со схемой:
```
account_storage$_ last_trans_lt:uint64 balance:CurrencyCollection state:AccountState = AccountStorage;
```
Читаем `last_trans_lt:uint64`, это 64 бита, хранящие lt последней транзакции аккаунта. И, наконец, баланс, представленный схемой:
```
currencies$_ grams:Grams other:ExtraCurrencyCollection = CurrencyCollection;
```
Отсюда мы прочитаем `grams:Grams`, который будет являться балансом аккаунта в нано-тонах.
`grams:Grams` это `VarUInteger 16`, для хранения 16 (в бинарном виде `10000`, отняв 1 получим `1111`) значит читаем первые 4 бита, и полученное значение умножаем на 8, далее читаем полученное количество бит, это и будет нашим балансом. 

Разберем на наших данных наши оставшиеся биты:
```
100000000111010111110100100011100111011011101001111101100000011000000100110
```
Читаем первые 4 бита - `1000`, это 8. 8*8=64, читаем следующие 64 бита = `0000011101011111010010001110011101101110100111110110000001100000`, убрав лишние нулевые биты, получим `11101011111010010001110011101101110100111110110000001100000`, что равно `531223439883591776`, и, переведя из нано в тон, получаем `531223439.883591776`.

На этом мы остановимся, так как уже разобрали все основные кейсы, остальное можно получить аналогичным образом с тем, что мы разбрали. Так же, дополнительную информацию по разобру TL-B можно найти в [оффициальной документации](/TL-B.md)

##### Другие методы
Теперь, изучив всю информацию, вы можете вызывать и обрабатывать ответы и от других методов lite-server'а. Принцип тот же :)

## Дополнительные технические детали хендшейка

#### Получение айди ключа
Айди ключа - это SHA256 хэш сериализованой TL схемы.

Чаще всего применяются следующие TL схемы:
```
pub.ed25519 key:int256 = PublicKey -- ID c6b41348
pub.aes key:int256 = PublicKey     -- ID d4adbc2d
pub.overlay name:bytes = PublicKey -- ID cb45ba34
pub.unenc data:bytes = PublicKey   -- ID 0a451fb6
pk.aes key:int256 = PrivateKey     -- ID 3751e8a5
```

Как пример, для ключей типа ED25519, которые используются для хендшейка, ID ключа будет являться sha256 хешом от 
**[0xC6, 0xB4, 0x13, 0x48]** и **публичного ключа**, (массива байт размером 36, префикс + ключ)

[Пример кода](https://github.com/xssnick/tonutils-go/blob/2b5e5a0e6ceaf3f28309b0833cb45de81c580acc/liteclient/crypto.go#L16)

#### Шифрование данных Handshake пакета
Хэндшейк пакет отправляется в полуоткрытом виде, зашифрованы только 160 байт, содержащие информацию о постоянных шифрах.

Чтобы их зашифровать, нам нужен AES-CTR шифр, для его получения нам нужен SHA256 хэш от 160 байт и [общий ключ ECDH](#%D0%BF%D0%BE%D0%BB%D1%83%D1%87%D0%B5%D0%BD%D0%B8%D0%B5-%D0%BE%D0%B1%D1%89%D0%B5%D0%B3%D0%BE-%D0%BA%D0%BB%D1%8E%D1%87%D0%B0-%D0%BF%D0%BE-ecdh)

Шифр собирается следующим образом:
* ключ = (0 - 15 байты общего ключа) + (16 - 31 байты хэша)
* iv   = (0 - 3 байты хэша) + (20 - 31 байты общего ключа)

После того, как шифр собран, мы шифруем им наши 160 байт.

[Пример кода](https://github.com/xssnick/tonutils-go/blob/2b5e5a0e6ceaf3f28309b0833cb45de81c580acc/liteclient/connection.go#L361)

#### Получение общего ключа по ECDH
Для расчета общего ключа нам понадобится наш приватный ключ и публичный ключ сервера. 

Суть DH в получении общего секретного ключа, без разглашения приватной информации. Приведу пример, как это происходит, в максимально упрощенном виде.
Предположим, что нужно сгенерировать общий ключ между нами и сервером, процесс будет выглядеть так:
1. Мы генерируем секретное и публичное числа, например **6** и **7**
2. Сервер генерирует секретное и публичное числа, например **5** и **15**
3. Мы с сервером обмениваемся публичными числами, отправляем серверу **7**, он нам отправляет **15**.
4. Мы высчитываем: **7^6 mod 15 = 4**
5. Сервер высчитывает: **7^5 mod 15 = 7**
6. Обмениваемся полученными числами, мы серверу **4**, он нам **7**
7. Мы высчитываем **7^6 mod 15 = 4**
8. Сервер высчитывает: **4^5 mod 15 = 4**
9. Общий ключ = **4**

Детали самого ECDH будут опущены, чтобы не усложнять прочтение. Он вычисляется с помощью 2 ключей, приватного и публичного, путем нахождения общей точки на кривой. Если интересно, то лучше почитать про это отдельно.

[Пример кода](https://github.com/xssnick/tonutils-go/blob/2b5e5a0e6ceaf3f28309b0833cb45de81c580acc/liteclient/crypto.go#L32)
