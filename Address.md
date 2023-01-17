
### Addresses
The addresses in TON are formed from the sha256 hash of the TL-B structure of StateInit. The address can be calculated even before the contract is deployed to the network.

##### Serialization
The standard address like `EQBL2_3lMiyywU17g-or8N7v9hDmPCpttzBPE2isF2GTzpK4` is the base64 uri encoded bytes. 
Its length is 36 bytes, the last 2 of which are the crc16 checksum with the XMODEM table. The first byte is the flags, the second is the workchain.
32 bytes in the middle are the data of the address itself, often represented in diagrams as int256.

[Decoding example](https://github.com/xssnick/tonutils-go/blob/3d9ee052689376061bf7e4a22037ff131183afad/address/addr.go#L156)
