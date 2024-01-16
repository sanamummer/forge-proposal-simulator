# Multisig Proposal

After adding FPS to your project dependencies, the next step is to create the first Proposal contract. In this example, we will create a proposal that deploys a new instance of `Vault.sol` and a new ERC20 token, then transfer ownership of both contracts to the multisig wallet.

Vault contract:
```solidity
pragma solidity ^0.8.0;

import { Ownable } from "@openzeppelin/access/Ownable.sol";
import { Pausable } from "@openzeppelin/security/Pausable.sol";
import { IERC20 } from "@openzeppelin/token/ERC20/IERC20.sol";

contract Vault is Ownable, Pausable {
    uint256 public LOCK_PERIOD = 1 weeks;

    struct Deposit {
        uint256 amount;
        uint256 timestamp;
    }

    mapping(address => mapping(address => Deposit)) public deposits;
    mapping(address => bool) public tokenWhitelist;

    constructor() Ownable() Pausable() {}

    function whitelistToken(address token, bool active) external onlyOwner {
        tokenWhitelist[token] = active;
    }

    function deposit(address token, uint256 amount) external whenNotPaused {
        require(tokenWhitelist[token], "Vault: token must be active");
        require(amount > 0, "Vault: amount must be greater than 0");
        require(token != address(0), "Vault: token must not be 0x0");

        Deposit storage userDeposit = deposits[token][msg.sender];
        userDeposit.amount += amount;
        userDeposit.timestamp = block.timestamp;

        IERC20(token).transferFrom(msg.sender, address(this), amount);
    }

    function withdraw(
        address token,
        address payable to,
        uint256 amount
    ) external whenNotPaused {
        require(tokenWhitelist[token], "Vault: token must be active");
        require(amount > 0, "Vault: amount must be greater than 0");
        require(token != address(0), "Vault: token must not be 0x0");
        require(
            deposits[token][msg.sender].amount >= amount,
            "Vault: insufficient balance"
        );
        require(
            deposits[token][msg.sender].timestamp + LOCK_PERIOD <
                block.timestamp,
            "Vault: lock period has not passed"
        );

        Deposit storage userDeposit = deposits[token][msg.sender];
        userDeposit.amount -= amount;

        IERC20(token).transfer(to, amount);
    }

    function pause() external onlyOwner {
        _pause();
    }

    function unpause() external onlyOwner {
        _unpause();
    }
}
```

Proposal files are located in the `proposals` folder. Create a new file called `MULTISIG_01.sol` and add the following code:

```solidity
pragma solidity ^0.8.0;

import { MultisigProposal } from "@forge-proposal-simulator/proposals/MultisigProposal.sol";
import { Addresses } from "@forge-proposal-simulator/addresses/Addresses.sol";
import { Vault } from "path/to/Vault.sol";
import { MockToken } from "path/to/MockToken.sol";

// MULTISIG_01 proposal deploys a Vault contract and an ERC20 token contract
// Then the proposal transfers ownership of both Vault and ERC20 to the multisig address
// Finally the proposal whitelist the ERC20 token in the Vault contract
contract MULTISIG_01 is MultisigProposal {
    // Returns the name of the proposal.
    function name() public pure override returns (string memory) {
        return "MULTISIG_01";
    }

    // Provides a brief description of the proposal.
    function description() public pure override returns (string memory) {
        return "Deploy Vault contract";
    }

    // Deploys a vault contract and an ERC20 token contract.
    function _deploy(Addresses addresses, address) internal override {
        // Deploy needed contracts
        Vault timelockVault = new Vault();
        MockToken token = new MockToken();

        // Add deployed contracts to the address registry
        addresses.addAddress("VAULT", address(timelockVault));
        addresses.addAddress("TOKEN_1", address(token));
    }

    // Transfers vault ownership to dev multisig.
    // Transfer token ownership to dev multisig.
    // Transfers all tokens to dev multisig.
    function _afterDeploy(
        Addresses addresses,
        address deployer
    ) internal override {
        // Get needed addresses from addresses registry
        address devMultisig = addresses.getAddress("DEV_MULTISIG");
        Vault timelockVault = Vault(addresses.getAddress("VAULT"));
        MockToken token = MockToken(addresses.getAddress("TOKEN_1"));

        // Transfer ownership of the contracts to the multisig address
        timelockVault.transferOwnership(devMultisig);
        token.transferOwnership(devMultisig);

        // Transfer tokens from deployer to multisig address
        token.transfer(devMultisig, token.balanceOf(address(deployer)));
    }

    // Sets up actions for the proposal, in this case, setting the MockToken to active.
    function _build(Addresses addresses) internal override {
        // Get vault and token addresses (deployed on _deploy step)
        address timelockVault = addresses.getAddress("VAULT");
        address token = addresses.getAddress("TOKEN_1");

        // Push action to whitelist the MockToken
        _pushAction(
            timelockVault,
            abi.encodeWithSignature(
                "whitelistToken(address,bool)",
                token,
                true
            ),
            "Set token to active"
        );
    }

    // Executes the proposal actions.
    function _run(Addresses addresses, address) internal override {
        // Call parent _run function to check if there are actions to execute
        super._run(addresses, address(0));

        // Get multisig address
        address multisig = addresses.getAddress("DEV_MULTISIG");

        // Simulates actions on Multisig address
        _simulateActions(multisig);
    }

    // Validates the post-execution state.
    function _validate(Addresses addresses, address) internal override {
        // Get needed addresses from addresses registry
        address devMultisig = addresses.getAddress("DEV_MULTISIG");
        Vault timelockVault = Vault(addresses.getAddress("VAULT"));
        MockToken token = MockToken(addresses.getAddress("TOKEN_1"));

        // Validate post-execution state
        // Vault ownership should be transferred to multisig
        assertEq(timelockVault.owner(), devMultisig);
        // Token should be whitelisted on the Vault contract
        assertTrue(timelockVault.tokenWhitelist(address(token)));
        // Vault should not be paused
        assertFalse(timelockVault.paused());
        // Token ownership should be transferred to multisig
        assertEq(token.owner(), devMultisig);
        // Token balance of multisig should be equal to total supply
        assertEq(token.balanceOf(devMultisig), token.totalSupply());
    }
}
```

Let's go through each of the functions we are overriding here.

-   `name()`: Define the name of your proposal.
-   `description()`: Provide a detailed description of your proposal.
-   `_deploy()`: Deploy any necessary contracts. This example demonstrates the
    deployment of Vault and an ERC20 token. Once the contracts are deployed,
    they are added to the `Addresses` contract by calling `addAddress()`.
-   `_build(`): Set the necessary actions for your proposal. In this example, ERC20 token is whitelisted on the Vault contract
-   `_run()`: Execute the proposal actions outlined in the `_build()` step. This
    function performs a call to `simulateActions()` from the inherited
    `MultisigProposal` contract. Internally, `_simulateActions()` simulates a call to the [Multicall3](https://www.multicall3.com/) contract with the calldata generated from the actions set up in the build step.
-   `_validate()`: This final step is crucial for validating the post-execution state. It ensures that the multisig is the new owner of Vault and token, the tokens were transferred to multisig and the token was whitelisted on the Vault contract

With the first proposal contract prepared, it's time to proceed with execution. There are two options available:

1. **Using `foundry test`**: Details on this method can be found in the [integration-tests.md](../testing/integration-tests.md "mention") section.
2. **Using `foundry script`**: This is the chosen method for this scenario.

Before proceeding with the `foundry script`, it is necessary to set up the
[Addresses](../overview/architecture/addresses.md) contract. The next step
involves creating an `addresses.json` file.

```json
[
    {
        "addr": "0x3dd46846eed8D147841AE162C8425c08BD8E1b41",
        "name": "DEV_MULTISIG",
        "chainId": 31337
    }
]
```

With the JSON file prepared for use with `Addresses.sol`, the next step is to create a script that inherits from `ScriptSuite`.

```solidity
pragma solidity ^0.8.0;

import { ScriptSuite } from "@forge-proposal-simulator/script/ScriptSuite.s.sol";
import {MULTISIG_01} from "proposals/MULTISIG_01.sol";

// @notice MultisigScript is a script that run MULTISIG_01 proposal
// MULTISIG_01 proposal deploys a Vault contract and an ERC20 token contract
// Then the proposal transfers ownership of both Vault and ERC20 to the multisig address
// Finally the proposal whitelist the ERC20 token in the Vault contract
// @dev Use this script to simulates or run a single proposal
// Use this as a template to create your own script
// `forge script script/MultisigScript.s.sol:MultisigScript -vvvv --rpc-url {rpc} --broadcast --verify --etherscan-api-key {key}`
contract MultisigScript is ScriptSuite {
    string public constant ADDRESSES_PATH = "./addresses/addresses.json";

    constructor() ScriptSuite(ADDRESSES_PATH, new MULTISIG_01()) {}

    function run() public override {
        proposal.setDebug(true);

        // Execute proposal
        super.run();
    }
}
```

Ensure that the `DEV_MULTISIG` address corresponds to a valid Multisig Gnosis Safe contract. If this is not the case, the script will fail, displaying the error: `Multisig address doesn't match Gnosis Safe contract bytecode`.

For those who wish to complete this tutorial on a local blockchain without the necessity of deploying a Gnosis Safe Account, it is possible to modify `MultisigScript` as follows:

```solidity
contract MultisigScript is ScriptSuite {
    string public constant ADDRESSES_PATH = "./addresses/addresses.json";

    constructor() ScriptSuite(ADDRESSES_PATH, new MULTISIG_01()) {}

    bytes public constant SAFE_BYTECODE =
        hex"608060405273ffffffffffffffffffffffffffffffffffffffff600054167fa619486e0000000000000000000000000000000000000000000000000000000060003514156050578060005260206000f35b3660008037600080366000845af43d6000803e60008114156070573d6000fd5b3d6000f3fea2646970667358221220d1429297349653a4918076d650332de1a1068c5f3e07c5c82360c277770b955264736f6c63430007060033";

    function run() public override {
        proposal.setDebug(true);

        // Set Gnosis Safe bytecode
        vm.etch(addresses.getAddress("DEV_MULTISIG"), SAFE_BYTECODE);

        // Execute proposal
        super.run();
    }
}
```

Running the script:

```sh
forge script path/to/MultisigScript.s.sol
```

The script will output the following:

```sh
  Addresses before running proposal:
  DEV_MULTISIG 0x3dd46846eed8D147841AE162C8425c08BD8E1b41
  TEAM_MULTISIG 0x7da82C7AB4771ff031b66538D2fB9b0B047f6CF9
  PROTOCOL_TIMELOCK 0x1a9C8182C09F50C8318d769245beA52c32BE35BC
  DAO_MULTISIG 0x10A19e7eE7d7F8a52822f6817de8ea18204F2e4f
  TIMELOCK_PROPOSER 0x10A19e7eE7d7F8a52822f6817de8ea18204F2e4f
  TIMELOCK_EXECUTOR 0x10A19e7eE7d7F8a52822f6817de8ea18204F2e4f
  Calldata:
  0x252dba4200000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000001000000000000000000000000000000000000000000000000000000000000002000000000000000000000000090193c961a926261b756d1e5bb255e67ff9498a1000000000000000000000000000000000000000000000000000000000000004000000000000000000000000000000000000000000000000000000000000000440ffb1d8b000000000000000000000000a8452ec99ce0c64f20701db7dd3abdb607c00496000000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000
  Multicall result:
  0x00000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000000000040000000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000000
  Addresses after running proposals:
  DEV_MULTISIG 0x3dd46846eed8D147841AE162C8425c08BD8E1b41
  TEAM_MULTISIG 0x7da82C7AB4771ff031b66538D2fB9b0B047f6CF9
  PROTOCOL_TIMELOCK 0x1a9C8182C09F50C8318d769245beA52c32BE35BC
  DAO_MULTISIG 0x10A19e7eE7d7F8a52822f6817de8ea18204F2e4f
  TIMELOCK_PROPOSER 0x10A19e7eE7d7F8a52822f6817de8ea18204F2e4f
  TIMELOCK_EXECUTOR 0x10A19e7eE7d7F8a52822f6817de8ea18204F2e4f
  VAULT 0x90193C961A926261B756D1E5bb255e67ff9498A1
  TOKEN_1 0xA8452Ec99ce0C64f20701dB7dD3abDb607c00496
```

A signer from the multisig address can check whether the calldata proposed on the multisig matches the calldata obtained from the call. It is important to note that two new addresses have been added to the `Addresses.sol` storage. These addresses are not included in the JSON file and must be added manually for accuracy.