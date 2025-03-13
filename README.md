# Blockchain Development Notes

## Day 1: Introduction to Blockchain & Cryptography

### What is Blockchain?
- A **decentralized** and **immutable** ledger system.
- Transactions are recorded in blocks and linked together.
- Uses **cryptographic hashing** and **consensus mechanisms**.

### Cryptographic Hashing
- A **one-way function** that maps data to a fixed-size string.
- Example: SHA-256 Hash Function (Used in Bitcoin, Ethereum, etc.)

```python
import hashlib

data = "Hello Blockchain"
hash_result = hashlib.sha256(data.encode()).hexdigest()
print(hash_result)  # 256-bit hash output
```

### Public & Private Keys (Elliptic Curve Cryptography - ECC)
- **Public Key**: Acts as an **address** to receive funds.
- **Private Key**: Used to **sign transactions**.
- Ethereum uses **secp256k1** curve.

```python
from ecdsa import SigningKey, SECP256k1

sk = SigningKey.generate(curve=SECP256k1)  # Private Key
vk = sk.verifying_key  # Public Key

print("Private Key:", sk.to_string().hex())
print("Public Key:", vk.to_string().hex())
```

---

## Day 2: Proof of Work & Mining

### What is Proof of Work (PoW)?
- A mechanism where miners solve a computational problem to **add new blocks**.
- Requires **finding a valid nonce** so the hash has leading zeros.

### Mining Simulation in Python
```python
import hashlib

def mine_block(data, difficulty):
    nonce = 0
    while True:
        hash_result = hashlib.sha256(f"{data}{nonce}".encode()).hexdigest()
        if hash_result.startswith("0" * difficulty):
            return nonce, hash_result
        nonce += 1

nonce, final_hash = mine_block("Block Data", 4)
print("Nonce Found:", nonce)
print("Block Hash:", final_hash)
```

---

## Day 3: Smart Contracts & Solidity Basics

### Solidity Syntax Overview
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract HelloWorld {
    string public message = "Hello, Blockchain!";
}
```

### Important Solidity Concepts
- `pragma solidity ^0.8.20;` → Specifies the Solidity compiler version.
- `public` → Makes a variable accessible externally.
- `contract` → Defines a smart contract.

### Functions & State Variables
```solidity
contract Example {
    uint256 public number;
    
    function setNumber(uint256 _value) public {
        number = _value;
    }
}
```

---

## Day 4: Building a Voting Smart Contract

### Complete Voting Contract
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract Voting {
    address public owner;
    bool public votingOpen;

    struct Candidate {
        string name;
        uint256 voteCount;
    }

    mapping(uint256 => Candidate) public candidates;
    uint256 public candidateCount;

    mapping(address => bool) public hasVoted;
    event CandidateAdded(uint256 candidateId, string name);
    event Voted(address voter, uint256 candidateId);
    event VotingEnded(string winner);

    modifier onlyOwner() {
        require(msg.sender == owner, "Only owner can call this function");
        _;
    }

    constructor() {
        owner = msg.sender;
        votingOpen = true;
    }

    function addCandidate(string memory _name) public onlyOwner {
        require(votingOpen, "Voting has ended");
        candidates[candidateCount] = Candidate(_name, 0);
        emit CandidateAdded(candidateCount, _name);
        candidateCount++;
    }

    function vote(uint256 candidateId) public {
        require(votingOpen, "Voting has ended");
        require(!hasVoted[msg.sender], "You have already voted");
        require(candidateId < candidateCount, "Invalid candidate ID");

        candidates[candidateId].voteCount++;
        hasVoted[msg.sender] = true;
        emit Voted(msg.sender, candidateId);
    }

    function endVoting() public onlyOwner {
        require(votingOpen, "Voting already ended");
        votingOpen = false;
        uint256 winningVoteCount = 0;
        string memory winner;

        for (uint256 i = 0; i < candidateCount; i++) {
            if (candidates[i].voteCount > winningVoteCount) {
                winningVoteCount = candidates[i].voteCount;
                winner = candidates[i].name;
            }
        }
        emit VotingEnded(winner);
    }
}
```

---

## Next Steps
1. Learn Gas Optimization.
2. Implement Smart Contract Security.
3. Deploy on Ethereum Testnet.

---
This document will be **updated daily** with new Solidity lessons and projects! 🚀
