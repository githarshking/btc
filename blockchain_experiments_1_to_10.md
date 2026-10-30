# Blockchain Lab Experiments (1 to 10)

> **Reference:** Extracted and optimized from `blockchain 1-10.pdf`. All code has been minimized for faster copying, pasting, and execution.

## General Setup & Execution Steps

**For Python Programs (Exp 2, 5, 10):**
1. Install [Python](https://python.org).
2. Save the code as a `.py` file (e.g., `script.py`).
3. Run in your terminal: `python script.py`

**For Hardhat / Solidity Programs (Exp 3, 4, 7, 8):**
1. Install [Node.js](https://nodejs.org).
2. Open terminal in a new folder and run: 
   ```bash
   npm init -y
   npm install --save-dev hardhat @nomiclabs/hardhat-ethers ethers
   npx hardhat # Select "Create an empty hardhat.config.js"
   ```
3. Place `.sol` files in a `contracts/` folder and `.js` files in a `scripts/` folder.
4. Run locally: `npx hardhat run scripts/your_script.js`

---

## Experiment 1: Generate Public & Private Keys (OpenSSL)

**Aim:** Generate an RSA key pair using OpenSSL.
**Execution:** Install OpenSSL, open PowerShell/CMD, and run the following.

```bash
# Generate 2048-bit private key
openssl genpkey -algorithm RSA -out private.pem -pkeyopt rsa_keygen_bits:2048

# Extract public key
openssl rsa -in private.pem -pubout -out public.pem
```

---

## Experiment 2: Simple Blockchain in Python

**Aim:** Create a basic blockchain demonstrating blocks, hashing, and chaining.

```python
import hashlib, time

class Block:
    def __init__(self, data, prev="0"):
        self.data, self.prev, self.ts = data, prev, time.time()
        self.hash = hashlib.sha256(f"{self.ts}{data}{prev}".encode()).hexdigest()

# Initialize Chain
chain = [Block("Genesis")]
chain.append(Block("Second Block", chain[-1].hash))
chain.append(Block("Third Block", chain[-1].hash))

# Print Chain
for i, b in enumerate(chain): 
    print(f"Block {i}: {b.data} | Hash: {b.hash[:15]}... | Prev: {b.prev[:15]}...")
```

---

## Experiment 3: Smart Contract on Local Blockchain

**Aim:** Write and test a simple storage contract locally using Hardhat.

**`contracts/SimpleStorage.sol`**
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
contract SimpleStorage {
    uint public value;
    function set(uint _v) external { value = _v; }
}
```

**`scripts/test.js`** *(Run with: `npx hardhat run scripts/test.js`)*
```javascript
const hre = require("hardhat");
async function main() {
  const Factory = await hre.ethers.getContractFactory("SimpleStorage");
  const contract = await Factory.deploy();
  await contract.set(42);
  console.log("Stored Value:", (await contract.value()).toString()); // Expect 42
}
main();
```

---

## Experiment 4: Deploy to Ethereum Testnet (Sepolia)

**Aim:** Deploy a contract to a public testnet.
**Setup:** Install `dotenv` (`npm i dotenv`), create a `.env` file with `SEPOLIA_URL=...` and `PRIVATE_KEY=...`. 

**`hardhat.config.js`**
```javascript
require("@nomiclabs/hardhat-ethers");
require("dotenv").config();
module.exports = {
  solidity: "0.8.20",
  networks: { sepolia: { url: process.env.SEPOLIA_URL, accounts: [process.env.PRIVATE_KEY] } }
};
```

**`scripts/deploy.js`** *(Run: `npx hardhat run scripts/deploy.js --network sepolia`)*
```javascript
const hre = require("hardhat");
async function main() {
  const Factory = await hre.ethers.getContractFactory("SimpleStorage"); // From Exp 3
  const c = await Factory.deploy();
  console.log("Deployed to Sepolia at:", c.address);
}
main();
```

---

## Experiment 5: Multi-User Cryptocurrency (Python)

**Aim:** Simulate a basic cryptocurrency with balances and mining.

```python
import hashlib, time, json

class CryptoChain:
    def __init__(self):
        self.chain, self.txs = [{"hash": "0"}], []
        
    def add_tx(self, sender, to, amt):
        self.txs.append({"s": sender, "t": to, "a": amt})
        
    def mine(self, miner):
        self.txs.append({"s": "SYS", "t": miner, "a": 50}) # Mining reward
        blk = {"txs": self.txs, "prev": self.chain[-1]["hash"], "ts": time.time()}
        blk["hash"] = hashlib.sha256(json.dumps(blk, sort_keys=True).encode()).hexdigest()
        self.chain.append(blk)
        self.txs = []
        
    def balance(self, user):
        return sum(t["a"] if t["t"]==user else -t["a"] 
                   for b in self.chain for t in b.get("txs",[]) if user in (t["s"], t["t"]))

# Demo
c = CryptoChain()
c.add_tx("SYS", "Alice", 100) # Fund Alice
c.mine("Miner1")
c.add_tx("Alice", "Bob", 20)  # Alice -> Bob
c.mine("Miner1")

print(f"Alice: {c.balance('Alice')}, Bob: {c.balance('Bob')}, Miner: {c.balance('Miner1')}")
```

---

## Experiment 6: Hyperledger Fabric Chaincode (Go)

**Aim:** Write an ultra-compact Asset Management chaincode for Fabric.
**Setup:** Place in `fabric-samples/chaincode/simpleasset/go/simpleasset.go` and deploy via test-network scripts.

```go
package main
import (
    "encoding/json"; "fmt"; "github.com/hyperledger/fabric-contract-api-go/contractapi"
)

type SC struct { contractapi.Contract }
type Asset struct { ID, Owner string; Value int }

func (s *SC) Create(ctx contractapi.TransactionContextInterface, id, owner string, v int) error {
    b, _ := json.Marshal(Asset{id, owner, v})
    return ctx.GetStub().PutState(id, b)
}

func (s *SC) Read(ctx contractapi.TransactionContextInterface, id string) (*Asset, error) {
    b, _ := ctx.GetStub().GetState(id)
    var a Asset; json.Unmarshal(b, &a)
    return &a, nil
}

func main() { 
    c, _ := contractapi.NewChaincode(new(SC)); c.Start() 
}
```

---

## Experiment 7: Balance Transfer Smart Contract

**Aim:** Send ETH to a contract, and let the owner transfer it out.

**`contracts/Transfer.sol`**
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
contract Transfer {
    address owner = msg.sender;
    receive() external payable {} // Accept ETH
    
    function sendOut(address payable to, uint amt) external {
        require(msg.sender == owner, "Not owner");
        to.call{value: amt}("");
    }
}
```

**`scripts/transfer.js`**
```javascript
const hre = require("hardhat");
async function main() {
  const [owner, rec] = await hre.ethers.getSigners();
  const c = await (await hre.ethers.getContractFactory("Transfer")).deploy();
  
  // Fund contract
  await owner.sendTransaction({to: c.address, value: hre.ethers.utils.parseEther("1")});
  // Transfer out
  await c.sendOut(rec.address, hre.ethers.utils.parseEther("0.5"));
  
  console.log("Recipient Bal:", hre.ethers.utils.formatEther(await rec.getBalance()));
}
main();
```

---

## Experiment 8: ERC20 Token Creation

**Aim:** Create an ERC20 token using OpenZeppelin.
**Setup:** `npm install @openzeppelin/contracts`

**`contracts/MyToken.sol`**
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract MyToken is ERC20 {
    constructor() ERC20("DemoToken", "DMT") { 
        _mint(msg.sender, 1000 * 10**18); // Mint 1000 tokens to deployer
    }
}
```

**`scripts/token.js`**
```javascript
const hre = require("hardhat");
async function main() {
  const [owner, user] = await hre.ethers.getSigners();
  const t = await (await hre.ethers.getContractFactory("MyToken")).deploy();
  
  // Send 50 tokens
  await t.transfer(user.address, hre.ethers.utils.parseUnits("50", 18));
  console.log("User Bal:", hre.ethers.utils.formatUnits(await t.balanceOf(user.address), 18));
}
main();
```

---

## Experiment 9: Setup MetaMask & Connect Web3 Wallet

**Aim:** Connect a simple HTML frontend to MetaMask.
**Setup:** Install MetaMask extension in your browser, save code as `index.html`, and open it.

```html
<!DOCTYPE html>
<html>
<body>
  <h2>Connect MetaMask</h2>
  <button id="btn">Connect Wallet</button>
  <p id="res"></p>

  <script>
    document.getElementById('btn').onclick = async () => {
      if (window.ethereum) {
        const accounts = await window.ethereum.request({ method: 'eth_requestAccounts' });
        document.getElementById('res').innerText = "Connected: " + accounts[0];
      } else {
        alert("Please install MetaMask!");
      }
    }
  </script>
</body>
</html>
```

---

## Experiment 10: Merkle Trees in Blockchain

**Aim:** Generate a Merkle Root for a list of transactions in Python.

```python
import hashlib

def hash_pair(a, b): 
    return hashlib.sha256((a+b).encode()).hexdigest()

def get_merkle_root(txs):
    if not txs: return ""
    lvl = [hashlib.sha256(t.encode()).hexdigest() for t in txs]
    
    while len(lvl) > 1:
        # Pair up hashes; duplicate last item if odd number of nodes
        lvl = [hash_pair(lvl[i], lvl[i+1] if i+1 < len(lvl) else lvl[i]) 
               for i in range(0, len(lvl), 2)]
    return lvl[0]

# Demo
transactions = ["Tx1", "Tx2", "Tx3"]
root = get_merkle_root(transactions)
print(f"Transactions: {transactions}\nMerkle Root: {root}")
```