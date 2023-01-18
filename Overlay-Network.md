## Overlay network

The architecture of TON is built in such a way that a lot of chains can exist simultaneously and independently in it - they can be both private and public.
Nodes have the ability to choose which shards and chains they store and process.
At the same time, the data exchange protocol remains unchanged due to its universality. Technologies such as DHT, RLDP and overlays allow this to be achieved.
We are already familiar with the first two, in this section we will get acquainted with overlays.

Overlays are responsible for dividing a single network into additional sub-networks. Overlays can be both public, to which anyone can connect, and private, where additional data is needed for entry, known only to a certain circle of people.

All chains in TON, including the masterchain, communicate using their overlay.
To join it, you need to find the nodes that are already in it, and start exchanging data with them.
You can find nodes using DHT.


### Interaction with other overlay nodes

We have already analyzed an example with finding overlay nodes in an article about DHT,
in the section [Search for nodes that store the state of the blockchain](https://github.com/xssnick/ton-deep-doc/blob/master/DHT.md#%D0%BF%D0%BE%D0%B8%D1%81%D0%BA-%D0%BD%D0%BE%D0%B4-%D1%85%D1%80%D0%B0%D0%BD%D1%8F%D1%89%D0%B8%D1%85-%D1%81%D0%BE%D1%81%D1%82%D0%BE%D1%8F%D0%BD%D0%B8%D0%B5-%D0%B1%D0%BB%D0%BE%D0%BA%D1%87%D0%B5%D0%B8%D0%BD%D0%B0). In this section, we will focus on interacting with them.

When querying the DHT, we will get the addresses of the overlay nodes, from which we can find out the addresses of other nodes of this overlay using [overlay.getRandomPeers](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/ton_api.tl#L237). 
nce we connect to a sufficient number of nodes, we can receive all block information and other chain events from them, as well as send our transactions to them for processing.

#### Let's find more neighbors

Let's look at an example of getting nodes in an overlay.

To do this, send a request `overlay.getRandomPeers`, serialize the TL schema:
```
overlay.node id:PublicKey overlay:int256 version:int signature:bytes = overlay.Node;
overlay.nodes nodes:(vector overlay.node) = overlay.Nodes;

overlay.getRandomPeers peers:overlay.nodes = overlay.Nodes;
```
`peers` - should contain the peers we know so we don't get them back, but since we don't know any yet, `peers.nodes` will be an empty array.

Each request inside the overlay must be prefixed with the TL schema:
```
overlay.query overlay:int256 = True;
```
The `overlay` should be the id of the overlay - the id of the `tonNode.ShardPublicOverlayId` schema key - the same one we used to search the DHT.

We need to unite 2 serialized schemas by simply concatenating 2 byte arrays, `overlay.query` will come first, `overlay.getRandomPeers` second.

We wrap the resulting array in the `adnl.message.query` scheme and send it via ADNL. In response, we are waiting for `overlay.nodes` - this will be a list of overlay nodes to which we can connect and, if necessary, repeat the same request to them until we get enough connections.

#### Functional requests

Once the connection is established, we can access the overlay nodes with [requests](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/ton_api.tl#L413) `tonNode.*`.

For requests of this kind, the RLDP protocol is used. And it's important not to forget the `overlay.query` prefix - it must be used for every query in the overlay.

There is nothing unusual about the requests themselves, they are very similar to what we [did in the article about ADNL TCP](https://github.com/xssnick/ton-deep-doc/blob/master/ADNL-TCP-Liteserver.md#getmasterchaininfo). 

For example, the `downloadBlockFull` request uses the already familiar block id:
```
tonNode.downloadBlockFull block:tonNode.blockIdExt = tonNode.DataFull;
```
By passing it, we will be able to download the full information about the block, in response we will receive:
```
tonNode.dataFull id:tonNode.blockIdExt proof:bytes block:bytes is_link:Bool = tonNode.DataFull;
  или
tonNode.dataFullEmpty = tonNode.DataFull;
```
If present, the `block` field will contain data in TL-B format.

Thus, we can receive information directly from the nodes.
