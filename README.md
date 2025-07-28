

# Advanced Foundry & zkSync Mastery for Protocol Architects

This document provides a professional-grade, in-depth guide for senior smart contract engineers to master advanced Foundry workflows, DevOps automation, and zkSync development. It assumes a working knowledge of Solidity and basic Foundry commands.

---

## 1. Foundry Advanced Mastery

### 1.1. Project Structure Best Practices

A production-grade Foundry project extends the basic structure for clarity, separation of concerns, and ease of maintenance.

```
.
├── broadcast/         # Deployment artifacts from `forge script`
├── cache/             # Foundry's cache files
├── lib/               # Git submodules for dependencies (e.g., forge-std, solmate)
├── script/            # Deployment and interaction scripts (.s.sol)
├── src/               # Core contract source code
│   ├── interfaces/
│   └── utils/
├── test/              # Test files (.t.sol)
│   ├── integration/
│   ├── invariants/
│   └── unit/
├── .env.example       # Example environment variables
├── .gitignore
├── foundry.toml       # Project configuration
└── remappings.txt     # Dependency path remappings
```

**`foundry.toml` Configuration:**

Key profiles for different environments are essential.

```toml
[profile.default]
src = "src"
out = "out"
libs = ["lib"]
solc_version = "0.8.24"
optimizer = true
optimizer_runs = 200_000

[profile.ci]
# Profile for continuous integration runs
fuzz = { runs = 1024 }
invariant = { runs = 256 }

[rpc_endpoints]
mainnet = "${MAINNET_RPC_URL}"
sepolia = "${SEPOLIA_RPC_URL}"
zksync_sepolia = "${ZKSYNC_SEPOLIA_RPC_URL}"

[etherscan]
mainnet = { key = "${ETHERSCAN_API_KEY}" }
sepolia = { key = "${ETHERSCAN_API_KEY}" }
```

### 1.2. Advanced Testing with `forge-std`

#### Unit & Integration Tests
- **Unit Tests**: Isolate and test a single contract's logic. Use mocks for external dependencies.
- **Integration Tests**: Test the interaction between multiple contracts, often on a forked mainnet environment to ensure real-world compatibility.

#### Fuzz Testing
Fuzzing automatically generates random inputs to test for edge cases that manual tests might miss.

```solidity
// test/unit/MyToken.t.sol
import {Test} from "forge-std/Test.sol";
import {MyToken} from "src/MyToken.sol";

contract MyTokenTest is Test {
    MyToken public myToken;

    function setUp() public {
        myToken = new MyToken("My Token", "MTK");
    }

    // Fuzz test the transfer function
    function testFuzz_transfer(uint256 amount) public {
        address from = address(0x1337);
        address to = address(0x42);
        
        // Ensure the test environment has enough tokens to avoid reverts on valid logic
        vm.deal(from, 1e18);
        myToken.mint(from, 1e18);

        // Constrain inputs to valid ranges
        amount = bound(amount, 0, myToken.balanceOf(from));

        uint256 fromBalanceBefore = myToken.balanceOf(from);
        uint256 toBalanceBefore = myToken.balanceOf(to);

        myToken.transfer(to, amount);

        assertEq(myToken.balanceOf(from), fromBalanceBefore - amount);
        assertEq(myToken.balanceOf(to), toBalanceBefore + amount);
    }
}
```

#### Invariant Testing
Invariant tests define properties of your system that should *always* remain true, no matter what sequence of valid user actions is performed. Foundry uses a "handler" contract to execute a series of random calls on your target contract.

**Example**: The total supply of a token should never change unless a mint or burn function is called.

```solidity
// test/invariants/MyTokenInvariants.t.sol
import {Test, stdError} from "forge-std/Test.sol";
import {MyToken} from "src/MyToken.sol";
import {Handler} from "./Handler.sol";

contract MyTokenInvariants is Test {
    MyToken internal myToken;
    Handler internal handler;

    function setUp() public {
        myToken = new MyToken("My Token", "MTK");
        handler = new Handler(myToken);
        // The handler contract is where you define the sequence of actions
        // that can be called on the target contract.
        targetContract(address(handler));
    }

    // Invariant: Total supply only changes upon minting or burning.
    function invariant_totalSupply() public {
        uint256 initialTotalSupply = myToken.totalSupply();
        
        // The handler will perform a sequence of random calls here.
        // We check the state *after* the sequence.
        
        // This is a simplified example. A real handler would track mint/burn calls.
        // For this example, we assume no mint/burn functions are exposed in the handler.
        assertEq(myToken.totalSupply(), initialTotalSupply, "Total supply changed unexpectedly");
    }
}
```
Run with: `forge test --match-path test/invariants/* -vv`

### 1.3. Cheatcodes Deep Dive

Cheatcodes are the core of Foundry's testing power.

- **State Manipulation**:
  - `vm.prank(address)`: Sets `msg.sender` for the *next* call only.
  - `vm.startPrank(address)`: Sets `msg.sender` for all subsequent calls until `stopPrank()`.
  - `vm.deal(address, amount)`: Sets an address's ETH balance.
  - `vm.roll(blockNumber)`: Sets `block.number`.
  - `vm.warp(timestamp)`: Sets `block.timestamp`.

- **Expecting Events & Reverts**:
  - `vm.expectEmit(bool, bool, bool, bool)`: Checks for an event with indexed/non-indexed parameter matching.
  - `vm.expectRevert(bytes)`: Expects a specific error message. Use `stdError.Panic(uint256)` for panics or `abi.encodeWithSignature("MyError()")` for custom errors.

- **Debugging & Tracing**:
  - `vm.recordLogs()`: Starts recording emitted events.
  - `Vm.Log[] memory entries = vm.getRecordedLogs()`: Retrieves recorded logs.
  - `console.log(...)`: The classic debugging tool, visible with `-vv` or higher verbosity.

### 1.4. Mocking and Mainnet Forking

Forking is essential for integration testing against deployed protocols without mocks.

**CLI Command for Forking**:
```bash
# Start a local Anvil node forking Ethereum mainnet
anvil --fork-url $MAINNET_RPC_URL --fork-block-number 18000000
```

**Example: Testing against Uniswap V3 on a Fork**

```solidity
// test/integration/UniswapInteraction.t.sol
import {Test} from "forge-std/Test.sol";
import {IUniswapV3Pool} from "src/interfaces/IUniswapV3Pool.sol";
import {IERC20} from "forge-std/interfaces/IERC20.sol";

contract UniswapInteractionTest is Test {
    // Mainnet address for WETH/USDC 0.05% pool
    IUniswapV3Pool constant POOL = IUniswapV3Pool(0x88e6A0c2dDD26FEEb64F039a2c41296FcB3f5640);
    IERC20 constant WETH = IERC20(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2);
    IERC20 constant USDC = IERC20(0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48);
    address user = makeAddr("user");

    function testSwapWethForUsdc() public {
        // Give the user some WETH to trade
        deal(address(WETH), user, 10 ether);

        vm.startPrank(user);
        // Approve the pool to spend our WETH
        WETH.approve(address(POOL), 10 ether);

        // Perform the swap (simplified for example)
        // In a real scenario, you'd call the Uniswap Router
        // This is a low-level example showing direct pool interaction
        // A real swap would require a callback implementation.
        
        // For a simpler test: check oracle price
        (int24 tick, , , , , , ) = POOL.slot0();
        assertTrue(tick != 0);
        vm.stopPrank();
    }
}
```
Run test on the fork: `forge test --fork-url $MAINNET_RPC_URL --match-path test/integration/*`

### 1.5. Deployment with `forge script`

`forge script` provides a robust way to deploy contracts using Solidity.

**Key Components**:
- **`.env` file**: Store `PRIVATE_KEY`, `RPC_URL`, `ETHERSCAN_API_KEY`.
- **Script Contract**: Inherits from `Script.sol`.
- **`run()` function**: The entry point for the script.
- **`vm.startBroadcast()` & `vm.stopBroadcast()`**: The core of deployment. Any state change between these calls is sent as a real transaction.

```solidity
// script/DeployMyContract.s.sol
import {Script} from "forge-std/Script.sol";
import {MyContract} from "../src/MyContract.sol";

contract DeployMyContract is Script {
    function run() external returns (MyContract) {
        uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");
        vm.startBroadcast(deployerPrivateKey);

        MyContract myContract = new MyContract("Initial Value");

        vm.stopBroadcast();
        return myContract;
    }
}
```

**Deployment Command**:
```bash
# Load .env variables and deploy to Sepolia
source .env && forge script script/DeployMyContract.s.sol:DeployMyContract --rpc-url $SEPOLIA_RPC_URL --broadcast --verify -vvvv
```

### 1.6. Gas Optimization & Reporting

- **Gas Reports**: `forge test --gas-report`
- **Gas Snapshots**: `forge snapshot` to compare gas costs between code changes.

**Key Patterns**:
- `immutable` vs. `constant`: `immutable` variables are set in the constructor and read from code, cheaper than `constant` for dynamic values. `constant` values are inlined at compile time.
- **Custom Errors**: `error Unauthorized();` is significantly cheaper than `require(condition, "Unauthorized")`.
- **Bitpacking / Struct Packing**: Order struct variables from smallest to largest (e.g., `uint128`, `uint128`, `uint256`) to allow the EVM to pack them into single 256-bit slots.
- `calldata` vs. `memory`: Use `calldata` for external function arguments that are read-only, avoiding unnecessary memory copies.

---

## 2. Foundry DevOps with `foundry-devops`

The Cyfrin `foundry-devops` template standardizes and automates deployment, verification, and CI/CD.

### 2.1. Setup and Usage

Clone the template to start a new project.
```bash
git clone https://github.com/Cyfrin/foundry-devops
cd foundry-devops
# Follow setup instructions in the repository
```

### 2.2. Broadcast Files & Deployment Management

After a deployment, `broadcast/` is populated. The `run-latest.json` file is a critical artifact containing:
- All transactions sent.
- Contract addresses created.
- Chain ID.

This file is used by subsequent scripts to locate already-deployed contracts, enabling modular deployment strategies.

### 2.3. Auto-Verification with Etherscan

The template's `Makefile` or helper scripts typically include a verification flag.

```bash
# Example command from a Makefile-based setup
make deploy chain=sepolia script=DeployMyContract
```
This command reads the `ETHERSCAN_API_KEY` from your `.env` file and passes the `--verify` flag to `forge script`, automatically verifying the contract source code on Etherscan.

### 2.4. CI/CD with GitHub Actions

A robust CI/CD pipeline ensures code quality and automates deployment.

**`.github/workflows/ci.yml`**:
```yaml
name: CI

on:
  push:
    branches: [ main ]
  pull_request:

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
      with:
        submodules: recursive

    - name: Install Foundry
      uses: foundry-rs/foundry-toolchain@v1

    - name: Run Linter (forge fmt)
      run: forge fmt --check

    - name: Run Tests
      run: forge test -vv

  deploy:
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    needs: check
    environment:
      name: production
      url: https://sepolia.etherscan.io/address/<YOUR_CONTRACT_ADDRESS>

    steps:
    - uses: actions/checkout@v3
      with:
        submodules: recursive

    - name: Install Foundry
      uses: foundry-rs/foundry-toolchain@v1

    - name: Deploy to Sepolia
      run: |
        source .env.ci # Load CI-specific env vars
        forge script script/DeployMyContract.s.sol:DeployMyContract --rpc-url ${{ secrets.SEPOLIA_RPC_URL }} --private-key ${{ secrets.PRIVATE_KEY }} --broadcast --verify
      env:
        ETHERSCAN_API_KEY: ${{ secrets.ETHERSCAN_API_KEY }}
```

### 2.5. Managing Upgradeable Deployments

For upgradeable contracts (e.g., UUPS proxies), scripts must handle multiple steps:
1. Deploy the implementation contract.
2. Deploy the `ERC1967Proxy` contract, linking it to the implementation address.
3. Call the `initialize` function on the proxy.

Your deployment script would look like this:
```solidity
// script/DeployUpgradeable.s.sol
// ... imports for Proxy, Implementation ...

function run() external {
    vm.startBroadcast(vm.envUint("PRIVATE_KEY"));

    Implementation impl = new Implementation();
    ERC1967Proxy proxy = new ERC1967Proxy(address(impl), "");

    // Initialize via the proxy
    Implementation(address(proxy)).initialize();

    vm.stopBroadcast();
}
```

---

## 3. zkSync Development with Foundry

### 3.1. Project Setup

While native zkSync support in Foundry is evolving, a hybrid approach is common. You write Solidity code and use `zksync-cli` or the Hardhat zkSync plugins for compilation and deployment.

1.  **Install `zksync-cli`**:
    ```bash
    npm i -g @matterlabs/zksync-cli
    ```
2.  **Configure `foundry.toml`**: Add the zkSync testnet RPC endpoint.
3.  **Write standard Solidity**: Most EVM patterns work, but be aware of key differences.

### 3.2. EVM vs. zkEVM Differences

- **Native Account Abstraction**: Every account can be a smart contract. EOA-initiated transactions are processed through a bootloader.
- **Gas Model**: Gas costs include L1 pubdata costs for state changes, which is a major factor in optimization.
- **Opcodes**: Some opcodes behave differently or are not supported (e.g., `PUSH0` was added later). `block.timestamp` and `block.number` have different meanings.
- **Pointers & Memory**: `zksolc` (the zkSync compiler) has stricter rules around memory and pointers than `solc`.

### 3.3. Deployment to zkSync Testnet

Use `zksync-cli` to deploy contracts compiled with `zksolc`.

```bash
# Create a new project or integrate into existing
zksync-cli create my-zksync-project

# To deploy a contract
zksync-cli contract deploy --rpc <zksync_rpc_url> \
  --private-key <your_private_key> \
  --contract-name <contract_to_deploy> \
  --args <constructor_args>
```

### 3.4. Hybrid Approach: Foundry + Hardhat/zksync-cli

- **Develop & Test in Foundry**: Use Foundry for its speed and powerful testing suite for all standard EVM logic. Mock out zkSync-specific features.
- **Compile & Deploy with zkSync Tools**: Use `zksync-cli` or a Hardhat project with the `@matterlabs/hardhat-zksync-solc` and `@matterlabs/hardhat-zksync-deploy` plugins for the final compilation and deployment steps.

This hybrid model leverages the best of both ecosystems.

### 3.5. Writing zkSync-Native Scripts

While `forge script` can be used to send transactions to a zkSync network (as it's EVM-compatible), deploying complex systems or interacting with zkSync-native features often requires `ethers.js` v6+ with the `zksync-ethers` plugin.

**Example `ethers.js` deployment script (for use with Hardhat or Node.js)**:
```javascript
// scripts/deploy.mjs
import { Wallet, Provider } from "zksync-ethers";
import * as ethers from "ethers";
import { HardhatRuntimeEnvironment } from "hardhat/types";
import { Deployer } from "@matterlabs/hardhat-zksync-deploy";

export default async function (hre: HardhatRuntimeEnvironment) {
  const provider = new Provider("https.sepolia.era.zksync.dev");
  const wallet = new Wallet(process.env.PRIVATE_KEY, provider);
  const deployer = new Deployer(hre, wallet);

  const artifact = await deployer.loadArtifact("MyContract");
  const myContract = await deployer.deploy(artifact, ["Initial Value"]);

  console.log(`Deployed to ${await myContract.getAddress()}`);
}
```
This script would be run via `npx hardhat deploy-zksync --script deploy.mjs --network zkSyncSepolia`.
