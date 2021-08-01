# Layer2_SPV

Simplified Payment Verification (“SPV”) proofs. SPV Proofs are compact proofs of payment. This tool can be used to prove to a client that isn’t running a full node that an on-chain payment actually happened.

This tool has to perform following two steps:

- Using the block’s transaction root, find the transaction that the user is claiming (aka transaction inclusion proof)
- Make sure that the block that contains the user transaction is a valid block in the target chain (aka block inclusion proof)


Target Layer2 and Eth_Mainnet

|  language\Layer2 | zksync | arbitrum | optimistm | Eth_Mainnet | polygon | 
| ----      | ---- | ---- | ---- | ---- | ---- |
|  API      | [ ] | [ ] | [ ] |  [ ] |  [ ] |
|  python   | [ ] | [ ] | [ ] |  [ ] |  [ ] |
|  solidity | [ ] | [ ] | [ ] |  [ ] |  [ ] |
|  js       | [ ] | [ ] | [ ] |  [ ] |  [ ] |



## Program

#### Security assumption
- Statehash is trusted.Different from the side chain, Layer2 runs on L1, which means that the statehash of layer2 has the security level of ETH, and we can get it in the smart contract.
- The block browser is guaranteed to provide services. For users, it is not realistic to run a low-level program, so just provide the necessary information through the browser. 
- The third-party API is guaranteed to provide services. Light nodes need full nodes to maintain the correct data, but now there are many programs like infurra that replace full nodes in providing services

#### Merkle-patricia-tree (trie) 

- Prove the existence or non-existence of the transaction




## Layer2
#### zksync

- Statehash on L1: https://etherscan.io/address/0xabea9132b05a70803a4e85094fd0e1800777fbef
- The block browser: https://zkscan.io/
- API: https://zksync.io/apiv02-docs/


The forward generation sequence of data ([data structure](https://github.com/matter-labs/zksync/blob/master/docs/protocol.md#example-1))









