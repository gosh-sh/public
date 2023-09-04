# How to upgrade smart-contracts

In TVM, a contract code is simply regular data packed into a Tree of Cell. You can get a cell with your code using TvmCell code = tvm.code(), set a new code to your contract using tvm.setCode(newCode) (the code will be changed after the transaction completes), or even set a new contract code within the current transaction(only for current) using tvm.setCurrentCode(newCode).

### There are two challenges with contract upgrades:

The first problem is with the contract storage structure. We have tvm.setCode() and we can change the contract code at any time. But the problem is with the contract storage. All of our variables are stored in a single Tree of Cell (TvmCell + references to child cells), and Solidity simply uses a mapping to these cells, which is created during compilation.

This works like this: TVM has a register c4, where the constant state of the contract is stored, which is a cell. TVM also has a register c7, which is an array (tuple) for temporary data that is always empty at the beginning of a transaction.

Every time, at the beginning of transaction execution, the contract unpack the all the state variables from the one Cell(c4) to the tuple(c7) to access the variables just by indexes of tuple. If transaction in not pure view at the end of transaction contract pack variables back from c7 to c4.

Compiler generates unpacking function on compile time and await from contract storage some structure. So if the new c ode has a different set of variables, the contract will not be able to unpack the old state

To solve this problem, we pack our entire state into a single cell, do tvm.setCode(_newCode), then tvm.setCurrentCode(_newCode);, and pass our entire current state to the function onCodeUpgrade(oldState) of the new code. There we unpack it and write it to the new storage structure.

Let's consider an example of how this could be implemented for TokenWallet tip3:

```
contract TokenWallet {
  // 
  static address root_;
  static address owner_;
  
  uint128 balance_;
  uint32 version_;

  // ... We skipped all functions

  // Wallet owner can ask TokenRoot for the new version of WalletCode 
  function upgrade(address remainingGasTo) override external onlyOwner {
    ITokenRootUpgradeable(root_).requestUpgradeWallet{ value: 0, flag: TokenMsgFlag.REMAINING_GAS, bounce: false }(
      version_,
      owner_
    );
  }

  // In answer he receive the last version of code.   
  function acceptUpgrade(TvmCell newCode, uint32 newVersion) override external onlyRoot {
    if (version_ != newVersion) {
      // Encode all state variable to TvmCell
      TvmCell state = abi.encode(root_, owner_, balance_, version_, newVersion);
      // Set new code from the next transaction
      tvm.setcode(newCode);
      // Set new code in the current transaction
      tvm.setCurrentCode(newCode);
      // Call onCodeUpgrade of the new code.
      onCodeUpgrade(state);
    }
  }
}

// Function onCodeUpgrade of the new code
// You can call only onCodeUpgrade after the tvm.setCurrentCode()

function onCodeUpgrade(TvmCell data) private {
  // We reset the contract storage to zero because if we added any variable,
  // our storage structure would change. This call does not touch the special
  // variables _pubkey, _replayTs, _constructorFlag, and initializes
  // all other variables in the temporary register c7 to zeros.
  
  tvm.resetStorage();
  
  // Decode old state  
  (address root, address owner, uint128 balance, uint32 fromVersion, uint32 newVersion) =
        abi.decode(data, (address, address, uint128, uint32, uint32));

  // initializing our new state
  root_ = root;
  owner_ = owner;
  balance_ = balance;
  version_ = newVersion;

  // Some other logic

  // After this method contract will pack c7 to c4 with 
  // the new structure.
}
```
In principle, this is enough if we want to upgrade a standalone contract, such as TokenRoot. We have built into the contract the possibility that the storage structure may change in a new version (for example, new variables may be added).

But if we want to upgrade contracts that use distributed programming, we have a problem number two, which is the issue of how we check the sender's address. If you are not familiar with the paradigm of distributed programming as it is understood in ES, read the corresponding article in this chapter.

When a tip3 wallet transfers tokens to another tip3 wallet, the receiving wallet checks that the message has come from a legitimate contract with the same code as itself and deployed from the same root. Simplified like this:

```
function acceptTransfer(
  uint128 amount,
  address sender,
  address remainingGasTo
) override external
{
  TvmCell senderStateInit = tvm.buildStateInit({
      contr: TokenWallet, // our contract, to guide compiler how to pack static variables
      varInit: {
        root_: root_,
        owner_: walletOwner
    },
    pubkey: 0,
    code: tvm.code() // The same code as our contract has
  });
  require(msg.sender == address(tvm.hash(_buildWalletInitData(senderStateInit))), Errors.SENDER_IS_NOT_VALID_WALLET);

  tvm.rawReserve(TokenGas.TARGET_WALLET_BALANCE, 0);
  balance_ += amount;
  remainingGasTo.transfer({ value: 0, flag: TokenMsgFlag.ALL_NOT_RESERVED + TokenMsgFlag.IGNORE_ERRORS, bounce: false });
}
```
It seems that if TokenRoot initially deployed TokenWallet contracts with one code, and then we updated the code for wallets in it, then the new contracts will not be able to verify the addresses of the old ones and vice versa. This is because the address depends on the initial code and data of the contract, and if we change the code in the future, the address will remain the same.


To solve this problem, a pattern was devised where we always deploy such contracts through a small intermediary contract that is immediately updated with the current code. The whole task of this contract is to calculate the initial address from it. The accepted name for such contracts is a platform.


Here is how a platform contract for TokenWallet might look:

```
contract TokenWalletPlatform {
  // Your platform must have the same static 
  // variables as your contract has
  // or security in distributed programming 
  // will be broken
  address static root;
  address static owner;

  constructor(TvmCell walletCode, uint32 walletVersion, address sender, address remainingGasTo)
      public
  {
   // This contract can only be deployed by the root contract
   // or another platform smart contract deployed by the same root,
   // but with a different owner. This is because tip3 wallets
   // can deploy other wallets of the same token if they need
   // to transfer tokens to a wallet that has not yet been deployed.
    if (msg.sender == root || (sender.value != 0 && _getExpectedAddress(sender) == msg.sender)) {
      initialize(walletCode, walletVersion, remainingGasTo);
    } else {
      remainingGasTo.transfer({
        value: 0,
        flag: TokenMsgFlag.ALL_NOT_RESERVED + TokenMsgFlag.DESTROY_IF_ZERO,
        bounce: false
      });
    }
  }

  function _getExpectedAddress(address owner_) private view returns (address) {
    // Get expected address for the contract
    // deployed with the same platform code
    TvmCell stateInit = tvm.buildStateInit({
      contr: TokenWalletPlatform,
      varInit: {
          root: root,
          owner: owner_
      },
      pubkey: 0,
      code: tvm.code()
    });
    return address(tvm.hash(stateInit));
  }

  function initialize(TvmCell walletCode, uint32 walletVersion, address remainingGasTo) private {
      TvmCell data = abi.encode(
        root,
        owner,
        // Zero balance
        uint128(0),
        // We upgrading from zero version
        uint32(0),
        walletVersion,
        // We pass platform code to the wallet,
        // Wallet will use it to calculate address
        // of the contract deployed by using such platform
        tvm.code(),
        remainingGasTo
      );

      tvm.setcode(walletCode);
      tvm.setCurrentCode(walletCode);

      // Call onCodeUpgrade from the actual wallet code.
      onCodeUpgrade(data);
  }

  function onCodeUpgrade(TvmCell data) private {}
}
```
It turns out that our TokenWallet version 1 should receive a call to onCodeUpgrade(data) from the platform instead of the constructor:

```
contract TokenWallet {
  // Переменные уже не static
  address root;
  address owner;

  uint128 balance_;
  uint32 version_;

  // We store platformCode to verify sender's address
  TvmCell platformCode_;

  constructor()
      public
  {
    // We never deploy this code directly
    // only by onCodeUpgrade
    revert();
  }

  // ... all functions of the contract ...  //
  
  function onCodeUpgrade(TvmCell data) private {
    tvm.rawReserve(_reserve(), 2);
    tvm.resetStorage();

    // Decode the state 
    (address root, address owner, uint128 balance, uint32 fromVersion, uint32 newVersion, TvmCell platformCode, adddres remainingGasTo) = 
        abi.decode(data, (address, address, uint128, uint32, uint32, TvmCell, address));

    // Initializing state
    root_ = root;
    owner_ = owner;
    balance_ = balance;
    version_ = newVersion;
    platformCode_ = platformCode;
    
    if (remainingGasTo.value != 0 && remainingGasTo != address(this)) {
      remainingGasTo.transfer({
      value: 0,
      flag: TokenMsgFlag.ALL_NOT_RESERVED + TokenMsgFlag.IGNORE_ERRORS,
      bounce: false
      });
    }
  }
}
```
So our transfer function must look like this:
```
function acceptTransfer(
  uint128 amount,
  address sender,
  address remainingGasTo
) override external
{
  TvmCell senderStateInit = tvm.buildStateInit({
      contr: TokenWalletPlatform,
      varInit: {
        root_: root_,
        owner_: walletOwner
    },
    pubkey: 0,
    code: platformCode_
  });
  require(msg.sender == address(tvm.hash(_buildWalletInitData(senderStateInit))), Errors.SENDER_IS_NOT_VALID_WALLET);

  tvm.rawReserve(TokenGas.TARGET_WALLET_BALANCE, 0);
  balance_ += amount;
  remainingGasTo.transfer({ value: 0, flag: TokenMsgFlag.ALL_NOT_RESERVED + TokenMsgFlag.IGNORE_ERRORS, bounce: false });
}
```
So now wallets of different versions can make transfers to each other and verify each other's code. It is important to understand that the static variables of the platform contract should 1 to 1 reflect the static variables of your contract. For example, if address root_ was not a static variable and was not checked in the constructor, anyone could deploy wallets for your token with any balances.


You can read the complete code for the Upgradable tip3 token in the repository.
