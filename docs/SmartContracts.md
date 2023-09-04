# Smart contracts

# Toolchain

We develop smart contracts with solidity, but of course, TON Solidity is enhanced with additional types and functionality to adapt with TVM and blockchain actor model. The compiler has an excellent docs and examples.

It is customary to use extension for smart contract files .tsol, which means Threaded solidity, to separate smart contracts from the usual synchronous solidity, although .sol also occurs. There are community-driven plugins for jetbrains idea and visual studio code.

To get a smart contract code that is suitable for blockchain deployment, there are several steps to follow. For first, you need to compile .tsol files with TON Solidity compiler. In the output you will have two files, the .code with the TVM assembler and .abi.json where the functions/variables/headers of the contract to interact with it are described.

Next, we need to use TVM-linker, which will take the .code and .abi files, link the necessary parts with the standard library, add the entry points for the functions and create a .tvc file with bytecode of the contract. The .tvc file is a serialized tree of cells which we described in the chapter about tvm.

And the most interesting stage. We need to deploy our smart contract to Everscale blockchain network and start to interact with it. Here we have four of the most popular tools that helps us with this goal:

everscale-inpage-provider – a library for interacting with blockchain developed by broxus. Uncomplicated learning curve. The advantage is that the library can be used inside your server application, as well as with the most popular browser add-on wallet EverWallet.

locklift – development environment by broxus, usefull key/network management, tests and migrations. Created in the likeness of frameworks such as hardhat or truffle.

tonos-cli – a complete cli for interacting with the Everscale blockchain. Allows deployment and interaction with smart contracts. Suitable for Makefile and bash enthusiasts.

eversdk AppKit JS JS library that simplifies deployment and interaction with smart contracts. Simple diving curve and lots of examples. There are bindings to popular languages. As well allows you to search the blockchain stack using graphql queries, and subscribe to updates.


# Practice
In this guide, we will not cover the basics of solidity programming, but only the ES-specific things. In the first points, we will use Eversdk Appkit JS and everscale-inpage-provider for deploying and interacting with smart contracts, then we'll move on to locklift framework.


So let's write our first smart contract and think out what's going on:

```pragma ton-solidity >= 0.61.2;
// This header informs sdk which will create the
// external message has to be signed by a key.
// Also directing the compiler that it should
// only accepted signed external messages
pragma AbiHeader pubkey;

contract SimpleStorage {

  // Just random static variable to see the difference between
  // static and state variables in the deploy process.
  uint static public random_number;

  // State variable for storing value
  uint public variable = 0;

  constructor(uint _initial_value) public {
    // We check that the contract has a pubkey set.
    // tvm.pubkey() - is essentially a static variable,
    // which is set at the moment of the creation of the contract,
    // We can set any pubkey here or just leave it empty.
    require(tvm.pubkey() != 0, 101);
    
    // msg.pubkey() - public key with which the message was signed,
    // it can be  0 if the pragma AbiHeader pubkey has not been defined;
    // We check that the constructor was called by a message signed by a private key
    // from that pubkey that was set when the contract was deployed.
    require(msg.pubkey() == tvm.pubkey(), 102);
    
    // We agree to pay for the transaction calling the constructor
    // from the balance of the smart contract
    tvm.accept();

    // set variable to the passed initial value
    variable = _initial_value;
  }

  // Modifier that allows to accept some external messages
  modifier checkOwnerAndAccept {
    // Check that message was signed with contracts key.
    require(msg.pubkey() == tvm.pubkey(), 102);
    tvm.accept();
    _;
  }

  function get() public view returns(uint) {
    return variable;
  }

  // Function that adds its argument to the state variable.
  function set(uint _value) external checkOwnerAndAccept {
    variable = _value;
  }
}
```

You have already read about the external message and the life cycle of a smart contract here.


And so we wrote our first smart contract, which just stores a number, and allows you to change it for the owner of the contract. Now let's put it on the ES network and try to interact with it. To compile and link the contract we'll use everdev.

But, before it, you should install docker to setup everscale local node. Local node allow you to test smart contracts as the easiest way, without worrying about test coins.

Install docker desktop https://www.docker.com

```
// install everdev
npm i -g everdev


// Run a local node. It's important, docker must be run before.
// The local node also automatically launches the blockchain explorer at http://localhost

everdev se start


// The clone github repo with examples
git clone https://github.com/mnill/everscale-crash-course-snippets && cd everscale-crash-course-snippets/basic


// Install dependencies
npm i


// Compile and link our smart contract. everdev will automatically downloads
// the latest versions of the compiler and TVM linker.

mkdir -p artifacts && everdev sol compile -o ./artifacts contracts/SimpleStorage.tsol


// Build SimpleStorage.tvc (sources) and SimpleStorage.abi.json (ABI) into the wrapper
// for easy use in js
everdev js wrap ./artifacts/SimpleStorage.abi.json


// run the script, which will replicate our contract and interact with it
node test.js
```


Let's analyze what happens in the test.js file

## inpage-providerever:

```
inpage-providereversdk
const { Address, ProviderRpcClient } = require('everscale-inpage-provider');
const { EverscaleStandaloneClient } = require('everscale-standalone-client/nodejs');
const { SimpleStorageContract } = require("./artifacts/SimpleStorageContract");
const { SimpleKeystore } = require("everscale-standalone-client/client/keystore");
const { getGiverKeypair, getTokensFromGiver } = require("./giver");

const keyStore = new SimpleKeystore();
const ever = new ProviderRpcClient({
  fallback: () =>
    // We are setting up a fallback client because we are 
    // using a provider from nodejs. Fallback is a client
    // to the blockchain that is used if the EverWallet
    // extension is not available in the browser or if we
    // are using inpage-provider from nodejs.
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
      keystore: keyStore
    }),
});

async function main() {
  // Add giver keypair to provider keystore
  // Giver is a generic name for any smart contract
  // that is used to send Ever's for deploying
  // smart contracts by an external message.

  // The local node has a pre-deployed giver
  // with a simple interface. Add their keypair
  // to the keyStore, because provider
  // always looking into keystore to find a keypair
  // to sign an external message.
  keyStore.addKeyPair(getGiverKeypair());

  // Generate a random keypair to set as owner of
  // our SimpleStorage contract. In production, you
  // of course need to import this keypair.
  const keyPair = SimpleKeystore.generateKeyPair();
  keyStore.addKeyPair(keyPair);


  // Calculate a stateInit of our contract.

  // StateInit - it is a packed contract Code and InitialData(storage)

  // If the destination contract is not deployed, the validator
  // will check that hash(stateInit) == address(contract) and
  // initialize the contract with the code and initial data 
  // from stateInit before the transaction starts. 

  // We also got expectedAddress, because address is just hash(stateInit)
  const {address: expectedAddress, stateInit} = await ever.getStateInit(SimpleStorageContract.abi, {
    // Tvc it is code + zeroInitial data.
    // function will replace zero data to actual
    // from initParams.
    tvc: SimpleStorageContract.tvc,
    workchain: 0,
    publicKey: keyPair.publicKey,
    initParams: {
      random_number: Math.floor(Math.random() * 10000)
    }
  });


  // To deploy a smart contract with an external message, 
  // we first need to send the necessary amount of Ever 
  // to its address using the parameter bounce: false when
  // sending. Then, even if the destination account does not 
  // exist, the coins will remain at its address and the account
  // status will change from NotExist to NotInitialized

  // Local node has pre-deployed giver. For the testnet/mainnet
  // You need to setup giver by yourself.
  // Check "Setup environment for testnet/mainnet" article
  // to figure out how to do this.
  await getTokensFromGiver(ever, expectedAddress, 1_000_000_000); // 1 ever = 1_000_000_000 nanoEvers

  // Now we can send an external message to our account
  
  // Create an instance of contract
  const contract = new ever.Contract(SimpleStorageContract.abi, expectedAddress);

  // We just send an external message to our contract
  // that one will call constructor with arguments _initial_value = 1
  // And also we attach stateInit to our message,
  // because our contract must be initialized first.
  await extractError(contract.methods.constructor({_initial_value: 1}).sendExternal({
    stateInit: stateInit,
    // Provider will search for the signer for this pubkey in the keyStore
    publicKey: keyPair.publicKey,
  }))

  // Call view method 'get'.
  // To do this, sdk will download full account state and run TVM locally
  let {value0: variableValue} = await contract.methods.get({}).call({});
  console.log('Account successfully deployed at address', expectedAddress.toString(), 'variable is set to', variableValue);

  console.log('Set variable to 42')
  await extractError(contract.methods.set({_value: 42}).sendExternal({
    publicKey: keyPair.publicKey,
  }))

  let {value0: newVariableValue} = await contract.methods.get({}).call({});

  if (newVariableValue !== '42') {
    throw new Error('Variable is not 42')
  } else {
    console.log('Success, variable value is', newVariableValue);
  }

  console.log("Test successful");
}

async function extractError(transactionPromise) {
  return transactionPromise.then(res => {
    if (res.transaction.aborted) {
      throw new Error(`Transaction aborted with code ${res.transaction.exitCode}`)
    }
    return res;
  });
}

(async () => {
  try {
    console.log("Hello EVER!");
    await main();
    process.exit(0);
  } catch (error) {
    console.error(error);
  }
})();
```
## EverSDK:

```
const { Account } = require("@eversdk/appkit");
const { TonClient, signerKeys, signerNone } = require("@eversdk/core");
const { libNode } = require("@eversdk/lib-node");

TonClient.useBinaryLibrary(libNode);

const { SimpleStorageContract } = require("./artifacts/SimpleStorageContract")

// We create a client connection to the local node
const client = new TonClient({
  network: {
    // Local EVER OS SE instance URL here
    endpoints: [ "http://localhost" ]
  }
});


async function main(client) {
  try {
    // Generate random keys pair
    const keys = await TonClient.default.crypto.generate_random_sign_keys();

    // Create an instance of contract.
    // To create an instance we need to specify
    // * signer - keypair to sign an external messages we will send to this contract. keys.public is the same key is tvm.pubkey()
    // * client - a client to a blockchain
    // * initData - it is initial values for all STATIC variables.

    let simpleStorageInstance= new Account(SimpleStorageContract, {
      signer: signerKeys(keys),
      client,
      initData: {
        random_number: Math.floor(Math.random() * 100)
      },
    });

    // Contract address is always a hash(hash(contract_code) + hash(initial value of all static variables)).
    // So now when we have known contract code and all initial data we can calculate future address of the contract.
    const simpleStorageInstanceAddress = await simpleStorageInstance.getAddress();

    // Note: that is tvm.pubkey() is also a static variable. Just a hidden one.
    // So our contract address depend on CODE + random_number + keys.pubkey

    // Deploy our contract.
    // We use param useGiver: true - this is mean sdk will use pre-deployed wallet with EVERs on the local network to
    // send a necessary amount of EVERs to the contract to deploy it.
    // initInput - it is variables that will be passed to the constructor.

    // Note: the contract address is not dependent on the variables we will send to the constructor.
    await simpleStorageInstance.deploy({
      useGiver: true,
      initInput: {
        _initial_value: '0x1'
      }})
    console.log(`Simple storage deployed at : ${simpleStorageInstanceAddress}`);


    // Note: you can also create an instance of the contract not only by specifying a pubkey and static variables and just
    // by the address. Like this:
    // simpleStorageInstance = new Account(SimpleStorageContract, {
    //   signer: signerNone(),
    //   client,
    //   address: simpleStorageInstanceAddress
    // });

    // runLocal - a way to call a view methods of the contracts.
    // sdk will download current account state + code and execute tvm locally to get the answer.

    let response = await simpleStorageInstance.runLocal("variable", {});
    console.log('After deploy variable param is', response.decoded.output.variable);

    // run - send an external message to the contract. Specified signer will be used to sign the message.
    console.log('Set variable to 0xFF');
    await simpleStorageInstance.run("set", {
      _value: 0xFF
    });

    console.log('Success');

    response = await simpleStorageInstance.runLocal("variable", {});
    console.log('Now variable param is', response.decoded.output.variable);

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
```


How the account is deployed we described in The account life cycle

The process of deploying a contract from a contract is similar, except that we just send ever + stateInit + variables to call the constructor in one message.

How to configure the environment to interact with testnet/mainnet read here

There is a little confusion about definitions. Our smart contract has a function set(uint _value) external method. The modifier external in solidity has nothing to do with external\internal messages. In the case of solidity it is simply an indication that the given method can only be called by a message (any, external from the outside world or internal - a message sent from another contract) and cannot be called from another method of the same contract.

Great, we've developed and deployed our first smart contract using inpage-provider and eversdk appkit js.


