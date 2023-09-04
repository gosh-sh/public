# Wallets

In TVM, a transaction chain can be started by any smart contract that accepts external messages. In some applications it makes sense to write public keys directly into the contract, for example if you're playing chess, it's reasonable to just put the public keys of the two players in the contract, and take it in turns to receive messages from the players, depending on whose turn it is. Such applications can exist because the contract must explicitly agree to pay of an incoming message (do tvm.accept()), otherwise the transaction won't start and nobody will pay for it.

But in general, it is customary for the user to start interacting with contracts through his own smart-contract wallet. This is similar to how it works in EVM networks, with a correction for asynchronous interaction.

There are now two main wallets in TVM network:

TVM Minimalist wallet for one user, the main one in the TVM Minimalist wallet browser extension. Its advantages are very small code size, and it does not require a separate transaction for deploating (it has no constructor), it will be deploated on the first outgoing transaction. We will use it in the inpage-provider examples.
SetcodeMultisig(multisig2) is a formally verified multisig. Most often it is used as a single user wallet, that is, it has one custodian. We will use it in the eversdk examples.

For an example of interacting with contracts through the wallet, let's write a simple dice contract. At the same time, we will get to know more about the logic of sending messages in tvm.

Before reading the contract, We recommend reading about the work of transaction executor, to better understand the logic of TVM. At the most basic level it should be understood that there are 5 phases of transaction execution, the two main ones are compute and action. In the compute phase the contract code is executed; when you create any messages, for example to send money to another contract, you simply put the intentions to create messages in a special c5 register. It isn't checked that you have enough money to send all the messages. After successfully completing the compute phase, the action phase begins, where these messages are created. If an error occurs in the action phase, such as not having enough money to pay for a message, the account state will be rolled back to the beginning of the transaction and all messages willn't be sent (if the COMMIT(tvm.commit()) the instruction wasn't called after register was fulled with messages).

Okay, let's write our dice game contract:

```
pragma ton-solidity >= 0.64.0;
// We do not set AbiHeader pubkey; because our contract will not accept external messages at all.

library Errors {
    uint16 constant NON_ZERO_PUBLIC_KEY                             = 1000;
    uint16 constant ZERO_OWNER                                      = 1001;
    uint16 constant NOT_OWNER                                       = 1002;
    uint16 constant BET_VALUE_TOO_SMALL                             = 1003;
    uint16 constant INSUFFICIENT_BALANCE                            = 1004;
}

contract Dice {
  address static owner;

  event Game(address player, uint8 bet, uint8 result,  uint128 prize);

  constructor() public {
    // Contract pub key is NOT set
    // we will not accept any external messages 
    require(tvm.pubkey() == 0, Errors.NON_ZERO_PUBLIC_KEY);

    // Owner must be set
    require(owner.value != 0, Errors.ZERO_OWNER);

    // Constructor must be called by the owner. This check is not necessary 
    // in this contract because our constructor has no params.
    // As you remember address of the contract is 
    // hash(code + tvm.pubkey + static variables), so if you
    // have address + static variables you can proof which
    // one was set on deploy time. But constructor can be called
    // by anyone and there you often need to check is constructor
    // caller authorized contract(or by pubkey).
    // You will realize more about it in the next chapter "distributed programming"
    require(msg.sender == owner, Errors.NOT_OWNER);
  }

  modifier checkOwner {
      require(msg.sender == owner, Errors.NOT_OWNER);
      _;
  }

  // view method to call off-chain to get max bet
  function maxBet() public view returns (uint128) {
      if (address(this).balance < 0.5 ever * 6)
          return 0;
      return address(this).balance / 6;
  }

  function roll(uint8 _bet_dice_value) external {
    // check incoming message has at least 0.5 EVERs.
    require(msg.value >= 0.5 ever, Errors.BET_VALUE_TOO_SMALL);
    // check that our contract has enough balance to payout. 
    // address(this).balance already includes msg.value.
    require(address(this).balance >= msg.value * 6, Errors.INSUFFICIENT_BALANCE);

    // Shuffle rnd. This is toy random and theoretically can be manipulated by 
    // collator. Do not use it in serious production.
    rnd.shuffle();
  
    // 0..5
    uint8 dice_result_ = rnd.next(6);  
    
    if (_bet_dice_value == dice_result_) {
      // tvm.rawReserve - it is like to send amount of EVERs to your self
      // there we first send to our self (address(this).balance - msg.value * 6)
      tvm.rawReserve(address(this).balance - msg.value * 6, 2);

      // Send an external message from the contract to no where.
      // It is a log message to easily catch all games off-chain.
      emit Game(msg.sender, _bet_dice_value, dice_result_, msg.value * 6);

      // Then we send all LEFT amount of value to the player. Read below why so.
      msg.sender.transfer({value: 0, flag: 128, bounce: false});
    } else {
      emit Game(msg.sender, _bet_dice_value, dice_result_, 0);
    }
  }
  
  function cashOut(address to, uint128 value) external checkOwner {
    require(to.value != 0);
    to.transfer({
        value: value,
        flag: 0,
        bounce: true
    });
  }
}
```


In Ethereum we have two different entities, a smart contract balance and a user account balance. User starts the transaction and pays the gas fees from its own balance. But since we have an asynchronous system with multiple threads, we can't charge for gas from the account that started the transaction chain, because it might be in another thread. Instead, we calculate how much maximum gas(and other fees, like storage, and forward) it will cost to execute the entire chain of transactions and attach that amount to the initial message, and in the logic of the application we just pass the value on further down the chain. This is possible because the price of gas doesn't float with demand, instead we add more threads as increase in demand.

Before starting the compute phase, the storage fee for all time since the last transaction is deducted from the account balance, and then the balance of the incoming internal message is added to the contract balance. The contract balance (already topped up) is available at address(this).balance and the incoming value at msg.value. If the contract does not call tvm.accept() at runtime, execution may abort transaction if the gas charge exceeds msg.value. But it is important to understand that tvm.accept() is only needed to accept an external message (from the outside world) and not the internal message of another contract.

Another subtle point is that the value in the incoming internal message should only be enough to pay for gas, and you can send any amount of EVER available in the contract balance. We will discuss this in more detail in the article "Carefully working with value" in this chapter.

So let's look again at our contract. We have an incoming internal message with more than 0.5 ever, and if the player wins we simply reserve on the contract address(this).balance - msg.value * 6 and send back to the player whatever remains. That is, the player will get msg.value * 6 subtract the payment of all gas.

Very nice logic to work exactly with asynchronous messages. We don't know exactly how much gas a transaction takes, but we know how much we want to leave Ever on the contract, and we pass the rest on to the chain.

Now let's look at how to work with such contracts on the user side and on the owner side. The full code of this example is available at inpage-provider and eversdk.

## inpage-provider:

```
inpage-providereversdk
const { Address, ProviderRpcClient } = require('everscale-inpage-provider');
const { EverscaleStandaloneClient, EverWalletAccount, SimpleAccountsStorage } = require('everscale-standalone-client/nodejs');
const { SimpleKeystore } = require("everscale-standalone-client/client/keystore");
const { getGiverKeypair, getTokensFromGiver } = require("./giver");
const BigNumber = require('bignumber.js');

const { DiceContract } = require('./artifacts/DiceContract');

const keyStore = new SimpleKeystore();
const accountStorage = new SimpleAccountsStorage();

const ever = new ProviderRpcClient({
  fallback: () =>
    EverscaleStandaloneClient.create({
      connection: {
        id: 1, // connection id
        type: 'graphql',
        group: "localnet",
        data: {
          endpoints: ['127.0.0.1'],
          latencyDetectionInterval: 1000,
          local: true,
        },
      },
      keystore: keyStore,
      accountsStorage: accountStorage
    }),
});

async function main() {

  keyStore.addKeyPair(getGiverKeypair());

  // Player keypair
  const playerKeys = SimpleKeystore.generateKeyPair();
  keyStore.addKeyPair(playerKeys);

  // Casino owner keypair
  const diceOwnerKeys = SimpleKeystore.generateKeyPair();
  keyStore.addKeyPair(diceOwnerKeys);

  // in-page-provider initially was designed as web3-like
  // interface that one injected into web page from browser
  // extension. So they have accounts(wallets) as first class
  // citizen. In case we're using everscale-standalone-client
  // we can manage accounts and keys on our side.

  // So in this example in additional for keyStore
  // we are added simple accountStorage. So we can add
  // 'wallet' smart contract as account and use
  // them to start any transaction.

  // EverWallet support several types of wallet,
  // most popular - EverWallet and Multisig2.
  // We will use ever wallet. It is very simple
  // smart contract owned by a pubkey.

  // It can be easily defined from pubkey.
  // Address of the EverWallet smart-contract depend
  // only on three params:
  // 1. code
  // 2. pubkey
  // 3. nonce (by default undefinied, used to deploy several wallet from one pubkey)

  // So we can define wallet just by pubkey, code is already
  // hardcoded in the everscale-standalone-client.
  const diceOwnerWallet = await EverWalletAccount.fromPubkey({publicKey: diceOwnerKeys.publicKey, workchain: 0});

  // Send some tokens to our wallet
  await getTokensFromGiver(ever, diceOwnerWallet.address, 10_000_000_000);

  // Then just add our account to the account storage.
  accountStorage.addAccount(diceOwnerWallet);

  console.log('dice owner address is', diceOwnerWallet.address.toString());
  
  // EverWallet no has a constructor and doesn't require
  // separate action for deploy.
  // in-page-provider will track is the wallet
  // deployed and if it is not deployed provider will add
  // the wallet's StateInit in the first external message.

  // Now we will deploy DiceContract by internal message from our
  // wallet. At first, we need to calculate dice contract address
  // and stateInit
  const {
    address: diceExpectedAddress,
    stateInit: diceExpectedStateInit
  } = await ever.getStateInit(DiceContract.abi, {
    // Dice code + empty data
    tvc: DiceContract.tvc,
    workchain: 0,
    // we did not set pubkey, because our dice contract
    // is internal owned (by internal messsage)
    initParams: {
      // static params only owner
      owner_: diceOwnerWallet.address
    }
  });

  const diceContract = new ever.Contract(DiceContract.abi, diceExpectedAddress);

  // By using send({ from, amount }) we can send an internal message
  // from the wallet account.
  let tx = await extractError(diceContract.methods
    .constructor({})
    .send({
      from: diceOwnerWallet.address,
      amount: '7000000000', // 7 ever
      stateInit: diceExpectedStateInit
    }));

  // This is just a sugar, what really happened in the
  // lines above (pseudocode):
  // Sdk encoded an internal message body which one
  // is calling constructor with no arguments
  // This is the same as
  // let body = await (new diceContract.methods.constructor({}).encodeInternal();
  // Then by specifying - from: diceOwnerWallet.address
  // we tell provider to send this body
  // from the connected wallet account by internal
  // and attach to this internal amount Ever's from the
  // wallet account.
  
  // Also we specified stateInit, and sdk will also add
  // the stateInit to the internal message sended from the wallet account.

  // EverWallet is written on low level language, but on solidity
  // they probably will have function like this (pseudocode):

  //   function submitTransaction(
  //     address dest,
  //     uint128 value,
  //     bool bounce,
  //     TvmCell payload,
  //     optional(TvmCell) stateInit
  // ) external {
  //       require(msg.pubkey() == tvm.pubkey(), 100);
  //       dest.transfer({
  //         value: value,
  //         bounce: bounce,
  //         flag: 3,
  //         body: payload,
  //         stateInit: stateInit
  //     });
  //   }

  // So the all sugar in send: from - is just encoding target
  // contract method call into internal message and send
  // from the connected wallet. Easy.

  // One more important thing there, .send({from: address})
  // return the first transaction in the transaction chain.

  // So you have not any information is the internal message
  // you sent from the wallet successful or not. Or ever
  // you can not be sure is it delivered already or still
  // on the way. Also, you can have a really long transaction chain.

  // So to wait all transaction in the transaction tree you
  // need to subscribe to the transaction tree and wait until
  // all messages will be delivered.

  let constructorCallSuccess = false;
  const subscriber = new ever.Subscriber();
  await subscriber.trace(tx).tap(tx_in_tree => {
    if (tx_in_tree.account.equals(diceContract.address) && tx_in_tree.aborted === false)
      constructorCallSuccess = true;
  }).finished();

  if (!constructorCallSuccess) {
    throw new Error(`Successful constructor transaction on ${diceContract.address.toString()} not found`);
  }

  console.log('dice contract deployed at', diceContract.address.toString());

  // Okay, now we have dice contract deployed from diceOwnerWallet
  // Let's deploy our player Wallet and try to play
  const playerWallet = await EverWalletAccount.fromPubkey({publicKey: playerKeys.publicKey, workchain: 0});
  await getTokensFromGiver(ever, playerWallet.address, 30_000_000_000);
  accountStorage.addAccount(playerWallet);

  console.log('player wallet address is', playerWallet.address.toString(), '\n');
  // Let's play!
  for (let i = 0; i < 20; i++) {
    console.log('Try to play!');

    let maxBet = (await diceContract.methods.maxBet({}).call()).value0;
    if (new BigNumber(maxBet).lt(1_000_000_000)) {
      throw new Error('Max bet is less then 1 ever');
    }

    // Call roll method of the diceContract by internal message from
    // our player's wallet
    let tx = await diceContract.methods.roll({_bet_dice_value: 5})
      .send({
        from: playerWallet.address,
        amount: '1000000000' // 1 ever
      });

    // Looking for roll transaction
    const subscriber = new ever.Subscriber();
    let play_tx = await subscriber.trace(tx).filter(tx_in_tree => {
      return tx_in_tree.account._address === diceContract.address.toString();
    }).first();

    if (!play_tx) {
      throw new Error('Play transaction is not found!');
    }

    // Looking for the event Game(address player, uint8 bet, uint8 result,  uint128 prize);
    let decoded_events = await diceContract.decodeTransactionEvents({
      transaction: play_tx,
    });

    if (decoded_events[0].data.bet === decoded_events[0].data.result) {
      console.log('We won', new BigNumber(decoded_events[0].data.prize).shiftedBy(-9).toFixed(1), 'Evers\n');
      break;
    } else {
      console.log('We lose\n');
    }
    await sleep(1);
  }

  console.log("Test successful");
}

(async () => {
  try {
    console.log("Hello localhost EVER!");
    await main();
    process.exit(0);
  } catch (error) {
    console.error(error);
  }
})();

async function extractError(transactionPromise) {
  return transactionPromise.then(res => {
    if (res.transaction?.aborted || res.aborted) {
      throw new Error(`Transaction aborted with code ${res.transaction?.exitCode || res.exitCode}`)
    }
    return res;
  });
}

function sleep(seconds) {
  return new Promise(resolve => setTimeout(resolve, seconds * 1000));
}
```
## EverSDK:
```
const { Account } = require("@eversdk/appkit");
const {
  TonClient,
  signerKeys,
  signerNone,
} = require("@eversdk/core");
const { libNode } = require("@eversdk/lib-node");
const BigNumber = require('bignumber.js');

TonClient.useBinaryLibrary(libNode);

const { DiceContract } = require("./artifacts/DiceContract");
const { SetcodeMultisigContract } = require("./artifacts/SetcodeMultisigContract");

// We create a client connection to the local node
const client = new TonClient({
  network: {
    // Local EVER OS SE instance URL here
    endpoints: [ "http://localhost" ]
  }
});

async function main(client) {
  try {
    // Generate random keys pairs
    const players_keys = await TonClient.default.crypto.generate_random_sign_keys();
    const casino_owner_keys = await TonClient.default.crypto.generate_random_sign_keys();

    // Create an instance of multisig for the casino owner
    let casinoOwnerMultisig = new Account(SetcodeMultisigContract, {
      signer: signerKeys(casino_owner_keys),
      client,
      initData: {},
    });

    const casinoOwnerMultisigAddress = await casinoOwnerMultisig.getAddress();

    // Deploy msig with 1 owner for casino owner
    await casinoOwnerMultisig.deploy({
      useGiver: true,
      initInput: {
        //constructor values
        owners: [`0x${casino_owner_keys.public}`],
        reqConfirms: 1,
        lifetime: 3600
      }})

    console.log(`Casino owner multisig is deployed : ${casinoOwnerMultisigAddress}`);

    // Our dice conrract instance
    let diceContract = new Account(DiceContract,{
      signer: signerNone(),
      client,
      initData: {
        owner_: casinoOwnerMultisigAddress
      },
    });

    // To deploy the contract from another contract we need to calculate initial data (static variables + pubkey)
    let initData = (await client.abi.encode_initial_data({
      abi: diceContract.abi,
      initial_data: {
        owner_: casinoOwnerMultisigAddress
      },
      initial_pubkey: `0x0000000000000000000000000000000000000000000000000000000000000000` // if zero pubkey must be such string
    })).data

    // We take code + initial data and make StateInit.
    // We will send stateInit along with the message to call the constructor.
    // Validator will check is hash(stateInit) === address and will Initialize our contract
    // (will add code + initial data to account )
    const diceContractStateInit = (await client.boc.encode_tvc({
      code: DiceContract.code,
      data: initData
    })).tvc;

    // Secondly we encode a message which one will call the constructor in the same transaction
    // Technically contract can be deployed without calling a constructor or any other function and the account will have
    // status - active. Solidity has a special hidden variable "_constructorFlag" and always check it before any call.
    // So your contract can not be called before the constructor will be called successfully.

    // We encode a message with params is_internal: true/signer: signerNone because this is an internal message
    // we would like to send. Then we will put the encoded internal message as an argument "payload" into the external
    // message we will send to our multisig

    const diceDeployMessage = (await client.abi.encode_message_body({
      abi: diceContract.abi,
      call_set: {
        function_name: "constructor",
        input: {},
      },
      is_internal: true,
      signer: signerNone(),
    })).body

    const diceContractAddress = await diceContract.getAddress();

    // We call submitTransaction method of multisig and pass diceDeployMessage and diceContractStateInit to deploy
    // our dice contract.
    // There we are creating an external message with arguments and sending it to the contract. For signing the message
    // by default will be used "signer" which one we specified when created and "casinoOwnerMultisig" instance - casino_owner_keys
    await casinoOwnerMultisig.run('submitTransaction', {
      dest: diceContractAddress,
      value: 9_000_000_000, // 9 evers
      bounce: true,
      allBalance: false,
      payload: diceDeployMessage,
      stateInit: diceContractStateInit
    })

    // SIMPLIFIED how submitTransaction of msig2 looks like:
    // But if your msig2 has more than 1 custodian they will be putted into queue
    // and will wait for confirmation from other custodians
    //   function submitTransaction(
    //     address dest,
    //     uint128 value,
    //     bool bounce,
    //     bool allBalance,
    //     TvmCell payload,
    //     TvmCell stateInit
    // ) {
    //       require(msg.pubkey() == m_ownerKey, 100);
    //       uint8 flags = FLAG_IGNORE_ERRORS(1) | FLAG_PAY_FWD_FEE_FROM_BALANCE(2);
    //       if (allBalance) {
    //           flags = FLAG_IGNORE_ERRORS(1) | FLAG_SEND_ALL_REMAINING(128);
    //           value = 0;
    //       }
    //       dest.transfer({
    //         value: value,
    //         bounce: bounce,
    //         flag: sendFlags,
    //         body: payload,
    //         stateInit: stateInit
    //     });
    //   }


    console.log('Dice contract deployed, max bet is', nanoEversToEvers((await diceContract.runLocal('maxBet', {}, {})).decoded.output.value0));

    // In the same way we create multisig for the player
    let playerMultisig = new Account(SetcodeMultisigContract, {
      signer: signerKeys(players_keys),
      client,
      initData: {},
    });

    const playerMultisigAddress = await casinoOwnerMultisig.getAddress();

    await playerMultisig.deploy({
      useGiver: true,
      initInput: {
        //constructor values
        owners: [`0x${players_keys.public}`],
        reqConfirms: 1,
        lifetime: 3600
      }})


    // We encode a message with params is_internal: true/signer: signerNone because this is an internal message
    // we would like to send. Then we will put the encoded internal message as an argument "payload" into the external
    // message we will send to our multisig
    const makeBetMessage = (await client.abi.encode_message_body({
      abi: diceContract.abi,
      call_set: {
        function_name: "roll",
        input: {
          _bet_dice_value: 0
        },
      },
      is_internal: true,
      signer: signerNone(),
    })).body;

    for (let i = 0; i < 100; i++) {
      // We use bignumber because it is good practise, maximum safe int in js is only 9_000_000 evers
      // https://stackoverflow.com/questions/307179/what-is-javascripts-highest-integer-value-that-a-number-can-go-to-without-losin

      let maxBet = new BigNumber((await diceContract.runLocal('maxBet', {}, {})).decoded.output.value0);
      if (maxBet.lt(0.6 * 1_000_000_000)) {
        console.log('Dice contract has not enough balance to play');
        break;
      }
      let ourBalance = new BigNumber(await playerMultisig.getBalance());

      // 0.7 because 0.6 + gas;
      if (ourBalance.lt(0.7 * 1_000_000_000)) {
        console.log('We have not enough Evers to play');
        break;
      }
      console.log('\nTry to roll a dice...');

      let result = await playerMultisig.run('sendTransaction', {
        dest: diceContractAddress,
        value: 600_000_000, // 0.6 evers
        bounce: true, // Send funds back on any error in desctination account
        flags: 1, // Pay delivery fee from the multisig balance, not from the value.
        payload: makeBetMessage
      });

      // Load list of all transactions and messages
      let transaction_tree = await client.net.query_transaction_tree({
        in_msg: result.transaction.in_msg,
        abi_registry: [playerMultisig.abi, diceContract.abi]});

      // Look for game event log
      let gameLogMessage = transaction_tree.messages.find(m => m.src === diceContractAddress && m.decoded_body && m.decoded_body.name === 'Game');
      if (!gameLogMessage)
        throw new Error('Game not found');

      if (gameLogMessage.decoded_body.value.prize !== '0') {
        console.log('We won', nanoEversToEvers(gameLogMessage.decoded_body.value.prize), 'Evers');
        break;
      } else {
        console.log('Lose!')
      }
      await sleep(1);
    }

    console.log("Test successful");
  } catch (e) {
    console.error(e);
  }
}

(async () => {
  try {
    console.log("Hello localhost EVER!");
    await main(client);
    process.exit(0);
  } catch (error) {
    if (error.code === 504) {
      console.error(`Network is inaccessible. You have to start EVER OS SE using \`everdev se start\`.\n If you run SE on another port or ip, replace http://localhost endpoint with http://localhost:port or http://ip:port in index.js file.`);
    } else {
      console.error(error);
    }
  }
  client.close();
})();

// To human readable amount
function nanoEversToEvers(nano) {
  return (nano / 1_000_000_000).toString();
}

function sleep(seconds) {
  return new Promise(resolve => setTimeout(resolve, seconds * 1000));
}
```

We have been introduced to wallets and internally owned contracts that are managed from a different address than external messages. As a reminder, the complete example is available for inpage-provider and eversdk.

In the next article we will learn what distributed programming is and write a simple implementation of the tip-3 token(ERC20 equivalent in TVM blockchains, which takes into asynchronous model).
