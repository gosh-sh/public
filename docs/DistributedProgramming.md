# Distributed programming
Distributed Programming was first described in Free TON White Paper by Mitja Goroshevsky.
Distributed Programming was used to create TIP-3 Token Standard as well as Destributed Name System DeNS and is the main programming paradigm used in TVM blockchains.

In this article, we will examine what distributed programming is and what problems it solves, using the example of the TIP-3 token. This is a distributed alternative to ERC-20, where each user has their own smart contract for storing their balance, and users can send tokens to each other directly without a central node.


What problems we solve with distributed programming:
### Throughput
If we have one smart contract, its throughput is limited by the gas limit of one processing thread. Since a thread releases its block every few seconds, the throughput of one thread is the same or a bit less than that of chains in a classic blockchain. So if we want to get more than 20-50 transactions per second in your application, we need to shard it. Fast block release is important due to asynchronicity and multithreading - if we need to engage N contracts for our action, we may have to wait for N blocks for finality in the worst case.

### Paid storage
All blockchains have the problem of state bloat. Validators need to store all the data of their blockchain (or their shard) in order to validate transactions in blocks. If a blockchain wants to maintain decentralization and expects to exist for decades, they have to artificially limit the writing speed to the blockchain, and as a result, users have to compete with each other in an auction for the right to write their data to the blockchain.


TVM blockchains decided to take a different approach, and not limit the writing speed to the blockchain, but each account must pay a rental fee for storing its state, which is linear to the size of the code and data of this contract. When a contract runs out of money to pay for storage, it is first frozen and then deleted. In this way, the size of the state remains compact.


Now let's imagine that we have an erc-20 like mem-token contract with one large hashmap of token holders. Everyone who has ever bought this token will be recorded in this contract. Eventually, we will have a situation where users with large balances will have to pay for the storage of those who have only pennies left on their accounts. To avoid the need for programmers to come up with a way to clean up the garbage in the smart contract, in TVM blockchains it is customary to deploy a small wallet on each user's balance, which only stores their balance. In this way, each user pays for themselves, and at the same time we get a uniform distribution of contracts across threads, which gives them enormous throughput.


This applies not only to tokens, but to all other applications as well. For example, each proposal in a DAO can be a separate contract, each trading pair in an AMM dex, each game in chess, etc.



## How it works in practice
We will examine how a tip-3 token works using code examples to understand how distributed programming works and why it is safe to transfer tokens peer-to-peer without a central node. Our tip-3 token is the same in terms of interfaces as the real one, but it is a greatly simplified example that does not support all functions of the standard. In production, it is necessary to use the tip-3 implementation from broxus. We will discuss how to work with it conveniently in the next article on locklift.

Full code of this example available for everscale-inpage-provider and eversdk.

Our token consists of 2 contracts:

TokenRoot.tsol is controlled by whoever deployed the token, allowing them to print tokens. Additionally, anyone can call it to deploy the wallets of individual users.
TokenWallet.tsol is a wallet contract for individual users. Yes, each user has their own small contract that stores their token balance.

A few thoughts to keep in mind while looking at the code:

The contract address is the hash (contract code + static variables).
If we know the wallet contract code and its initial variables (the root address and the owner contract address), then we can calculate what address this contract will have.
When one wallet receives a message from another wallet, it can determine from the sender's address whether the sending wallet has exactly the same code, to see if it really has the tokens it is sending you.

### TokenRoot.tsol
```
pragma ton-solidity >= 0.62.0;

import "./TokenWallet.tsol";
import "./interfaces/ITokenRoot.tsol";
import "./libraries/Errors.tsol";
import "./libraries/TokenMsgFlag.tsol";


contract TokenRoot is ITokenRoot {
  
  // just token metadata
  string static name_;
  string static symbol_;
  
  // decimals refers to how divisible a token can be, from 0
  // (not at all divisible) to 18 (pretty much continuous) 
  // and even higher if required.
  // native ever has decimals = 9, so 1 ever = 1_000_000_000 nano evers.
  // just google it if not familiar with token decimals.
  uint8 static decimals_;
  
  // owner of the root
  address static rootOwner_;
  
  // The code of the wallet contract is needed to deploy 
  // the wallet contract.
  // In the tvm the code is also stored in the TvmCell 
  // and it can be sent via messages.
  TvmCell static walletCode_;
  
  
  // total minted amount
  uint128 totalSupply_;
  
  constructor(address remainingGasTo) public {
    // Pubkey for this smart contract must be not set
    require(tvm.pubkey() == 0, Errors.NON_ZERO_PUBLIC_KEY);
    
    // Root owner must be set
    require(rootOwner_.value != 0, Errors.OWNER_IS_NOT_SET);
    
    // This constructor must be called from root owner
    require(msg.sender == rootOwner_, Errors.NOT_OWNER);
  
    // We reserve 1 EVER on this account.
    // It is like send a message with value to your self.
    tvm.rawReserve(1 ever, 0);
  
    if (remainingGasTo.value != 0) {
      // transfer all value are left after reserve to the remainingGasTo
      remainingGasTo.transfer({
        value: 0,
        flag: 128,
        bounce: false
      });
    }
  }
  
  modifier onlyRootOwner() {
    require(rootOwner_ == msg.sender, Errors.NOT_OWNER);
    _;
  }
  
  function _buildWalletInitData(address walletOwner) internal view returns (TvmCell) {
    // StateInit - the message deploying the contract where we establish 
    // the code the contract and its static variables.
    // Essentially the hash(stateInit) is the contract address.
    // The contract address depends on the code and the initial variables.
    // So we can determine the contract address just by knowing its code
    // and initial variables (not those that are sent in the constructor).
    
    // Pay attention on what the wallet address depend on.
    // He is depend on root_address(this), wallet code and the owner's address.
    
    return tvm.buildStateInit({
      contr: TokenWallet,  // Target contract, to guide compiler how to pack static variables
      varInit: {
        root_: address(this),
        owner_: walletOwner
      },
      pubkey: 0,
      code: walletCode_
    });
  }
  
  function _deployWallet(
    TvmCell stateInit,
    uint128 deployWalletValue
  ) private returns (address) {
    // Here we create one message that will deploy the contract
    // (if the contract is already deployed , nothing will happen)
    // also this message will call the constructor
    // () without arguments .
    address wallet = new TokenWallet {
      stateInit: stateInit,
      value: deployWalletValue, // the amount of native coins we are sending with the message
      wid: address(this).wid, // the same workchain id
      // this flag denotes that we are paying for the creation
      // of the message from the contract balance separately
      // not from the value we sending
      flag: TokenMsgFlag.SENDER_PAYS_FEES
    }();
    
    return wallet;
  }
  
  function deployWallet(
    address walletOwner,
    uint128 deployWalletValue
  ) override public responsible returns (address tokenWallet)
  {
    // This is public function. Anyone can call it to deploy 
    // a token wallet for any owner
    // Also this function marked as responsible
    // so it will send back address of the deployed token wallet
    
    require(walletOwner.value != 0, Errors.WRONG_WALLET_OWNER);
    
    // We always reserve some amount of tokens on the account balance to keep it
    tvm.rawReserve(1 ever, 0);
    
    // Calculate the wallet initial data (code + static variables)
    TvmCell stateInit = _buildWalletInitData(walletOwner);
    
    // Deploy the wallet and get their address
    tokenWallet = _deployWallet(stateInit, deployWalletValue);
    
    // This function marked as responsible
    // Responsible is the special sugar for async calls which one waiting for the answer.
    // When someone calls responsible he must specify callback -
    // a function of the sender contract with the same arguments as
    // the responsible function returns.
    // solidity adds a hidden variable in the responsible function signature - uint32 answerId
    // AnswerID is not a unique id of the call, this is just a functionID that one must
    // Be called in the back message with variables this function return.
    
    // Look at TokenDice.tsol at the end of tihs article
    // for example how to call responsible function from the another smart contract
    
    // So the line below will be compiled to something like (pseudocode)
    // msg.sender.call({ 
    //  value: 0,
    //	flag: TokenMsgFlag.ALL_NOT_RESERVED - 128, 
    //	bounce: false,
    //	functionToCall: asnwerID,
    //	values: tokenWallet
    // })
    return { value: 0, flag: TokenMsgFlag.ALL_NOT_RESERVED, bounce: false } tokenWallet;
    
    // So we have three action in this function:
    // 1. tvm.rawReserve(1 ever, 0);
    // 2. sending an internal message to deploy a new wallet
    // 3. sending an internal message back to the sender with the answer for responsible with the address of deployed wallet
  }
  
  // Minting new tokens
  function mint(
    uint128 amount,
    address recipient,
    uint128 deployWalletValue,
    address remainingGasTo,
    bool notify,
    TvmCell payload
  ) override external onlyRootOwner {
    // Notify is a special flag - is the wallet must send a notify message to the contract owner
    // with information that he is accepted new tokens from the root
    // Payload - any data to send to the wallet owner with the notification method
    
    // Notify and Payload are necessary to build big decentralised applications in the
    // async blockchain. We use it in the TokenDice.tsol example at the end of this article
    
    require(amount > 0, Errors.WRONG_AMOUNT);
    require(recipient.value != 0, Errors.WRONG_RECIPIENT);
    
    // We always reserve some amount of tokens on the account balance to keep it
    tvm.rawReserve(1 ever, 0);
    
    TvmCell stateInit = _buildWalletInitData(recipient);
    
    // Be aware, address recipient - is not an address of target wallet contract
    // recipient is the address of the wallet owner!
    // The same logic also in EverWallet extension.
    // When you send tokens to someone you specify the wallet owner address,
    // wallet address calculate under the hood.
    
    address recipientWallet;
    
    if(deployWalletValue > 0) {
      // We need to deploy wallet first
      recipientWallet = _deployWallet(stateInit, deployWalletValue);
    } else {
      // Wallet is already deployed. Just calculate the address.
      // Address of the wallet for some owner is always the same
      recipientWallet = address(tvm.hash(stateInit));
    }
    
    totalSupply_ += amount;
    
    // There we set bounce: true
    // So if any error occurs in the transaction invoked by this message
    // Special bounce message will be created and we will handle it
    // in the onBounce function of this smart contract
    TokenWallet(recipientWallet).acceptMint{ value: 0, flag: TokenMsgFlag.ALL_NOT_RESERVED, bounce: true } (
      amount,
      remainingGasTo,
      notify,
      payload
    );
  }
  
  function _getExpectedWalletAddress(address walletOwner) internal view returns (address) {
    return address(tvm.hash(_buildWalletInitData(walletOwner)));
  }
  
  function rootOwner()
    override
    external
    view
    responsible
    returns (address)
  {
    // Responsible function to get rootOwner_ from another contract.
    return { value: 0, flag: TokenMsgFlag.REMAINING_GAS, bounce: false } rootOwner_;
  }
  
  function walletOf(address walletOwner)
    override
    public
    view
    responsible
    returns (address)
  {
    // Responsible function to derive wallet address for the owner
    require(walletOwner.value != 0, Errors.WRONG_WALLET_OWNER);
    return { value: 0, flag: TokenMsgFlag.REMAINING_GAS, bounce: false } _getExpectedWalletAddress(walletOwner);
  }
  
  function totalSupply() override external view responsible returns (uint128) {
    // Responsible function to get current total supply.
    return { value: 0, flag: TokenMsgFlag.REMAINING_GAS, bounce: false } totalSupply_;
  }
  
  onBounce(TvmSlice slice) external {
    // This is a utility function for handling errors. In the mint function we did not check
    // if the contract was deployed at the destination address.
    
    // If the internal message have bounce: true and an exception occurs
    // or the destination contract does not exist,
    // then automatically (if there is enough money  attached to the message)
    // a return message is sent with a call to the onBounce function.
    
    // This onBounce message has TvmSlice slice - which one contain functionID
    // and arguments was attached to the original message.
    
    // Now there is a stupid limitation and arguments are cutting after the first 224 bites
    // But this will be fixed soon.
    
    
    // We use this function to show you how to handle a situation
    // when tokens are minted to a non-existing address and to subtract from the total_supply
    // as the tokens were not printed.
    
    // This function cannot just be called, the message must have a special bounced: true flag,
    // which cannot be added manually when sending. There is no need to do additional checks that we actually sent
    // the message. So bad actor can not subtract from the total supply by sending unexpected bounced message.
    
    // But there is still some attack vectors on onBounce
    // which one will we cover in the "The danger of the responsible" page of this chapter.
    
    // decode functionID - what function we tried to call.
    // Under the hood functionID is first 32 bits of SHA256(functionName + functionArguments)
    // with first bit set to 0 for internal messages.
    
    uint32 functionId = slice.decode(uint32);
    if (functionId == tvm.functionId(TokenWallet.acceptMint)) {
      // decode amount of tokens we tried to mint
      uint128 latest_bounced_tokens = slice.decode(uint128);
      // decrease the total supply because mint was failed.
      totalSupply_ -= latest_bounced_tokens;
    }
  }
}
```

## TokenWallet.tsol
```
pragma ton-solidity >= 0.62.0;

import "./interfaces/ITokenWallet.tsol";
import "./interfaces/ITokenRoot.tsol";
import "./interfaces/IAcceptTokensTransferCallback.tsol";
import "./interfaces/IAcceptTokensMintCallback.tsol";

import "./libraries/Errors.tsol";
import "./libraries/TokenMsgFlag.tsol";


contract TokenWallet is ITokenWallet {
  
  address static root_;
  address static owner_;
  
  uint128 balance_;
  
  constructor() public {
    require(tvm.pubkey() == 0, Errors.NON_ZERO_PUBLIC_KEY);
    require(owner_.value != 0, Errors.WRONG_WALLET_OWNER);
  }
  
  modifier onlyRoot() {
    require(root_ == msg.sender, Errors.WRONG_ROOT_OWNER);
    _;
  }
  
  modifier onlyOwner() {
    require(owner_ == msg.sender, Errors.NOT_OWNER);
    _;
  }
  
  function _buildWalletInitData(address walletOwner) internal view returns (TvmCell) {
    // Build TokenWallet StateInit for walletOwner
    return tvm.buildStateInit({
      contr: TokenWallet, // our contract, to guide compiler how to pack static variables
      varInit: {
        root_: root_,
        owner_: walletOwner
      },
      pubkey: 0,
      code: tvm.code() // The same code as our contract has
    });
  }
  
  
  function transfer(
    uint128 amount,
    address recipient,
    uint128 deployWalletValue,
    address remainingGasTo,
    bool notify,
    TvmCell payload
  ) override external onlyOwner
  {
    // With this method we can send tokens to any similar wallet
    // directly. When doing this we can say that we want to first
    // deploy this wallet.
    
    // Be aware, address recipient - is not an address of target wallet contract
    // recipient is the address of the wallet owner!
    // The same logic also in EverWallet extension.
    // When you send tokens to someone to specify the wallet owner address,
    // wallet address calculate under the hood.

    // Notify and Payload are necessary to build big decentralised applications in the
    // async blockchain. We use it in the TokenDice.tsol example at the end of this article
    
    require(amount > 0, Errors.WRONG_AMOUNT);
    require(amount <= balance_, Errors.NOT_ENOUGH_BALANCE);
    require(recipient.value != 0 && recipient != owner_, Errors.WRONG_RECIPIENT);
    
    // We always reserve 0.1 ever on this contract.
    tvm.rawReserve(0.1 ever, 0);
    
    // Build state init for target wallet
    TvmCell stateInit = _buildWalletInitData(recipient);
    
    address recipientWallet;
    
    if (deployWalletValue > 0) {
      // Deploy target wallet contract and get the address
      recipientWallet = new TokenWallet {
        stateInit: stateInit,
        value: deployWalletValue,
        wid: address(this).wid,
        flag: TokenMsgFlag.SENDER_PAYS_FEES
      }();
    } else {
      // Just calculate address
      recipientWallet = address(tvm.hash(stateInit));
    }
    
    // Decrease our balance
    balance_ -= amount;
    
    // Call acceptTransfer method of the target wallet contract
    ITokenWallet(recipientWallet).acceptTransfer{ value: 0, flag: TokenMsgFlag.ALL_NOT_RESERVED, bounce: true }
    (
      amount,
      owner_,
      remainingGasTo,
      notify,
      payload
    );
  }
  
  function acceptTransfer(
    uint128 amount,
    address sender,
    address remainingGasTo,
    bool notify,
    TvmCell payload
  ) override external
  {
    // Transfer accepting function. This is a very nice concept.
    // We can send tokens directly from one wallet
    // to another because in ES a contract address is a uniquely
    // computed value. We can check that the contract that is
    // calling us is the same kind of contract as ours and has the same
    // Root and code. So we know for sure if the contract calls us
    // these tokens are real and come from the contract root or another wallet
    
    // Check msg.sender address has the same code and the same root
    require(msg.sender == address(tvm.hash(_buildWalletInitData(sender))), Errors.SENDER_IS_NOT_VALID_WALLET);
    
    // We always reserve 0.1 ever on this contract.
    tvm.rawReserve(0.1 ever, 0);
    
    balance_ += amount;
    
    // This part useful for building complex application with tip3
    // If notify set to true wallet will send callback with attached payload
    // to wallet's owner
    // Wallet owner can be smart contract with custom logic
    // We will cover this part in "Build a decentralized application"
    if (notify) {
      IAcceptTokensTransferCallback(owner_).onAcceptTokensTransfer{
        value: 0,
        flag: TokenMsgFlag.ALL_NOT_RESERVED + TokenMsgFlag.IGNORE_ERRORS,
        bounce: false
      } (
        root_,
        amount,
        sender,
        msg.sender,
        remainingGasTo,
        payload
      );
    } else {
      remainingGasTo.transfer({ value: 0, flag: TokenMsgFlag.ALL_NOT_RESERVED + TokenMsgFlag.IGNORE_ERRORS, bounce: false });
    }
  }
  
  function acceptMint(
    uint128 amount,
    address remainingGasTo,
    bool notify,
    TvmCell payload
  ) override external onlyRoot
  {
    // Just accept new tokens from the root
    
    // We always reserve 0.1 ever on this contract.
    tvm.rawReserve(0.1 ever, 0);
    
    balance_ += amount;
    
    // Notify wallet owner about new minted tokens
    if (notify) {
      IAcceptTokensMintCallback(owner_).onAcceptTokensMint{
        value: 0,
        bounce: false,
        flag: TokenMsgFlag.ALL_NOT_RESERVED + TokenMsgFlag.IGNORE_ERRORS
      }(
        root_,
        amount,
        remainingGasTo,
        payload
      );
    } else if (remainingGasTo.value != 0 && remainingGasTo != address(this)) {
      remainingGasTo.transfer({ value: 0, flag: TokenMsgFlag.ALL_NOT_RESERVED + TokenMsgFlag.IGNORE_ERRORS, bounce: false });
    }
  }
  
  function owner() external override view responsible returns (address) {
    // Just responsible function to get a token wallet owner
    return { value: 0, flag: TokenMsgFlag.REMAINING_GAS, bounce: false } owner_;
  }
  
  function balance() override external view responsible returns (uint128) {
    return { value: 0, flag: TokenMsgFlag.REMAINING_GAS, bounce: false } balance_;
  }
  
  onBounce(TvmSlice body) external {
    uint32 functionId = body.decode(uint32);
    if (functionId == tvm.functionId(ITokenWallet.acceptTransfer)) {
      uint128 amount = body.decode(uint128);
      balance_ += amount;
    }
  }
}
```

After carefully reading the above examples and playing with the code available at everscale-inpage-provider and eversdk, you should understand the principles of distributed programming. How contracts are deployed for different users/entities in TVM blockchains and most importantly, why it is secure.

If you have it all figured out in your head, I hope you also appreciated the beauty of such a solution with distributed contracts. It solves several problems at once:

There is no one big map where all balances are stored. Loading such a contract from the disk of validators is as fast as possible.
Each user deploys their own separate wallet and pays for the storage of their data in the state of the blockchain.
Thanks to the direct transfer of money between wallets, this maximally evenly distributes the load across the entire blockchain. Each wallet falls into a random thread, depending on the wallet address. So if there is a sudden increase in the number of transfers, one thread performance not be the bottleneck for our application.

As a bonus we rewrote our dice contract to be used with tip-3 tokens. Please read this contract to figure out how to use transfer payload in real application.
```
pragma ton-solidity >= 0.62.0;

import "./interfaces/ITokenRoot.tsol";
import "./interfaces/ITokenWallet.tsol";
import "./interfaces/IAcceptTokensTransferCallback.tsol";
import "./interfaces/IAcceptTokensMintCallback.tsol";
import "./libraries/TokenMsgFlag.tsol";

library DiceErrors {
  uint16 constant NOT_OWNER                                       = 1000;
  uint16 constant NON_ZERO_PUBLIC_KEY                             = 1001;
  uint16 constant NOT_ENOUGH_BALANCE                              = 1002;
  uint16 constant INVALID_TOKEN_ROOT                              = 1003;
  uint16 constant INVALID_TOKEN_WALLET                            = 1004;
}

contract TokenDice is IAcceptTokensTransferCallback, IAcceptTokensMintCallback {
  
  address static tokenRoot_;
  address static owner_;
  
  address public tokenWallet_;
  uint128 public balance_;
  
  event Game(address player, uint8 bet, uint8 result,  uint128 prize);
  
  constructor() public {
    // Check we called by the owner
    require(msg.sender == owner_, DiceErrors.NOT_OWNER);
    require(tvm.pubkey() == 0, DiceErrors.NON_ZERO_PUBLIC_KEY);
    
    // Contract balance at least 1.5 evers
    require(address(this).balance >= 1.5 ever, DiceErrors.NOT_ENOUGH_BALANCE);
    
    // Always reserve 1 ever on the contract balance
    tvm.rawReserve(1 ever, 0);
    
    // Call responsible function deployWallet of TokenRoot.tsol contract.
    // And set callback to our onWalletDeployed function
    ITokenRoot(tokenRoot_).deployWallet{
      value: 0,
      bounce: false,
      flag: 128,
      callback: onWalletDeployed
    }(address(this), 0.15 ever);
  }
  
  modifier onlyOwner() {
    require(msg.sender == owner_, DiceErrors.NOT_OWNER);
    _;
  }
  
  modifier onlyOurWallet() {
    require(tokenWallet_.value != 0 && msg.sender == tokenWallet_, DiceErrors.INVALID_TOKEN_WALLET);
    _;
  }
  
  function onWalletDeployed(
    address tokenWallet
  ) public {
    // There we got callback from ITokenRoot(tokenRoot_).deployWallet
    // How it is really work:
    // In constructor we called responsible function ITokenRoot(tokenRoot_).deployWallet
    // responsible keyword just adds a hidden function param "answerID"
    // answerID - just ID of function contract must call in the answer
    // So return { value: 0, flag: TokenMsgFlag.ALL_NOT_RESERVED, bounce: false } tokenWallet
    // in the deployWallet function will be compiled to something like
    // msg.sender.call{value: 0, flag: TokenMsgFlag.ALL_NOT_RESERVED, bounce: false, function:answerId}(tokenWallet)
    
    // There is no built-in check to make sure this function
    // is truly being called in answer to your call.
    // So we need to implement security checks manually
    require(msg.sender == tokenRoot_, DiceErrors.INVALID_TOKEN_ROOT);
    tokenWallet_ = tokenWallet;
    
    // Fun fact, when we get an answer here, that does not mean
    // that the wallet is deployed. This means that the Root
    // contract created an outgoing deploy message.
    // We can receive this message before the wallet is deployed
    // (the message is en route).
    // In principle, the LT (see additional information) guarantees us,
    // that if we want to call a wallet method from here,
    // our message will not arrive earlier than the wallet is deployed.
  }
  
  function onAcceptTokensMint(
    address tokenRoot,
    uint128 amount,
    address remainingGasTo,
    TvmCell payload
  ) override external onlyOurWallet {
    tvm.rawReserve(1 ever, 0);
    
    // TokenRoot owner mint some tokens to our address
    // And notified us about it, so just increase our local balance
    // to minted amount
    balance_ += amount;
    
    // Send the left gas back
    if (remainingGasTo.value != 0) {
      remainingGasTo.transfer(0, false, TokenMsgFlag.ALL_NOT_RESERVED);
    }
  }
  
  function onAcceptTokensTransfer(
    address tokenRoot,
    uint128 amount,
    address sender,
    address senderWallet,
    address remainingGasTo,
    TvmCell payload
  ) override external onlyOurWallet {
    tvm.rawReserve(1 ever, 0);
    balance_ += amount;
    
    // To roll a dice player must transfer some tokens to
    // our wallet, and set notify = true, payload = TVMCell(encode(uint8 bet_dice_value))
    
    // After successful transfer our wallet will call this callback
    // and we run game logic.
    // Look at test.js to figure out how to transfer some tokens with
    // attached data. How to make frontend we will cover in next chapter.
    
    // We got some new tokens on the our wallet.
    // To slice to decode data in payload
    
    TvmSlice payloadSlice = payload.toSlice();
    TvmCell empty;
    
    // If we have uint8 _bet_dice_value
    if (payloadSlice.bits() >= 8) {
      (uint8 _bet_dice_value) = payloadSlice.decode(uint8);
      
        if (balance_ >= amount * 6) {
        // We can play
        // Do not use this random in serious production.
        rnd.shuffle();
        uint8 dice_result_value = rnd.next(6);  // 0..5
        if (_bet_dice_value == dice_result_value) {
          // Send an external message from the contract to no where. It is a log message to easily catch all wins off-chain
          emit Game(sender, _bet_dice_value, dice_result_value, amount * 6);
          balance_ -= amount * 6;
          ITokenWallet(tokenWallet_).transfer{value: 0, bounce: true, flag: TokenMsgFlag.ALL_NOT_RESERVED}(
            amount * 6,
            sender,
            0.1 ever,
            remainingGasTo,
            false,
            empty
          );
        } else {
          emit Game(sender, _bet_dice_value, dice_result_value, 0);
          if (remainingGasTo.value != 0) {
            // Return the change
            remainingGasTo.transfer(0, false, TokenMsgFlag.ALL_NOT_RESERVED);
          }
        }
      } else {
        // cancel
        balance_ -= amount;
        ITokenWallet(tokenWallet_).transfer{value: 0, bounce: true, flag: TokenMsgFlag.ALL_NOT_RESERVED}(
          amount,
          sender,
          0.1 ever,
          remainingGasTo,
          false,
          empty
        );
      }
    } else if (remainingGasTo.value != 0) {
      // Just fulfilment
      remainingGasTo.transfer(0, false, TokenMsgFlag.ALL_NOT_RESERVED);
    }
  }
    
  function withdraw(address to, uint128 amount) external onlyOwner {
    tvm.rawReserve(1 ever, 0);
    // Allow to withdraw amount > balance
    // Because someone can send tokens without a notification.
    if (amount > balance_) {
      balance_ = 0;
    } else {
      balance_ -= amount;
    }
    
    TvmCell empty;
    
    ITokenWallet(tokenWallet_).transfer{value: 0, bounce: true, flag: TokenMsgFlag.ALL_NOT_RESERVED}(
      amount,
      to,
      0.1 ever,
      msg.sender,
      false,
      empty
    );
  }

  function maxBet() public view returns (uint128) {
    // view method to call off-chain to get max bet
    return balance_ / 5;
  }
}
```

We have just studied tip-3 token and the simplest application interacting with it. Although our token fully corresponds to production versions in terms of interfaces, it is still too simplified and does not contain all necessary methods. In production, you need to use broxus implementation of tip3.


There are many nuances taken into account there, and all necessary methods are present. For example, the burn method, similar to mint, allows you to burn tokens and send data about it to the owner contract TokenRoot. There is also an upgradable version of tip-3, which is more recommended for use.


