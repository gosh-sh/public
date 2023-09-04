# Replay protection

In the TVM blockchain architecture, smart contracts can accept arbitrary external messages. These messages can even be unsigned by any private key. An external message is allocated 10k credit gas, and the transaction execution is launched. The contract must conduct all necessary checks on the credit gas and either throw an error or agree to pay for the transaction from the contract balance by calling tvm.accept(). Typically, the smart contract checks that the message is signed by the private key of a public key recorded in the contract.

```
function set(uint _value) external checkOwnerAndAccept {
  require(msg.pubkey() == tvm.pubkey(), 102);
  tvm.accept();
  variable = _value;
}
```

Since it is considered in the TVM blockchain philosophy not to produce an infinite tail of data, validators are not required to store all messages that have been accepted by the contract, and the same external message can be included as many times as the contract agrees to pay for it. So every contract must implement its own protection against repeated external messages.


TON solidity has built-in protection against replay attacks. It works at the SDK + Abi level.


In addition to the hidden variable tvm.pubkey(), there is a hidden static variable uint64 timestamp in the contract. This is the time of the last accepted external message. In the abi file of each contract there is a "header": [] field, and it lists which additional values the SDK should add to the external message. By default, it is "header": ["time"], which indicates that the time of creation of the external message should be added to the external message.


So the default ABI includes the time of creation in the external message, and the compiler generates a check that, before choosing a function, verifies that the time of the last accepted message is less than the new one, updates the time. This is handled by pragma AbiHeader time, which is added by default (if you don't have a special function afterSignatureCheck declared, which we will discuss below)


This protection is quite primitive and will not work well if you are sending many external messages in parallel, so it is possible to make your own replay protection by declaring a special function afterSignatureCheck.


But in addition to the fact that the check is quite simple, it has one big drawback that needs to be understood. If there are errors in your contract that occur after tvm.accept(), such as some additional require(), then the transaction will fail, and the entire contract state will be rolled back to the start. This means that the timestamp will not be updated to the new one, and the validator will be able to include this external message as many times as he wants, as long as there are coins on the contract balance, or the message does not expire.


Most often, such an error is encountered in the action phase. For example, if you have a contract-wallet with a function like this:

```
function send(address _to, uint128 _value) external checkOwnerAndAccept {
  _to.transfer(_value, false, 0);
}
```
If the user tries to send more ever than he has on his account, or for example, tries to send everything exactly, but part will be spent on gas, and as a result, the action phase will fail with an error that there is not enough coins to send, then the transaction will be considered aborted, and the account state will be reset to the moment the transaction began - the validator will again be able to include this message, as it is valid again.

The moral here is quite simple - always do all checks before tvm.accept(). And when sending value, where possible, use flag + 2, this means that if any errors occur when sending this message, it should just be ignored. Or check exactly that you have enough coins to send.

There is also:

pragma AbiHeader expire; which adds a header expire, instructing the ABI to add the time after which this message is no longer considered valid, and the compiler adds a check based on the block time.

pragma AbiHeader pubkey; instructing the ABI/compiler that all external messages must be signed with a private key, and the compiler adds a signature check. Moreover, the signature does not necessarily have to be the private key of the same public key set as tvm.pubkey(). The ABI can sign with any pair of keys and attach the public key to the message for signature verification. That's why we check tvm.pubkey() == msg.pubkey() when receiving external messages. This pragma should always be added if you are going to receive external messages.

