Normally, when we deploy smart contracts on the blockchain, it is impossible to edit the code deployed. And it should be totally fine that way. The fact that it could not be deployed to replace the existing codes means that that particular contract can not be modified afterward. (this increase safety of people who interact with that contract)

Immutability comes with the drawback that bugs will not be fixed, gas optimizations won’t be implemented, existing functionality will not be improved

---

## What is an Upgradable Contract

> An Upgradable contract is a contract that can be (kind of) altered, after the deployment. At the time this article was written, to use an upgradeable smart contract, there is a tool, or plugin, to help us build. This plugin is introduced by [OpenZeppelin](https://www.openzeppelin.com/).

---

## 🤔 Why an Upgrade

Here’s what you’d need to do to fix a bug in a contract you cannot upgrade:

- Deploy a new version of the contract

- Manually migrate all state from the old one contract to the new one (which can be very expensive in terms of gas fees!)

- Update all contracts that interacted with the old contract to use the address of the new one

- Reach out to all your users and convince them to start using the new deployment (and handle both contracts being used simultaneously, as users are slow to migrate)

---

## 🙋 How does it work

We must first comprehend what a delegate call is in order to grasp how it operates.

> `delegatecall` is a low level function similar to call. When contract A executes delegatecall to contract B, B's code is executed with contract A's storage

<br>

First, an external caller makes a function call to the proxy. Second, the proxy delegates the call to the delegate, where the function code is located. Third, the result is returned to the proxy, which forwards it to the caller. Because the `delegatecall` is used to delegate the call, the called function is executed in the context of the proxy. This means that the storage of the proxy is used for function execution, thus resulting in the limitation that the storage of the delegate contract has to be append only. The `delegatecall` opcode was introduced in [EIP-7](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-7.md).

---

## 📃 Let's start making a Upgradable ERC-20 Contract

### **Step 1: Set up Development Environment**

Firstly initialize a npm project using

```
npm init -y
```

Install Hardhat

```
npm install --save-dev hardhat
```

and then create a Hardhat project and install necessary dependencies

```
npx hardhat
```

```
npm install --save-dev @openzeppelin/contracts-upgradeable @nomiclabs/hardhat-ethers @openzeppelin/hardhat-upgrades dotenv
```

---

### **Step 2: Setup Hardhat config**

Create a new file called `.env` in your project root and add the following details.

```
URL = 'URL_HERE'
PRIVATE_KEY = 'PRIVATE_KEY_HERE'
```

You can get your RPC url from [Alchemy](https://dashboard.alchemyapi.io/).
For this tutorial we will be using Polygon Mumbai Testnet.
Make sure you have some testnet funds in your account.

Open up `hardhat.config.js` and replace the existing code with this

```javascript
require("@nomiclabs/hardhat-ethers");
require("@openzeppelin/hardhat-upgrades");
require("dotenv").config();

module.exports = {
  solidity: "0.8.7",
  networks: {
    mumbai: {
      url: process.env.URL,
      accounts: [process.env.PRIVATE_KEY],
    },
  },
};
```

---

### **Step 3: Create a new ERC20 Upgradable Contract.**

Open the `Contracts` Folder and create a new file called `ERC20UpgradeableV1.sol`.

First we will make a Upgradable ERC-20 Contract using [OpenZeppelin Contract Wizard](https://docs.openzeppelin.com/contracts/4.x/wizard) using the following configuration.

<br>

![Contract Wizard](assets/1.png)

Here is a sample code for a upgradable contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.7;

import "@openzeppelin/contracts-upgradeable/token/ERC20/ERC20Upgradeable.sol";
import "@openzeppelin/contracts-upgradeable/token/ERC20/extensions/ERC20BurnableUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/security/PausableUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";

contract ERC20UpgradableV1 is Initializable, ERC20Upgradeable, ERC20BurnableUpgradeable, PausableUpgradeable, OwnableUpgradeable {
    /// @custom:oz-upgrades-unsafe-allow constructor
    constructor() {
        _disableInitializers();
    }

    function initialize() initializer public {
        __ERC20_init("ERC20Upgradable", "EUC");
        __ERC20Burnable_init();
        __Pausable_init();
        __Ownable_init();
    }

    function pause() public onlyOwner {
        _pause();
    }

    function unpause() public onlyOwner {
        _unpause();
    }

    function mint(address to, uint256 amount) public {
        _mint(to, amount);
    }

    function _beforeTokenTransfer(address from, address to, uint256 amount)
        internal
        whenNotPaused
        override
    {
        super._beforeTokenTransfer(from, to, amount);
    }
}
```

The above ERC-20 contract is a normal contract with `mint`,`pause` and `burn` properties.

---

### **Step 4: Deploy Contract.**

Now is the time to write our deploy Script.
For this we use [deployProxy](https://docs.openzeppelin.com/upgrades-plugins/1.x/api-hardhat-upgrades#deploy-proxy) function instead of normal deploy function.

Create a new file called `deploy.js` in the scripts folder.

```javascript
const { ethers, upgrades } = require("hardhat");

async function main() {
  const ERC20UpgradableV1 = await ethers.getContractFactory(
    "ERC20UpgradableV1"
  );
  console.log("Deploying ERC20UpgradableV1...");
  const contract = await upgrades.deployProxy(ERC20UpgradableV1, [], {
    initializer: "initialize",
    kind: "transparent",
  });
  await contract.deployed();
  console.log("ERC20UpgradableV1 deployed to:", contract.address);
}

main();
```

now run the script with the following command

```
npx hardhat run scripts/deploy.js --network mumbai
```

The output should be something like this

```
Compiled 11 Solidity files successfully
Deploying ERC20UpgradableV1...
ERC20UpgradableV1 deployed to: 0xC81cBaB47B1e6D6d20d4742721e29f22C5835dcB
```

---

### **Step 5: Testing Contract**

Now we have to test our smart contract so in terminal run

```
npx hardhat console --network mumbai
```

```
const Contract = await ethers.getContractFactory('ERC20UpgradableV1');
```

```
const contract = await Contract.attach('0xC81cBaB47B1e6D6d20d4742721e29f22C5835dcB');
```

The above two commands get the contract and connect to it

```
await contract.mint('0xBF4979305B43B0eB5Bb6a5C67ffB89408803d3e1',ethers.utils.parseEther("100.0"));
```

The above command will mint 100 Tokens to `0xBF4...3e1`

![Balance](assets/2.png)

---

### **Step 6: Create a V2 Contract**

Let's imagine we now wish to add a new feature to our contract, say `"whitelist"` On a conventional contract, we would not be able to modify the code, but on an upgradeable contract, we could simply direct calls to another implementation contract through the proxy contract.

In the contracts folder, let's create a new contract named `"ERC20UpgradableV2.sol"` and add some rudimentary whitelist functionality by mapping an address to a boolean and then determining if the address is whitelisted or not during the mint function.

Here's the code

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.7;

import "@openzeppelin/contracts-upgradeable/token/ERC20/ERC20Upgradeable.sol";
import "@openzeppelin/contracts-upgradeable/token/ERC20/extensions/ERC20BurnableUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/security/PausableUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";

contract ERC20UpgradableV2 is Initializable, ERC20Upgradeable, ERC20BurnableUpgradeable, PausableUpgradeable, OwnableUpgradeable {
    /// @custom:oz-upgrades-unsafe-allow constructor
    constructor() {
        _disableInitializers();
    }

    mapping(address => bool) whitelistedAddresses;

    function addUser(address _addressToWhitelist) public onlyOwner {
        whitelistedAddresses[_addressToWhitelist] = true;
    }

    function verifyUser(address _whitelistedAddress) public view returns(bool) {
        bool userIsWhitelisted = whitelistedAddresses[_whitelistedAddress];
        return userIsWhitelisted;
    }

    function initialize() initializer public {
        __ERC20_init("ERC20Upgradable", "EUC");
        __ERC20Burnable_init();
        __Pausable_init();
        __Ownable_init();
    }

    function pause() public onlyOwner {
        _pause();
    }

    function unpause() public onlyOwner {
        _unpause();
    }

    function mint(address to, uint256 amount) public {
        require(verifyUser(to));
        _mint(to, amount);
    }

    function _beforeTokenTransfer(address from, address to, uint256 amount)
        internal
        whenNotPaused
        override
    {
        super._beforeTokenTransfer(from, to, amount);
    }
}
```

---

### **Step 7: Upgrading the Contract**

Make a new file called `upgrade.js` in the scripts folder and add the following code to it.

```javascript
const { ethers, upgrades } = require("hardhat");

async function main() {
  const ERC20UpgradableV2 = await ethers.getContractFactory(
    "ERC20UpgradableV2"
  );
  console.log("Upgrading ERC20UpgradableV1...");
  await upgrades.upgradeProxy(
    "0xC81cBaB47B1e6D6d20d4742721e29f22C5835dcB",
    ERC20UpgradableV2
  );
  console.log("Upgraded Successfully");
}

main();
```

Run this file with the following command

```
npx hardhat run scripts/upgrade.js --network mumbai
```

You should get the following output

```
Upgrading ERC20UpgradableV1...
Upgraded Successfully
```

Let's do some tests now that the Contract has successfully been upgraded and whitelist capability has been added.

---

### **Step 8: Running Tests for Upgraded Contract**

Now we have to test our smart contract so in terminal run

```
npx hardhat console --network mumbai
```

```
const Contract = await ethers.getContractFactory('ERC20UpgradableV2');
```

```
const contract = await Contract.attach('0xC81cBaB47B1e6D6d20d4742721e29f22C5835dcB');
```

The above two commands get the contract and connect to it

Now let's check if we are whitelisted or not

```
await contract.verifyUser('0xBF4979305B43B0eB5Bb6a5C67ffB89408803d3e1');
```

The result should be `false`

Let's try to mint tokens without having a whitelist and see if we get an error or not

```
await contract.mint('0xBF4979305B43B0eB5Bb6a5C67ffB89408803d3e1',ethers.utils.parseEther("100.0"));
```

We get the following error

```
Error: cannot estimate gas; transaction may fail or may require manual gas limit
```

which indicates that the V2 contract handles execution, and because we lack whitelist access, we are unable to mint tokens.

Let's now give ourselves whitelist access and then try to mint tokens.

```
await contract.addUser('0xBF4979305B43B0eB5Bb6a5C67ffB89408803d3e1');
```

Now wait about 30 seconds till the transaction gets hashed and then again verify that the address has whitelist access.

```
await contract.verifyUser('0xBF4979305B43B0eB5Bb6a5C67ffB89408803d3e1');
```

This time we should get `true` and now we can mint tokens

```
await contract.mint('0xBF4979305B43B0eB5Bb6a5C67ffB89408803d3e1',ethers.utils.parseEther("100.0"));
```

## ![Balance](assets/3.png)

---
