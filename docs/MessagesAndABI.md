# Messages and abi

In the previous example, we looked at how to deploy a smart contract using an external message. In this article, we will take a look at ABI and see how SDK uses it to create external or internal messages.

A contract that receive internal messages
Let's rewrite the contract from the previous example to a contract that accepts messages from another contract (i.e. internal messages):
```
pragma ton-solidity >= 0.64.0;
pragma AbiHeader time;
pragma AbiHeader pubkey;
pragma AbiHeader expire;

contract SimpleStorage {
  // Contract that one can set variable
  address static private owner;
  uint private variable = 0;

  event VariableChanged(uint new_value);

  constructor() public {
    tvm.accept();
  }
  
  modifier checkOwner {
    // check the message sent by our owner
    require(msg.sender == owner, 100);
    _;
  }
  
  function get() public view returns(uint) {
    // View function to call offchain
    return variable;
  }


  function getInternal() public responsible view returns(uint) {
    // Function marked as responsible
    // function to get the value from another contract
    return {value: 0, flag: 64, bounce: false} variable;
  }

  
  function set(uint _value) external checkOwner {
    variable = _value;
    // log event to easily parse offchain
    emit VariableChanged(_value);
  }
}
```
In this contract, we set a static variable owner, and only allow our owner to call the set function.

## Structure of abi

Abi is generated by the ever solidity compiler, to help sdk pack messages. Let's look at the abi file of our contract:
```
{
  // Major version of ABI standart
  "ABI version": 2,
    // Full version of ABI
    // Can be – 2.0, 2.1, 2.2, 2.3
  version: "2.3",
  // Headers, specifying SDK which additional fields to attach to external message
  // Defined in the contract code, there are:
  // pragma AbiHeader time;
  // pragma AbiHeader pubkey;
  // pragma AbiHeader expire;
  header: [
    "time", "pubkey", "expire"
  ],
  // Description of callable function signatures
  // both internal and external messages
  functions: [
    {
      "name": "constructor",
      "inputs": [],
      "outputs": []
    },
    {
      "name": "get",
      "inputs": [],
      "outputs": [{"name":"value0","type":"uint256"}]
    },
    {
      "name": "getInternal",
      "inputs": [
        {"name":"answerId","type":"uint32"}
      ],
      "outputs": [
        {"name":"value0","type":"uint256"}
      ]
    },
    {
      "name": "set",
      "inputs": [{"name":"_value","type":"uint256"}],
      "outputs": []
    }
  ],
  // A description of the events that a contract can create
  events: [
    {
      "name": "VariableChanged",
      "inputs": [{"name":"new_value","type":"uint256"}],
      "outputs": []
    }
  ],
  // A list of static variables that must be specified to deploy the contract
  data: [
    {"key":1,"name":"owner","type":"address"}
    // There are also three hidden variables that SDK will set by itself
    // _pubkey, _timestamp, _constructorFlag
  ],
  // a list of all variables, so that you can
  // download the contract state and decode it
  fields: [
    {"name":"_pubkey","type":"uint256"}, // tvm.pubkey()
    {"name":"_timestamp","type":"uint64"}, // set by SDK
    {"name":"_constructorFlag","type":"bool"}, // set by SDK
    {"name":"owner","type":"address"},
    {"name":"variable","type":"uint256"}
  ]
};
```

You can read the full description of ABI and their evolution here, we will describe only the basic things to understand about messages and ABI, and it will help you understand how wallet smart contracts work.


## Packaging of message body

Basically, ABI describes how we pack the data into a TOC (Tree Of Cells, that described in the chapter about tvm). The external/internal message has the same main part, it is the body for the function call. Let's look at an example function from abi:
```
{
  "name": "set",
  "inputs": [{"name":"_value","type":"uint256"}],
  "outputs": []
},

When we asks sdk to call this function, it first needs to package the body. Body is simple:

// solidity
TvmCell body = abi.encode(tvm.functionId(contract.set), _value);
```
That is, after accepting a message with body, the contract simply decodes the functionID to be called and, if there is one, passes control to it. On the input, the arguments, their number and other features are checked. functionID is a first 32 bits of message body.


These two functions on Solidity do the same thing:

```
contract StorageOwner {
  function setTo(address to, uint variable_value) public {
    SimpleStorage(to).set{value: 0.1 ever, bounce: false, flag: 0}(variable_value);
  }
  
  function setToRaw(address to, uint variable_value) public {
      TvmCell body = abi.encode(tvm.functionId(SimpleStorage.set), variable_value);
  
      // Just send a raw message
      // https://github.com/tonlabs/TON-Solidity-Compiler/blob/master/API.md#addresstransfer
      // <address>.transfer(uint128 value, bool bounce, uint16 flag, TvmCell body);
      to.transfer(0.1 ever, false, 0, body);
  }
}
```
## Message types
There are two different entry points for external and internal messages in the bytecode of a smart contract.

What happens when an internal message is received:
```
Initializing msg.sender - address of sender's account (for external addr_none$00).
Initializing msg.value - value in nano-evers of incoming message (for external == 0).
```
The functionID is decoded, and control is transferred there if there is such a function.

With external messages, things are a bit more complicated, and the behavior of the contract depends on the pramga AbiHeader * specified in it. The standard ones are pragma AbiHeader pubkey;, pragma AbiHeader time; and pragma AbiHeader expire;, we will consider what happens if they are specified. In external message with such headers besides body are added:
```
timestamp - The creation time of the message, it is checked that its time is longer than the time of the previous received message, is explained in more detail in the article "replay protection" of this chapter.
expire - The time at which this message is considered valid. The default creation time + 2 minutes.
signature - Signature of all variables mentioned above + body by private key from the pubkey which one attached to the message. This can be not the same pubkey set in contract and available by tvm.pubkey(). That's why we usually check who signed the external message by calling require(msg.pubkey() == tvm.pubkey()); Also, the signature is depended on the abi version.
```
Accordingly external message passes checks described above, and further logic is the same as for internal messages. Just don't forget that contract should explicitly agree in function code to accept external message by calling tvm.accept(), otherwise it won't get into block.

You can also limit the entrypoints of an external or internal message function by specifying the modifiers externalMsg or internalMsg.

It should be understood that regular getters that are not marked as responsible do not make sense to call from other contracts, they are designed for the user to download the contract state and run the desired getter to get the value. Such a getter is run on the local TVM, using an external message. In order to start such a getter you must know not only the function interface, but also the version of the abi contract and what headers are specified there. Because local external message will go through all the same checks as onchain and message signature depends on all these parameters.

Our contract also has function getInternal() public responsible view returns(uint), a getter intended to be used with the internal message. As you can see from ABI, the keyword responsible just adds the first hidden variable, uint32 answerId, to the function arguments. This is the functionId to be called from the sender, by passing the answer to it. Let's look at an example code, these two functions do the same thing:
```
contract AnyContract {
  
  function getWithSugar(address from) public {
    RemoteContract(addr).getInternal{value: 1 ever, flag: 0, bounce: false, callback: AnyContract.onAnswer}(x);
  }
  
  function getRaw(address from) public {
    TvmCell body = abi.encode(tvm.functionId(SimpleStorage.getInternal), tvm.functionId(AnyContract.onAnswer));

    // <address>.transfer(uint128 value, bool bounce, uint16 flag, TvmCell body);
    from.transfer(1 ever, false, 0, body);
  }
  
  function onAnswer(uint value) public {
    // some logic here
  }
}
```

Of course sdk knows how to run the responsible methods of the contract locally. To do this, it emulates sending an internal message with infinite gas, and parses the response. In fact, writing responsible functions instead of normal getters is considered good practice. You create an interface for other contracts and users at once. And also, you only need to know only the interface of the function to run it, you don't need to know the abi version of the contract or the headers. This helps when you need to work with different contracts that implement the same interface.



## Wallets Smart Contracts

Let's take a look at what a smart wallet contract code might look like (simplified):
```
pragma ton-solidity >= 0.64.0;
pragma AbiHeader expire;
pragma AbiHeader pubkey;
pragma AbiHeader time;

contract Wallet {
  
  constructor() {
    tvm.accept();
  }
  
  function sendTransaction(
    address dest,
    uint128 value,
    bool bounce,
    uint8 flags,
    TvmCell body,
  ) external {
    require(msg.pubkey() == tvm.pubkey(), 100);
    tvm.accept();
    dest.transfer({
      value: value,
      bounce: bounce,
      flag: flags,
      body: body
    });
  }
}
```
When we need to call someone's contract method from our wallet, we simply ask SDK to encode a body for us to call, and send from the wallet by calling sendTransaction with an external call. The sdk will generate an external message and send it to the network. Then it will track all your wallet transactions for two minutes and return you a transaction or error message, if the message went stale before it hit the block.

If the first external message hits the block, the call chain will be finalized sooner or later, because for internal messages we have a 100% delivery guarantee and we can use SDK to subscribe to track the next transactions in the call chain.

Actually this is the main thing you need to know about message types and ABI, in the next lesson we will write a smart contract for the dice game, and we will use the real wallet contract to start the transactions from it.