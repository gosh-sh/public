# Carefully working with VALUE when creating messages

In general, whenever you create outgoing messages while processing incoming internal messages, you must be very careful with the value that you attach to messages.


I remind you that, according to the transaction execution phases, all actions take place after the compute phase in the action phase. In the compute phase, you simply accumulate all of your actions, whether it be sending a message or calling tvm.rawReserve() in a special register. If the compute phase is successfully completed, an attempt is made to execute all actions. If an error occurs during the execution of an action and that action does not have a special flag indicating that it should be ignored in the case of errors, the transaction will be considered unsuccessful and all state changes that occurred in the compute phase will be undone (all actions will be also canceled).


Here is an example of a function that can spend money from a contract account:
```
function deployWallet(
  address _owner,
  uint128 _deploy_evers
) external {
  TvmCell stateInit = tvm.buildStateInit({
    contr: TokenWalletContract,
    varInit: {
      root: address(this), 
      owner: _owner,
    },
    pubkey: 0,
    code: wallet_code
  });

  new TokenWalletContract {
     stateInit: stateInit,
     value: _deploy_evers,
     wid: address(this).wid,
     flag: 0
  }();
}
```
When processing an internal message, without tvm.accept(), the gas spent by the message cannot exceed the ever-attached gas. However, this only applies to gas payment, and the amount of coins we can send with the outgoing message is not restricted here, allowing the transaction to send more coins from the contract balance than what was present in the incoming message, even without tvm.accept().

As seen in the example above, we create an outgoing message and attach value = _deploy_evers to it, where _deploy_evers can be greater than the value in the incoming message.


Let's modify the example above:
```
function deployWallet(
  address _owner,
  uint128 _deploy_evers
) external {
  require(msg.value - 0.1 ever > _deploy_evers);

  TvmCell stateInit = tvm.buildStateInit({
     contr: TokenWalletContract,
     varInit: {
        root: address(this),
        owner: _owner
      },
      pubkey: 0,
      code: wallet_code
   });
   
  new TokenWalletContract{
     stateInit: stateInit,
     value: _deploy_evers,
     wid: address(this).wid,
     flag: 0
  }();

  msg.sender.transfer({ value: 0, bounce: false, flag: 64 });
}
```

Now we check if there are more 'ever' occurrences in the incoming message than 'gas + _deploy_evers', and at the end, we want to send back to the caller everything that remains after deducting the gas fee. This is also incorrect. Flag 64 - send as many coins as there were in the incoming message minus the gas spent. The trick here is that the number of coins we attached to the first out-coming message(_deploy_evers) is not considered in the gas spent because it is attached in the action phase. So, _deploy_evers will either be deducted from the contract's balance, or an error will occur if the contract's balance is insufficient.

In other words, the '64' flag can only be used if it is the only outgoing message in your function. It is most often used in simple 'responsible' functions, for example:

```
function balance() override external view responsible returns (uint128) {
  return { value: 0, flag: 64, bounce: false } balance_;
}
```

Let's write such function correctly.

```
function deployWallet(
  address _owner,
  uint128 _deploy_evers
) external {
  tvm.rawReserve(address(this).balance - msg.value, 2);

   TvmCell stateInit = tvm.buildStateInit({
      contr: TokenWalletContract,
      varInit: {
         root: address(this),
         owner: _owner
      },
      pubkey: 0,
      code: wallet_code
   });

  new TokenWalletContract{
     stateInit: stateInit,
     value: _deploy_evers,
     wid: address(this).wid,
     flag: 0
  }();

  msg.sender.transfer({ value: 0, bounce: false, flag: 128 });
}
```

So, we just reserved as many TVM Blockchain Native Tokens on the contract balance as there were on it before received the message, as the second action we sent a 'deploy' message, and as the third action, we sent back everything that was left after the first two actions to the caller.



However, the code above also has drawbacks, and it is the storage fee. Before the compute phase begins, a storage fee for its code + data will be charged from the account balance, and this means that the account balance will gradually decrease. But we want the contract to live as long as it is used at least once every few years.

Therefore, it is customary to make tvm.rawReserve not on the value address(this).balance - msg.value, but on some constant value recorded in the contract. For example, for the tip3 implementation from broxus, this is 1 Ever for TokenRoot and 0.1 for TokenWallet.


0.1 TVM Blockchain Native Tokens in the TokenWallet is enough for several years of storage, and if you have not used it for all this time, it will be frozen. That is, they will leave the account in the 'Frozen' state, delete the code and state, and leave only their hash. You will have a few more years to thaw it. The thawing process is similar to deployment, only you need to send a 'stateInit' of frozen data, not initial data.

From here we will move on to the question of how to calculate how much 'value' needs to be attached to the message in order for it to be enough to perform all actions in the call chain.

The gas price in TVM Blockchain is a constant value, and as the load increases, we do not increase the gas price, but add threads. So it is quite simple to calculate how much it costs to perform a certain action in each contract. Unfortunately, we currently do not have any instruments for automatically calculating the total gas in the call chain, so we need to run transactions manually. In 'locklift', the 'tracing' has a 'const gasUsed = traceTree?.totalGasUsed();' that shows the amount of gas in the chain. So we just need to look at all scenarios for executing our chain and calculate how much gas it needs.


There are two important nuances to consider when calculating the amount of value to attach to a message to be sure that it is sufficient to perform all actions:


Although the cost of gas is fixed, if a contract has storage operations with dynamic sized types(arrays, mappings, strings, etc), the cost of operations will increase logarithmically with the size of the storage. This has already been explained in the chapter on tvm, but let's repeat it here. The entire storage of a contract is one TvmCell with a reference to sub-cells. For example, if we have a mapping, it can be implemented like this:

![image](https://github.com/gosh-sh/datasets/assets/19651426/e857958e-4e1d-4513-af01-da9115b306ea)

Each circle in the picture is a separate cell. To get the value by key 2, TVM needs to load a cell of depth 0, then depth 1 and then depth 2. We have to pay gas for each time a cell is loaded. And if we change the value by key two , we will need to recalculate all references from the cell with the value of the root cell because the cell reference is a hash (cell.data + cell.refs). So, links to all cells along the way will change and we will need to change them from bottom to top.


So, the more elements our dictionary has, the deeper the cell will be and the more expensive it will be to work with. For a dictionary, the cost of gas will increase to O(log n) in a worst case scenario. (In reality, everything would be more complicated but O (log n) can be useful to look at as a worst case scenario).


The moral is simple - try to avoid mappings, arrays, mutable strings, and other structures of dynamic size, it is better to use distributed programming patterns when deploying a small contract for each entity.


If some contract in the chain still uses variables of dynamic size, check how much it will cost to interact with them when there are many elements, and add value with a large margin.


Basically, most contracts use tvm.rawReserve(someValue), which is a good practice. However, this means that you need to attach an additional amount of Evers to your messages that each contract reserves, and get change at the end of the chain. This is because if the contract has not been used for a very long time, the entire reserve will be taken from the value attached to the message (because the Evers on the contract will be eaten by storage).

Action phase errors
In addition to everything described above, there is also currently a bug in the node that causes bounce messages to not be created in the event of errors in the action phase. This means that if you incorrectly calculate how much value will be needed for a chain of transactions, and if the contract does not have enough value for rawReserve or to send a message, it will not create a bounce message even if there is enough value for it. Also, a bounce message will not be created if there is not enough value to pay for gas in the compute phase, in other words, there will not be enough value for creating the bounce message.

## External messages from contract to nowhere

When you emit Event(some_data), it is an external message that is sent from the contract to nowhere.


Its creation will always be paid for from the balance of the contract, and if you do not make tvm.rawReserve() before creating the event, be careful with the size of the data you are sending in this event. If, for example, you emit Event(player_name_string) and the user can enter any name of any length, such a message can be costly for your contract.



In principle, the information described above should be sufficient to understand how to calculate how many Ever-s need to be attached to a chain of transactions.


