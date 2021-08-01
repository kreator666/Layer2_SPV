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
- data structure: https://github.com/matter-labs/zksync/blob/master/docs/protocol.md#example-1



> block inclusion proof 

Storage.sol
```solidity
/// @dev Total blocks proven.
uint32 public totalBlocksProven;

/// @dev Stored hashed StoredBlockInfo for some block number
mapping(uint32 => bytes32) public storedBlockHashes;

/// @Rollup block stored data
/// @member blockNumber Rollup block number
/// @member priorityOperations Number of priority operations processed
/// @member pendingOnchainOperationsHash Hash of all operations that must be processed after verify
/// @member timestamp Rollup block timestamp, have the same format as Ethereum block constant
/// @member stateHash Root hash of the rollup state
/// @member commitment Verified input for the zkSync circuit
struct StoredBlockInfo {
    uint32 blockNumber;
    uint64 priorityOperations;
    bytes32 pendingOnchainOperationsHash;
    uint256 timestamp;
    bytes32 stateHash;
    bytes32 commitment;
}
/// @notice Returns the keccak hash of the ABI-encoded StoredBlockInfo
function hashStoredBlockInfo(StoredBlockInfo memory _storedBlockInfo) internal pure returns (bytes32) {
    return keccak256(abi.encode(_storedBlockInfo));
}
```

zksync.sol
```solidity
contract ZkSync is UpgradeableMaster, Storage, Config, Events, ReentrancyGuard  {...
    
    
    /// @notice Commit block
    /// @notice 1. Checks onchain operations, timestamp.
    /// @notice 2. Store block commitments
    function commitBlocks(StoredBlockInfo memory _lastCommittedBlockData, CommitBlockInfo[] memory _newBlocksData)
        external
        nonReentrant
    {
        ...
        storedBlockHashes[_lastCommittedBlockData.blockNumber] = hashStoredBlockInfo(_lastCommittedBlockData);
        ...
    }
}
```

block inclusion proof used in this pseudocode language:
``` 
 block_inclusion_proof = zksync(contract_Address_On_L1).storedBlockHashes[blockNum_Have_TargetTX]
 if (blockNum_Have_TargetTX <= totalBlocksProven):
    return True , "block_inclusion_proof is effective"
 else:
    return False, "block_inclusion_proof is not effective"
```


> transaction inclusion proof











