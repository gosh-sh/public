# Dangerous places of Ton-Solidity compiler

Here we will describe a few non-obvious things that can shoot yourself in the foot.

## Be careful with responsible functions

Responsible is a sugar for getting an answer from another contract. Under the hood, it's just the addition of another parameter, uint32 answerId, which is passed the id of the function to be called in the back message.

```
// this
function getOwner() public responsible returns (address) {
  return { value: 0, flag: 64, bounce: false } _owner;
}

// will be compiled to this
function getOwner(uint32 answerId) public {
  TvmCell payload = abi.encode(answerId, _owner);
  dest.transfer({
    value: 0,
    bounce: false,
    flag: 64,
    body: payload,
  });
}
```

There are two subtle points here. Firstly, always use flag: 64 or rawReserve and flag 128 to ensure that such calls do not affect the balance of the account, and secondly, never set bounce: true for a responsible function unless you have a very good reason to do so. This can be used to attack the onBounce handler.

Here is an example of a vulnerable contract (some kind of token):

```
contract VulnerableTokenContract  {
  uint128 balance;

  function get_balance() override external view responsible returns (uint128) {
    return{ flag: 64, value: 0, bounce: true} balance;
  }

  function transfer(uint128 _amount_tokens, uint256 _to_pubkey) external {
    require(tvm.pubkey() == msg.pubkey());
    require(balance > _amount_tokens);

    tvm.accept();
    address reciever = _getExpectedAddress(_to_pubkey); 
    balance -= _amount_tokens;
    VulnerableTokenContract(reciever).recieve{value: 7_500_000, flag: 0, bounce: true}(_amount_tokens, tvm.pubkey());
  }

  function recieve(uint128 _amount_tokens, uint256 _from_pubkey) {
    address expected_sender = _getExpectedAddress(_from_pubkey)
    require(msg.sender == expected_sender);
    balance += _amount_tokens;
  }

  onBounce(TvmSlice body) external {
    tvm.accept();
    uint32 functionId = body.decode(uint32);
    if (functionId == tvm.functionId(VulnerableContract.transfer)) {
      uint128 amount_tokens = body.decode(uint128);
      balance += amount_tokens;
    }
  }
}
```

The vulnerability lies in the fact that the attacker can call the get_balance function and pass an answerId that matches tvm.functionId(VulnerableTokenContract.transfer), and when it gets a response, it will throw an error. Our contract will then receive an onBounce call, but not from the same contract as itself, but from the attacker, resulting in doubling its balance. This is a rather complex thought, try to trace the attack vector through the contract. The key here is that you should always set bounce: false in responsible functions.


## Replay protection

Please read article about replay protection to figure out why you should use flag: N + 2 if your contract accepts external messages.


## Constructor and searching by the code hash

We have already talked about two hidden static variables, the first is tvm.pubkey(), and the second is _timestamp for replay protection. But there is a third and final hidden static variable - _constructorFlag.

The thing is that tvm is a fairly simple virtual machine, and the constructor in Solidity is attached to it as if by crutches. At the assembly code level, there is no such thing as a contract constructor. There are just two entry points for code - receiving internal messages and receiving external messages, and one cell with all the contract data. To make a constructor, they simply made a boolean flag that is set to 0 by default. After a successful call to the constructor, this flag is set to 1. And when trying to call any other function, it is checked that the constructor flag is set to 1.

As we remember, the contract address is hash(hash(code) + hash(data - static variables)). Let's consider an example contract.

```
Contract A {
  static address owner_;
  uint someNumber;

  constructor(uint _setValue) {
    require(msg.sender === owner_);
    someNumber = _setValue;
  }

  function getValue() public view returns(uint) {
    return someNumber;
  }

  function getOwner() public view returns(address){
    return owner_;
  }
}
```
It seems like that knowing the code of this contract, we can trust its content. For example, we can search contracts offchain by the code hash, filter them by the desired owner, and trust the values obtained from getValue(); However, knowledge of the code is not sufficient for this. It is also necessary to verify that the contract address is calculated from the correct initial data.

We can deploy a contract with any initial data. We can specify any owner_ there and make it seem like the constructor has already been called and write any someNumber (By specifying non-standard initial data). However, such a contract will have a different address.

The moral is that it is often not enough to know the hash code of a contract to trust its content. It is also necessary to verify that its address is calculated from the correct initial data.

## Credit gas


When contract receive an external message, it is allocated 10,000 credit gas. The contract must use this gas to perform all necessary checks and agree to pay for the transaction if the message is valid.


As we described in the chapter on the TVM, the entire storage of a contract is a single large TvmCell with subcells. The more data you have in storage, the more expensive it is to work with it. This means that you should avoid checks that are not constant in time. For example, it is better not to have an infinitely sized mapping, where a large number of public keys that are permitted to run transactions in this contract are recorded. Because the larger the mapping, the more expensive it is to read from it (by Log N), and at some point 10,000 credit gas may not be enough to check.


Or this contract from the compiler's example repository implements custom replay protection and uses a mapping to record hashes of messages it has received in recent times, and then clears them when they have expired and cannot be replayed again. However, this contract may simply be stopped on the deadlock if it receives too many messages per second, and the size of the mapping grows beyond a certain limit (10k of gas will not enough to perform all checks and write a message to the mapping before tvm.accept()).


Global variables are also stored in a single Tree of cell (TvmCell with references to child cells), this is the TVM register c4. To avoid having to go far through the tree every time you access variables, Solidity unpacks them all into a special temporary register, c7, which is an array (tuple). Consider this as doing abi.decode(TvmCell contract_state,(...all variables)). And then when accessing the state variable, you just access the array by index. At the end of the transaction, it serializes back from c7 to c4. So the more state variables you have declared, the more gas you spend on their serialization/deserialization at the beginning and end of the transaction.
