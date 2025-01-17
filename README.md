# RoundingBugsNotes


# Issue explanation: `Dust` edge case. Bounty

https://x.com/gandu_whitehat/status/1803794103248806223

**Core problem**:

The core problem here, is that we have the dust mechanism in the system. So, when the burn function executed, it checks whether the amount after the burn less than dust. If yes, it clears out the balance 

<img width="504" alt="Screenshot 2024-11-06 at 10 19 53" src="https://github.com/user-attachments/assets/43028aba-3790-4b26-8f78-60460296d26f">


The vulnerability could occur in such way:

1. (Assume the dust thrash-hold is 10 wei), we deposit `11 wei` then. Given the 1:1 ratio, Account 1 receives 11 shares.
2. Account 2 deposits `1 wei` of the underlying token.Account 2 gets 1 share, given the 1:1 ratio.This action is to manipulate the pool's state and take advantage of the dust amount created by Account 1.
3. Account 1 initiates a redeem. So he redeem 1 wei of token, and after this operation, the balance of Account 1 is 10 Wei, which is at the dust threshold.
4. Dust Mechanism Triggered:
The redeem function checks if the remaining balance of Account 1 is less than or equal to 10 wei. Since it is, the function sets Account 1's balance to zero and considers the remaining 10 wei as dust.

**Final State:**
• Total shares: 1 wei (held by Account 2).
• Total Deposits: 11 wei (10 wei from Account 1 + 1 wei from Account 2), due to the dust amount created when Account 1's balance was zeroed.

So, we have the state where the `totalAmountOfUnderlying` tokens is 11 wei, while the `totalSupply` of shares is 1 wei

<aside>
💡 Above, we have achieved the initial conditions where totalSupply = 1 and totalAssets = 11. Now, if a user deposits 1 wei of assets, the shares minted to them would be calculated as (amount * totalSupply) / totalAssets = (1 * 1) / 11 = 0.09, which rounds down to 0.

</aside>

> Using this template of the donation attack we could inflate the prices and hack the protocol. Using this, the attacker can inflate the price of a share even if total assets are being tracked internally.
> 

**What steps lead auditor to find this bug?**:

Auditor knew this pattern in advance, so it was more easy for him to find it. However, the donation attack is the trickiest and the most profitable way of exploiting the bug, so we need to spend more time on it, trying to fuzz it, exploiting it.

**What question he posed that lead him to this bug?**

- There is a dust pattern, what if the user burn his amount to be = to the dust? His balance will be cleared out?
- How it will affect the `totalBalance` / `totalSupply` ? The will be a discrepancy, how we could exploit it?

**How to find it next time**:

Take a thorough look at the inflation/donation attack edge case, and use all my brainpower trying to exploit it, sit on it, like i was sitting at the ccip! 

# Issue explanation: `rounding` bounties idea’s. WiseLending

**Core explanation**:

A rounding donation attack requires the things:

1. A way to get one key value to empty / almost empty
2. A way to massively inflate another key number
3. A way to exploit the resulting rounding errors

### Intro

The first challenge was to get around the donation amount limits.
The attacker deposited a super tiny amount, then donated something under the amount limit. This passed the check, even though the percentage donation was 5,000,000x larger than the deposit

### stealth donation

1. The attacker then "exploited" the round-against-the-user rounding by intentionally loosing funds to protocol with a series of ever increasing deposits and withdraws, each sized to loose the maximum amount of funds to rounding. In effect, stealth donation by losing money. 
The protocol retains these cumulative losses, which can significantly increase the total assets in the protocol without a corresponding increase in the total supply of shares.
Then by withdrawing 1 token, the vault will burn 1 share from the user (by rounding up against him, as 1/3 = 0.3 ↑= 1), which will make a final donation of 2 tokens.
Over many such transactions, `totalAssets` could increase exponentially (e.g., to thousands of units of the underlying asset) while `totalSupply` remains largely unchanged.

> A lot of protocols integrated the check agains the external ‘donations’. However, because this one was happened due to internal rounding, the checks were bypassed.
> 
1. Once the attacker inflated the `totalSupply`-`totalAssets` he has made the final deposit of 6 figures to inflate the protocol completely.
   
    <img width="435" alt="Screenshot 2024-11-06 at 10 20 45" src="https://github.com/user-attachments/assets/cd3cab5c-36f8-4002-bb6a-1bc0a0fa32fc">

  
3. Once the attacker inflated the contract `totalSupply`-`totalAssets` significantly, so the 1 share costs a lot of tokens, attacker made a final large deposit into the pool from the **master contract**, this time for six shares worth. Additionally, attacker creates **extra helper contracts**, that did the same and acquire +6 shares, by depositing huge amount. So each of the attacker controlled accounts deposited significant amount of money to receive 1 share. One main contract = 6 share + helper contracts also have per 6 shares.
4. These shares were then used as collateral to borrow funds from other pools within the lending protocol. The value of the shares was more than the value of the borrowed funds, making the borrowing overcollateralized.
5. Next, the helper contracts asked to withdraw 1 raw token from its collateral. This was a super tiny amount, and would still leave the borrows overcollateralized. However, when the 1 raw was converted to shares and rounded up, 1 share was burned. And the helper lost all the collateral. Now this would be a disaster for a normal user. To make up numbers: if you had $1,000,000 in collateral, received $600,000 in borrows, and then lost the collateral, you would be out $400,000. In this case, what happened, is that the "lost" collateral now belonged to the PLP Pool, and i understand that this “collateral” included the money that we have “donated” at the beginning, to inflate the price. 
6. Due to the step that helper contract burned they shares, the protocol entered the insolvent state and accumulate funds from other funds, into the PLP pool. Finally, with the additional, accumulated shares, the attacker received more inflated amount, and using the 6 shares from the master contract, drained the protocol. After all the helpers had borrowed all the coins, while losing PLP hand over fist, the lending protocol was far richer. But all the protocol assets had moved from the coins of many pools to only PLP. And then the attacker withdraw all the PLP from the pool. GG.

**What steps lead auditor to find this bug?**:

He is Goat. Carefully prepared strategy. I guess that the main point were in the share calculation mechanism that he stares for a long time. He creates an state where one share equals a lot of assets. However, the main fundament of this attack was, without system being notified, create the state, where 1 share equals a lot of assets. And the only way of doing it was to constantly supply totalAssets-1 amount, to exploit the rounding to 0.

After that, he exploit the rounding mechanism, so the master contract loose the share. 

---

**Additional notes:** 

<aside>
💡 1. The most basic and widespread defense is to round shares/assets in favor of the vault:
2. What is taken from user should be rounded up, while what is given rounded down.
3. The second most deployed defensive mechanism is the decimal offset and dead shares deposit.

</aside>

Wise Lending implemented (1) an internal accounting (2) a throttle to limit the qty of token that can be directly donated to the contract. This (2) allow to account for direct donation if its below a certain value, while still making impossible to donate to an excessive point

<img width="502" alt="Screenshot 2024-11-06 at 10 21 22" src="https://github.com/user-attachments/assets/66fe92b1-247c-4a76-ade0-c61464a8ff53">


# Issue explanation: Abracadabra exploit. Inflating the `shares` via repaying all.

https://threesigma.xyz/blog/abracadabra-money-exploit

**Core problem**:

1. Attacker take the flashloan and triggered the `repayForAll` function, which allowed anyone to repay the debt of others. The main idea here is that such step incorrectly reduced `totalBorrowAssets` without adjusting `totalBorrowShares`, distorting the asset-to-share ratio.
    
<img width="786" alt="Screenshot 2024-11-06 at 10 22 03" src="https://github.com/user-attachments/assets/53189919-1413-417a-963f-3d9af21cfd78">

    
    💡 The attacker couldn't repay all of the amount as there was a check that `totalBorrowAssets` needed to be above a threshold of 1000 ETH. So, the attacker partially repaid the debt, manipulating the `totalBorrowAssets` to be greater than the threshold. As a result `totalBorrowAssets:totalBorrowShares` becomes approximately 1:26. In order to continue dropping the amount of assets, the attacker needed to manually repay liabilities for other borrowers.

    
2. After that, the attacker repay continues liabilities! The distortion was further exploited by repaying a negligible amount (100 wei) of shares for the last borrower, resulting in `totalBorrowedAssets` being 3 and `totalBorrowShares` 100.
3. The attacker then repaid **1 wei** (an extremely small amount) three times. Each repayment caused the `totalBorrowedAssets` to reduce to **0**, while `totalBorrowShares` reduced to **97 shares**.
4. From then on the attacker used a tiny bit of collateral, just 100 wei, and began a cycle where they borrowed 1 wei worth of assets and then paid back 1 wei of borrow shares. Because `totalBorrowShares` exceeded `totalBorrowAssets`, borrowing 1 wei of assets resulted in the creation (minting) of a **significant number** of borrow shares. However, since the protocol rounds up in its favour, when the attacker repaid 1 wei of borrow share, even though it is worth near 0, the protocol makes the attacker repay 1 wei of assets.
    
    💡 At the end, this loop inflated `totalBorrowShares` exponentially without increasing `totalBorrowAssets`, essentially making the shares value negligible.
    
    
5. After inflating the borrow shares, the attacker set `totalBorrowAssets` to zero while maintaining `totalBorrowShares` at an inflated level. The attacker then used another account to borrow the entire funds from the protocol against minimal collateral.
    
  <img width="577" alt="Screenshot 2024-11-06 at 10 23 00" src="https://github.com/user-attachments/assets/ad60b0df-398e-4cf9-81f1-2b4c7c996962">

    
6. After inflating the borrow shares, the attacker set `totalBorrowAssets` to zero while maintaining `totalBorrowShares` at an inflated level. The attacker then used another account to borrow the entire funds from the protocol against minimal collateral.

    <img width="645" alt="Screenshot 2024-11-06 at 10 23 16" src="https://github.com/user-attachments/assets/c9e505af-5f31-44a9-9384-f98c00db40eb">


**What steps lead auditor to find this bug?**:

So, we have the end function called `toBase`. However, starting from the beginning the main idea was to manipulate one the param (totalAssets/totalShares). So, the attacker end up with the idea that if we could ‘repay’ all of the assets of the user, it will not decrease the share. Here is where the interesting thing come to play, because for most of the auditor it would be enough to receive ‘high’, but attacker manage to exploit it. Secondly, when the attacker started to cycle of borrowing/repaying he also was sure that this 1 wei will be returned due to rounding. 

# Issue explanation: `rounding` error in the Graph

[The Graph Rounding Error Bugfix Review](https://medium.com/immunefi/the-graph-rounding-error-bugfix-review-c946ff470f65)

**Core problem**:

There were 2 problems. The first one was about the rounding error, the second one that the user could unstake the money without a lock period.

```solidity
function tokensToSignal(bytes32 _subgraphDeploymentlD, uint256 _tokensIn) public view override returns (uint256, uint256) 
{ 
uint256 curationTax = tokensIn.mul(uint256(curationTaxPercentabe)).div(MAX PPM);  
uint256 signalOut = _tokensToSignal(_subgraphDeploymentID, _tokensIn.sub(curationTax)); return (signalOut, curationTax); 
}
```

On the moment of the submission of the bug report the curation tax percentage was capped at 1% (variable equal to `uint256(10000)`), and the constant `MAX_PPM` was capped at 100% (variable equal to `uint256(1000000)`). That means that if an attacker is passing `uint256(99)` as a `tokensIn` variable, he will need to pay 990000/1000000 of the `curationTax`, which is rounded down to 0.

The second problem was that the user could unstake unfairly early the tokenAmount. The `lockTokens` function adds the unstaked tokens to an existing pool containing previously unstaked and locked tokens. This then calculates the new duration of lock to a value between the current lock duration and the maximum locked duration, which was capped at 28 days at the time of submission.

```solidity
function lockTokens(
        Stakes.Indexer storage stake,
        uint256 _tokens,
        uint256 _period
    ) internal {
        // Take into account period averaging for multiple unstake requests
        uint256 lockingPeriod = _period;
        if (stake.tokensLocked > 0) {
            lockingPeriod = MathUtils.weightedAverage(
                MathUtils.diffOrZero(stake.tokensLockedUntil, block.number), // Remaining thawing period
                stake.tokensLocked, // Weighted by remaining unstaked tokens
                _period, // Thawing period
                _tokens // Weighted by new tokens to unstake
            );
        }

        // Update balances
        stake.tokensLocked = stake.tokensLocked.add(_tokens);
        stake.tokensLockedUntil = block.number.add(lockingPeriod);
    }
```

If there are some tokens already unstaked and sitting in the locked pool, the calculation of a new lock time will be done like this:

`newLockedUntil = ((currentLockedUntil * currentlyUnstakedTokens) + (201600 * newUnstakedTokens)) / (currentUnstakedTokens + newUnstakedTokens)`

If `currentUnstakedTokens` is roughly 201600 times larger than `newUnstakedTokens`, `newLockedUntil` will evaluate to `currentLockedUntil`. Thus, the attacker can pass a small enough value to the `unstake` function to withdraw small portions of the staked tokens without any additional lock applied. After that, he could proceed with the same attack plan as in the first bug described in the article: unstake small batches of tokens to avoid adding any value to the lock period.

**What steps lead auditor to find this bug?**:

Create all possibles edge cases, related to the rounding errors.

**What question he posed that lead him to this bug?**

- What numbers play the role in the calculation?
- What if i provide the smaller number, could i receive 0?
- What if i receive 0, how could i exploit the protocol?
- What prerequisites need to exist, so i could round to 0?

**How to find it next time**:

Carefully examine rounding issues.

# Issue explanation: `ransome` contract

[How an informational issue ended up securing (almost) half a mil…](https://mirror.xyz/0xCf39521413F8De389771e35bB4C77b4bb827b7B3/HdSq7TVvk-s7DzQgN3u0pV8UFiVkaDft18HgmePTag4)

**Core problem**:

The core problem was around a function, that would allow to set an arbitrary token address.

```solidity
function setToken(address _token) public stablePhaseOnly {
        require(_token != address(0x0), "Address must be set");
        require(_token.supportsInterface(type(IERC20).interfaceId), "not a valid ERC20 token");
        // require(ERC165(_token).supportsInterface(type(IERC20).interfaceId), "not a valid ERC20 token");

        ERC20 erc20 = ERC20(_token);
        require(strcmp(erc20.symbol(), "TAL"), "token name is not TAL");

        token = _token;
    }
```

`setToken()` let anyone set the address of TAL token with arbitrary code as long as it was named “TAL” and followed the ERC20 standard.

There are 2 stages in the protocol, for staking they native token, as well as stable coin. So, initially the vulnerability was informational, once anyone set the malicious token address, it would prevent from the execution and grief the protocol during the `stablePhaseOnly`

```solidity
modifier stablePhaseOnly() {
        require(!_isTokenSet(), "Stable coin disabled");
        _;
    }

    /// Allows execution only while in token phase
    modifier tokenPhaseOnly() {
        require(_isTokenSet(), "TAL token not yet set");
        _;
    }
```

However, after some time figuring out the bug, the attacker realizes that there is no option to steal the funds. However, during the `swap` function, the malicious token is called on the transferFrom. 

```solidity
function swapStableForToken(uint256 _stableAmount) public onlyRole(DEFAULT_ADMIN_ROLE) tokenPhaseOnly {
        require(_stableAmount <= totalStableStored, "not enough stable coin left in the contract");
        uint256 tokenAmount = convertUsdToToken(_stableAmount);
        totalStableStored -= _stableAmount;
        IERC20(token).transferFrom(msg.sender, address(this), tokenAmount); // --> bingo, freeze time! zzzzz *freezing beam sounds*
        IERC20(stableCoin).transfer(msg.sender, _stableAmount);
    }
    }
```

So, we could make a tx revert on transfer, and it would permanent freezing of the funds, which is still enough for the critical. But ! What if we could create a ransom contract?

```solidity
 		bool ransom_paid;
    address owner;
    function withdrawRansom() public {
         if (address(this).balance > 100 ether) {
             payable(owner).transfer(address(this).balance);
             ransom_paid = true;
         }
    }
    function transferFrom(address from, address to, uint256 amount) public override returns (bool) {
        if (!ransom_paid) {
            revert("Ransom is not paid!");
        }
        return true; // ransom paid, unlock funds
    }
```

It means that the attacker can poses the protocol, and ask for an ransome. It is fucking cool!

**What steps lead auditor to find this bug?**:

Firstly they find an opportunity to set a malicious token and grief the protocol, and the owner could simply redeploy the protocol and spend the time/gas. However, moving forward they explore that the contract basically interacts with the token during the `swap` function. They figure out that if the token will revert on transfer it could result in the total protocol stop, which is enough. However, how to earn money as an attacker? Create the ransome contract! And “release” the contract, once the contract will be paid.

**What question he posed that lead him to this bug?**

- We could  grief the protocol, however could we create the benefit for an attacker?
- The contract interacts with our/malicious token, how we could break the protocol?
- Okay, we can stop the protocol if the revert on transfer….

**How to find it next time**:

1. Never leave the potential issue as it is
2. Always find the way how the protocol can interact with your malicious “contract”.

# Issue explanation: outdated `healthFactor`. Bug by Trust

https://www.trust-security.xyz/post/diving-deep-into-a-critical-protocol-insolvency-bug-in-fringe-fi-lending-platform

**Core problem**:

The core vulnerability of this classical lending contract, is that the `updateInterestInBorrowPositions()` function, which updates the accrued interest, is only called when the user requests to withdraw the maximum amount (`UINT256_MAX`).

The issue is that borrowers may withdraw collateral using non-MAX amount, and leave undercollaterized position in the contract. The accrued variable is stale, so it's not counted against the borrower

- ***Explanation of the vulnerability***
    
    ### 1. Initial Setup:
    
    - The attacker deposits Frax Share tokens worth $100 into the protocol. Due to the Loan-to-Value Ratio (LVR) of 71%, this gives them $71 worth of lending power.
    - The attacker borrows $20 from the protocol using their deposited tokens as collateral.
    
    ### 2. Time Passes:
    
    - Assume the borrow Annual Percentage Yield (APY) is 16.8%, meaning the debt grows over time due to interest.
    - Meanwhile, the lender APY is 8.6%, meaning the protocol pays out interest to users who lend money.
    
    ### 3. Interest Accumulation Issue:
    
    - Over 8 years, the borrower's debt (the "loanBody") should grow significantly due to the high borrow APY.
    - However, due to a bug in the system, the interest (accrual) on the borrower's debt is not correctly updated.
    - This means that the protocol still sees the attacker’s debt as $20 instead of the correct amount, which should be around $69.3 after 8 years (original $20 loan + $49.3 in accumulated interest).
    
    ### 4. Withdrawal of Collateral:
    
    - The attacker then withdraws $71.8 worth of their deposited tokens, leaving only $28.2 worth of tokens in the protocol.
    - The system checks if this withdrawal leaves the health factor (collateral value / debt value) above 1, which it mistakenly does because the debt is still incorrectly recorded as $20.
    
    ### 5. The Protocol's Loss:
    
    - After the withdrawal, the attacker's remaining collateral is $28.2.
    - The protocol incorrectly thinks this is sufficient to cover the $20 loan because it hasn’t accounted for the $49.3 in accumulated interest.
    - In reality, the loan should be $69.3, which means the protocol is now under-collateralized (collateral is less than the debt).
    
    ### 6. The Final Balance:
    
    - The attacker now has $71.8 in withdrawn tokens and $38.7 in borrowed tokens (the original $20 plus $18.7 earned from lending that $20 at 8.6% APY over 8 years).
    - The protocol, however, is left with only $28.2 in collateral for a loan that should be $69.3, meaning it has lost $41.1.
    
    ### Summary:
    
    The protocol loses money because it fails to update the borrower's debt with the accumulated interest, allowing the attacker to withdraw more collateral than they should be able to. This leaves the protocol with an under-collateralized loan and a financial loss.
    
    4o
    

So, the overall problem is that the `updateInterestInBorrowPositions()` is triggered only when the user withdraw the type(uint256).max, in any case it is not triggered and leave the accrue interest remains 0. Time after, the user could withdraw the amount that he deposited, plus the interest accrued on the lend money, however, the protocol will remain insolvent because the health factor will remain below 0, because it doesn’t accrue for the interest for the borrowed funds.

**What steps lead auditor to find this bug?**:

The main idea was here is that the auditor carefully examined the flow of the interest updating in the lending. Since it the most important part, because it determines that the protocol is solvent and all the debt correctly accounted. He come up that the one of the important feature is updated, only when the all value is passed. Which is incorrect.

**What question he posed that lead him to this bug?**

- How the debt and all params related to the interest are updated?
- Do they updated at right point in time?
- Is there a way to skip updating?
- What if attacker would provide input as min/max? What could happen?

**How to find it next time**:

In the lending, carefully examine the updating mechanism, since if it isn’t updated at correct point in time, the whole protocol will be squezzed. Liquidation and all this stuff also depends on it, so pay attention on the details and when and how it is updated. The updating flow must be the main point of the concentration.

# Issue explanation: `incorrect amount` is taken during an liquidation

https://medium.com/@smaul_1/enhancing-protocol-integrity-addressing-bugs-in-the-lybra-finance-contract-21c1e4b68387

**Core problem**:

During a liquidation instead of the `borrowedAmount`, the `depositedAmount` is used, as a result, more than the intended 50% debt can be liquidated, potentially affecting the borrowers' expectations and the protocol's risk management.

The second issue was based on the bug that the protocol doesn’t ensure that the depositor could hold the 7 days period. Instead he could leave it as it is.

**What steps lead auditor to find this bug?**:

Carefully examine which numbers play a role during a important process, whether they are correct and not affected by the strange/unnecessary calculation.

**What question he posed that lead him to this bug?**

**How to find it next time**:

Carefully take a look and divide into the peaces what plays a role during the calculation process. 

# Issue explanation: user deposit 0, but receive the `share` due to rounding

https://medium.com/immunefi/dfx-finance-rounding-error-bugfix-review-17ba5ffb4114

**Core problem**:

The core problem is that protocol allows to deposit the tokens into the AMM pool, however in case of the EURS token that has 2 decimal, the rounding could occur and user will receive some amount of shares, but will transferFrom will be 0 due to the rounding.

The shares are calculated based on the ratio of the deposited amount to the total pool value. But the actual `_amount` being transferred is calculated like this.

```solidity
amount_ = (_amount.mulu(10**tokenDecimals) * 1e6) / _rate;
```

So, the user could deposit minuscule amount of tokens, the share will be minted to them, but the `amount_` will be recalculated and user will transfer no funds.

**What steps lead auditor to find this bug?**:

Rounding error. When i send no funds but receive the shares it is very profitable vuln. So i must always concentrate and keep in mind. What if i could send a specific amount of token with some not usual amount of decimals. Could i receive the shares, with the amount being rounded during the transfer?

**What question he posed that lead him to this bug?**

- Are there some tokens with strange decimals?
- Could we send the value so we receive the share but transfer no fund due to rounding?

**How to find it next time**:

Keep an eye on the strange decimals. If there is, try to think in a way whether we could send the money as 0, but receiving the shares.

# Issue explanation: Chainlink roundCount and protocol “versions” `mismatch`

[The Napping Oracle](https://mirror.xyz/0x9D6b7f5e8d1b9dFea8dDD29c0DbD81687e721601/mm_D_HrqfntAkGM1DvVQvy1WuPbj99pKYfRp-xDbs8U)

**Core problem**:

The protocol make a ‘bet’ on the chainlink price feed. So each time the round is incremented in the chainlink, it must be incremented in the protocol. The problem is that it returns the `totalRoundAmount` of the specific phase. So, if we have 20 rounds, the phase will be updated and we will return 20 one more time and the final result still will be 20 instead of 40.

```solidity
while (round.phaseId() > _latestPhaseId()) {
   uint256 roundCount = registry.getRoundCount(base, quote, _latestPhaseId());
   _startingVersionForPhaseId.push(roundCount);
}
```

After, (it was in the 2 versions), if the protocol will try to get version 3, it will retrieves the starting round ID for phase 3 from Chainlink, which is the correct round ID for the start of phase 3.

<aside>
💡 The function calculates the round ID by adding the difference between the version (41) and the incorrect starting version (20), resulting in a round ID that is 20 rounds in the future.

</aside>

**What steps lead auditor to find this bug?**:

Once he finds the bug, he takes a look at all possible places where it occurs.

<img width="938" alt="Screenshot 2024-11-06 at 12 08 51" src="https://github.com/user-attachments/assets/5e667616-bb4f-4478-9528-798f291c874d">


# Issue explanation: Beanstalk `transferFrom` vulnerability

https://medium.com/immunefi/beanstalk-logic-error-bugfix-review-4fea17478716

https://bailsec.io/tpost/ejyrnasvy1-the-diamond-proxy-simply-explained

**Core problem**:

The beanstalk implemented the diamond proxy patter (we have one diamond proxy, and a lot of facets. Each facet contains some part of the implementation, so different facet have different logic). One of the facet had the `transferFrom` functionality. It could be triggered in 2 modes , internal and external, but the problem if the attacker want to transferFrom the tokens from some user, the `allowance[attacker, user]` would be check only for the internal transfer. For the external mode, the allowance would be skipped, and attacker could drain the funds of the user who has given the approval for the Beanstalk

```jsx
function transferToken(
       IERC20 token,
       address sender,
       address recipient,
       uint256 amount,
       From fromMode,
       To toMode
   ) internal returns (uint256 transferredAmount) {
       if (fromMode == From.EXTERNAL && toMode == To.EXTERNAL) {
           uint256 beforeBalance = token.balanceOf(recipient);
           token.safeTransferFrom(sender, recipient, amount);
           return token.balanceOf(recipient).sub(beforeBalance);
       }
       amount = receiveToken(token, amount, sender, fromMode);
       sendToken(token, amount, recipient, toMode);
       return amount;
   }
```

**What steps lead auditor to find this bug?**:

Auditor carefully examined the main flow of the token flow. Since the transfer method was overridden , it was important to check that it works properly in both modes. Also, when there is a function that allow to do smth “onBehalf” of someone, we need to carefully examine whether all necessary allowance are received and user could do any unauthorised actions with the users funds. 

**What question he posed that lead him to this bug?**

- What if the attacker withdraw onBehalf of the user, could he do it?
- Are there the check ensure no unauthorised access? Allowance is properly checked?
- What is the difference of the transfer behavior in both modes? Could we abuse it? Are they properly implemented

**How to find it next time**:

Funds flow, examine and comment every step in the fund flow. Devil is in details.

# Issue explanation: Oasis vuln. by TRUST

https://www.trust-security.xyz/post/taking-home-a-20k-bounty-with-oasis-platform-shutdown-vulnerability

**Core problem**:

> After almost a decade of vulnerability research, I am beyond convinced that 99% of our work is to reason about hidden assumptions made during development (1% is technical skills)
> 

Each user hold the proxy created for every user. Essentially, users always keep their funds in a private DSProxy smart wallet. To perform a trade, they delegate execution to Earn smart contracts which implement the trading strategy whilst staying in the DSProxy context (delegatecall pattern).

The `OperationExecutor` contract is the contract directly called by DSProxy. It receives an array of calls (service hash + calldata) and an operationName and will execute the operation. **So the main point here is to break the OperationExecuter**.

`OperationExecutor` verify the strategy and the call being made. It must be whitelisted

<aside>
💡

**ASSUMPTION**: `executeOp` will run in the context of user's DSProxy

</aside>

<aside>
💡

**ATTACKER ASSUMPTION**: call the `executeOp` directly, and corrupt the *existence* of OperationExecutor

</aside>

***However, there is a barrier:***

1. For existing operations, the function requires call target hashes that exactly match the registered action array for that operation, which limits flexibility and can cause errors if the inputs aren't identical.
    
    ```jsx
    if (hasActionsToVerify) {
      opStorage.verifyAction(calls[current].targetHash);
    }
    ```
    
2. The second issue is that the delegatecall target is resolved by ServiceRegistry's service mappings, which can only be changed by owner:
    
    ```jsx
    address target = registry.getServiceAddress(calls[current].targetHash);
    target.functionDelegateCall(
      calls[current].callData,
      "OpExecutor: low-level delegatecall failed"
    );
    ```
    
    ***So, to be able to proceed further we need to understand what is each ‘target’ and which strategy is implemented in these target and how we can intervene there? If there's a code path from one of these services leading to a controlled `delegatecall` (using calldata). So, we need to attacker → callAction → delegateCall → contract → delegateCall
    
    One of the dst. services was Proxy of the AAVE lending Pool, which exactly what we are looking for, it could make a delegateCall to the address that we want, and everything is still in the context of `OperationExecutor`.
    
    ```jsx
    function initialize(address _logic, bytes memory _data) public payable {
        require(_implementation() == address(0));
        assert(IMPLEMENTATION_SLOT == bytes32(uint256(keccak256('eip1967.proxy.implementation')) - 1));
        _setImplementation(_logic);
        if (_data.length > 0) {
          (bool success, ) = _logic.delegatecall(_data);
          require(success);
        }
      }
    ```
    
    However, it still doesn’t match because even if we could delegateCall forward, we can’t call the function “initialize” as a strategy, because the exact dst expect as the predefined/whitelisted strategy, because it will fail on the `verifyAction` step.
    
    ***But we don’t give up and we need to explore the newly added strategies into OperationRegistry to make sure that we don’t miss nothing
    
    <img width="866" alt="Screenshot 2024-11-06 at 10 26 19" src="https://github.com/user-attachments/assets/3840b0b8-bfc0-492e-a247-89c176387af8">

    
    However, recently the team made the strategy name called CustomOperation, which has no actions, and when no action is set to the name of the strategy, the `verifyAction` will be bypassed.
    
    ```jsx
    bool hasActionsToVerify = opStorage.hasActionsToVerify();
    for (uint256 current = 0; current < calls.length; current++) {
      if (hasActionsToVerify) {
        opStorage.verifyAction(calls[current].targetHash);
      }
      address target = registry.getServiceAddress(calls[current].targetHash);
      target.functionDelegateCall(
        calls[current].callData,
        "OpExecutor: low-level delegatecall failed"
      );
    }
    ```
    
    So, it seems like passing CustomOperation gives us the last critical primitive - action array bypass! Passing CustomOperation means we can specify any service that exists in the ServiceRegistry, regardless of its existence in the OperationRegistry. Call any service with our data. And it basically means that we could implement our theory with 2 delegate calls selfdestruct the protocol contract.
    

**What steps lead auditor to find this bug?**:

1. Thorough testing of the theories
2. 

**What question he posed that lead him to this bug?**

- what if the could provide malicious strategy.
- there is an assumption, but what steps could break this assumption?
- what if we call the contract directly, not from the proxy?
- what if there will be a function in the targets that would double delegate? COuld we self destruct?
- When the verifyAction will be bypassed? In what cases? How could i reach this state?
- What is the hidden assumption of this code? How we could brake this assumption?

**How to find it next time**:

If it is a bounty, maybe the tricky parts are located in the some recently made actions by the protocol, so keep an eye on it. Also, if have some kind of “actions” that must be verified, we should carefully take it into the examination and explore to which protocols these actions lead, and if so, explore how this external protocols can be exploited, and how we could “take a flow” there? Try to think outside the box.

# Issue explanation: `_beforeTokenTransfer` fails to update the state if to is address(0)

https://medium.com/immunefi/apwine-incorrect-check-of-delegations-bugfix-review-7e401a49c04f

**Core problem**:

<img width="577" alt="Screenshot 2024-11-06 at 10 27 02" src="https://github.com/user-attachments/assets/d47dcff7-c35e-441e-b1b7-0c5c311cad1e">


The problem here is that if the to will be address(0) the user state will not be updated in the hook which would allow the attacker to profit the double spending.

To be able to exploit we need to deposit the tokens and then delegate them to the attacker controlled address.

After, we would withdraw our tokens, however, once we will call the withdraw functions, the to will be address(0) and the state updating will not happen and the **protocol fails to account for the PTs that were delegated**. When the attacker withdraws their PTs, they receive their entire deposit back while the inflated delegation amount remains untouched. Eventually, the attacker could wait a few periods for the inflated delegation to generate yield, mint PTs and FYTs and call withdraw to redeem these tokens to IBT.

**What steps lead auditor to find this bug?**:

He carefully understood that the important state transition happens in the function A, B, C → so he made a heavy accent on it to be able to found out whether we could brick something and prevent from the correct state updating.

**What question he posed that lead him to this bug?**

- There is a hook, what if the hook will not be executed?
- Could we send to the address 0?
- What if we burn the tokens, what would happen?
- Where the most state updating parts are located?
- What if we could brick the state updating? On which blocks does it stays? How to put out this blocks?

**How to find it next time**:

Carefully examine the state updating logic and pay attention on it. Always there is a chance to break the updating, and once we will do it we could hack the protocol.

# Issue explanation: **Silo Finance Logic Error**

https://medium.com/immunefi/silo-finance-logic-error-bugfix-review-35de29bd934a

**Core problem**:

The core idea was that the attacker could manage to break the utilization rate of the pool that has still no funds, thus inflate the interest rate.

An attacker can manipulate the utilization rate by donating an ERC20 asset to the contract, and if the attacker had the majority of the shares in the market for that particular asset, borrowing the donated token would inflate the utilization rate of that particular asset

<img width="376" alt="Screenshot 2024-11-06 at 10 27 32" src="https://github.com/user-attachments/assets/0011a3fc-f476-4779-bb81-fa3abab720ae">


```jsx
 /// @inheritdoc IBaseSilo
   function liquidity(address _asset) public view returns (uint256) {
       return ERC20(_asset).balanceOf(address(this)) - _assetStorage[_asset].collateralOnlyDeposits;
   }
```

In the next block, due to `silo.assetStorage[WETH].totalBorrows` being way more than `silo.assetStorage[WETH].totalDeposits`, which would accrue the interest of amore than 5000 ETH, the 1e5 WETH of collateral that the attacker initially deposited is now worth more than the 5000 ETH, due to the inflated interest rate for the borrowed WETH token.

# Issue explanation: roundingError in the EUSD deposit

[DFX Finance Rounding Error Bugfix Review](https://medium.com/immunefi/dfx-finance-rounding-error-bugfix-review-17ba5ffb4114)

**Core problem**:

The core problem here is that during the deposit, the amount that is actually transferred from the user is calculated like this.

```solidity
uint256 _usdcBal = usdc.balanceOf(_addr).mul(1e18).div(_quoteWeight);

uint256 _rate = _usdcBal.mul(10**tokenDecimals).div(_tokenBal);

amount_ = (_amount.mulu(10**tokenDecimals) * 1e6) / _rate;
```

However, such method is vulnerable because, if it would be a usdc token, everything would be fine, but once we find out the tokens with the different decimals, we could profit in a such way that we would transfer 0 amount but still receive the LP tokens. And repeatedly execute it via the loop could give us the exploit

 

**How to find it next time**:

The main point to look at in the future is to see whether the “`actual`” amount that we transfer is calculated somehow. If yes → we need to check what entities plays the role and could we endUp in the situation when we transfer 0 amount? Mb different token decimals on the different chains which aren’t taken into an account?

# Issue explanation: bug by `deadrosesxyz`

**Core problem**:

People lock the project's token and based on the duration of the lock, they're allocated voting power. The ownership of the lock is stored in the way of an ERC721 (NFT). Every week users can vote for a gauge. When they vote for it, they 'deposit' the voting power into the gauge's bribe. To deposit, all the funds are withdrawn from the gauge

> *To summarize it - anytime `vote` is invoked, it first makes a call to `reset`, withdrawing all current votes from the corresponding bribes. After votes are successfully reset, the user 'deposits' into the bribes of the gauges they're voting for*
> 

The main problem is that during the withdraw which happened during the reset aka ‘clearing the gauge’ , the user could withdraw the special amount and not trigger the updating mechanism during the withdraw

```jsx
    function withdraw(uint256 amount, uint256 tokenId) external nonReentrant {
        require(amount > 0, "Cannot withdraw 0");
        require(msg.sender == voter);
        uint256 _startTimestamp = IMinter(minter).active_period(); 
        address _owner = IVotingEscrow(ve).ownerOf(tokenId);

        // incase of bribe contract reset in gauge proxy
        if (amount <= _balances[_owner][_startTimestamp]) {//@audit only if
            uint256 _oldSupply = _totalSupply[_startTimestamp]; 
            uint256 _oldBalance = _balances[_owner][_startTimestamp];
            _totalSupply[_startTimestamp] =  _oldSupply - amount;
            _balances[_owner][_startTimestamp] =  _oldBalance - amount;
            emit Withdrawn(tokenId, amount);
        }

    }
```

### Attack Path explained

This attack path exploits vulnerabilities in a voting and reward distribution system, allowing a user to inflate their balance in multiple gauges to claim disproportionate rewards. The exploit combines the ability to vote with NFTs of varying value, reset balances, and manipulate vote counts across multiple gauges. Here’s a step-by-step breakdown of how the attack works:

### Step-by-Step Breakdown:

1. **User Mints Low-Value Lock for Each Gauge (Dust Voting)**
    - The attacker mints a lock with a **very small amount of tokens (1 wei)** for each gauge (voting category). These small votes go unnoticed because they don't have much impact on the system.
    - For each gauge (G1, G2, G3, etc.), they assign **one low-value NFT**.
2. **User Distributes Votes Across Different Gauges**
    - The user votes with **one low-value NFT (1 wei)** on each of the different gauges. This establishes a presence in multiple gauges with minimal token commitment.
    - Since the value is extremely low, this action does not raise any red flags.
3. **User Mints a High-Value NFT**
    - The attacker mints another lock for a **high-value amount of tokens** (e.g., 1000 tokens).
    - This high-value NFT can be used for future manipulations because its balance is significantly larger.
4. **Wait for a Week (Balance Reset)**
    - The attack waits for **one epoch** (e.g., one week), during which voting results are processed.
    - By the end of the week, the user’s balance in each gauge should now be **reset to 0** because the dust votes (1 wei) were negligible in impact.
5. **Vote with the High-Value NFT in the First Gauge**
    - The attacker votes with the **high-value NFT** on **Gauge 1**. This sets their balance in Gauge 1 to **1000 tokens** (the value of the high-value NFT).
    - Now, their balance in this gauge is high, corresponding to the large amount of tokens they’ve committed via the high-value NFT.
6. **Call `vote.reset` for the Low-Value NFT on the Same Gauge**
    - The attacker then calls `vote.reset` for the **low-value NFT** (1 wei) on **Gauge 1**.
    - The system subtracts the balance of the low-value NFT from the high-value NFT's balance (i.e., **1000 tokens - 1 wei**).
    - The result is that the attacker’s balance in Gauge 1 remains high (essentially **1000 tokens**).
7. **Call `vote.reset` on the High-Value NFT**
    - The attacker then calls `vote.reset` for the **high-value NFT**. Normally, this would reduce their balance in Gauge 1. However, since the remaining balance is smaller than the high-value NFT’s balance (i.e., **1000 tokens - 1 wei**), the system does not reduce the balance any further.
    - Essentially, the system now incorrectly believes that the user has **no votes in Gauge 1**, but their balance remains at **1000 tokens**.
8. **Repeat for All Other Gauges**
    - The attacker repeats this process for every other gauge (G2, G3, etc.). They:
        1. Vote with the high-value NFT in a gauge.
        2. Reset the low-value NFT.
        3. Reset the high-value NFT.
    - Each time, the attacker’s balance is artificially inflated in each gauge.
9. **Transfer High-Value NFT to Another Wallet**
    - After inflating their balances across all the gauges, the attacker transfers the high-value NFT to a different wallet.
    - This new wallet can now repeat the process by leveraging **the low-value NFTs** they set up in step 1.
    - Since the system does not track or connect the NFTs to the user’s actions properly, this cycle can be repeated **endlessly**.
10. **Exploit Rewards System**
- In the end, the attacker has artificially high balances in all the gauges, allowing them to claim **a disproportionate share of rewards** from each gauge.
- Even though the NFTs are no longer used for voting, the attacker still receives rewards because their balances across gauges remain high.
- Furthermore, because the balance is distributed across multiple wallets, the system may not detect any suspicious activity or unusually high balances, allowing the attack to go unnoticed for a long time.

### Key Vulnerabilities:

1. **Insufficient Balance Reset Logic**: The system’s balance reset mechanism allows the attacker to keep an artificially high balance even after resetting votes.
2. **No Proper Connection Between NFTs and Balance**: The balance is decoupled from the actual NFTs that are being used for voting, allowing the attacker to manipulate votes and balances.
3. **Multiple Wallet Exploit**: By transferring NFTs to other wallets, the attacker can continue the exploit with little risk of being detected.
4. **Low-Value Voting Manipulation**: The attacker uses minimal value NFTs to set up the exploit without raising alarms.

### Conclusion:

This attack exploits the improper resetting of votes and balances. By leveraging both high- and low-value NFTs, the attacker is able to manipulate their balances across multiple gauges and gain access to disproportionate rewards without actually voting with those NFTs. The system’s inability to properly reset balances and track NFTs across wallets allows this exploit to be repeated multiple times, making it a serious vulnerability in the reward system.

# Issue explanation: ring buffer

**Core problem**:

The core problem is that we have the ring buffer structure in the order book of the NFT, so, when the ring buffer is filled up the first index and other is overwritten.

<img width="1017" alt="Screenshot 2024-11-06 at 10 28 42" src="https://github.com/user-attachments/assets/fdeb2855-aafe-4488-97da-f87f49af2ea9">

If we take closer look, it will become clear that the attacker could potentially own the approvals of the future NFT’s with the same index in the ring buffer

Attack:

1. Attacker create the order and receive the NFT with the index 0.
2. The attacker fill up the orders from 0 to MaxBuffer, meanwhile cancelling the previous orders. The main point here is that attacker “approves” the token NFTs to himself and then cancel.
3. The user create the order but due to the ring buffer he receives the recycled value of the tokenId(which was already used)

<img width="1016" alt="Screenshot 2024-11-06 at 10 29 10" src="https://github.com/user-attachments/assets/7a84099c-4826-4d13-97e3-a603f25f10d3">


To remedy this bug the order function must check whether this tokenId was burned or not already minted.

# Issue explanation: incorrect handling of the epochs during the rewards calculation

https://x.com/SpearbitDAO/status/1706741516582957135

**Core problem**:

The function receives the amount of funds from the user and call the notifyRewardAmount() to update the rewards that should be distributed.

<img width="504" alt="Screenshot 2024-11-06 at 10 29 45" src="https://github.com/user-attachments/assets/9d1ace0d-48b3-45a3-879d-85688fad3967">


If we take a look on the fn() we could see the problem. When we get the index we do it on the first second on the next second instead of the last time on the 

<img width="504" alt="Screenshot 2024-11-06 at 10 30 02" src="https://github.com/user-attachments/assets/a5f663c1-c6bd-483f-ae77-4e9fe8193cac">


So, it means that once we deposit the rewards, some of them will be assigned to the next epoch!

<img width="571" alt="Screenshot 2024-11-06 at 10 30 16" src="https://github.com/user-attachments/assets/803414e0-20e9-4cb4-9d81-50128e849858">


- If a user **deposits at the first second of the next epoch**, their deposited amount will incorrectly contribute to the **total supply of the previous epoch**, rather than the current one. So, the rewards will be locked.

**How to find it next time**:

When we will deal with the reward/epochs , make sure that everything is assigned to the correct epochs, brainstorm the ideas “what if” i deposit on the first seconds, since it could result in the critical. We need to not step to the next epoch!

# Issue explanation: attacker can grief the protocol, resulting in the permanent grief of the protocol

https://mirror.xyz/0x333247F2e126954ed6428e9135Ae9dE06A76BA32/Hhs0AGFqqemCljNa49AnYVUTrLPCvdyPtd23k4iwQ_M

**Core problem**:

The are 2 bugs here, the first one lies in the idea that we can deposit the funds onBehalf of the user, however, once we done it the strange line is executed.

```solidity
function internalDeposit(uint256 _amount, address _payer, address _who) private {
        require(getTime() < depositWindowEnd);
        require(ecoToken.transferFrom(_payer, address(this), _amount));
        DepositRecord storage _deposit = deposits[_who];
        uint256 _inflationMult = ecoToken.getPastLinearInflation(block.number);
        uint256 _gonsAmount = _amount * _inflationMult;
        address _primaryDelegate = ecoToken.getPrimaryDelegate(_who);
        ecoToken.delegateAmount(_primaryDelegate, _gonsAmount);
        **_deposit.lockupEnd = getTime() + duration;**
        _deposit.ecoDepositReward += (_amount * interest) / INTEREST_DIVISOR;
        _deposit.gonsDepositAmount += _gonsAmount;
        _deposit.delegate = _primaryDelegate;
    }
```

Basically, it increases the lockUp time of the guy who we deposit onBehalf. The ‘`duration`’ is set upon initialisation, so it is constant. So, we could basically deposit 0 value, which is possible, and after that we grief the guy. Once he decides to withdraw the money, he could do it, but paying the ‘penalty’ because he does it to early, and his token will be burned.

The second bug is more interesting. We could lock the funds of the victim, to do it, we should

1. Victim deposit 100 tokens
2. Attacker deposit the same amount of 100 tokens
    1. `internalDeposit` is doing delegation via `ecoToken.delegateAmount(_primaryDelegate, _gonsAmount);`, where the `_primaryDelegate` will be the victim
3. Once we changed it to the victim, attacker made the deposit once again, with the 0 value. It would set → `_deposit.delegate = _primaryDelegate`
4. After, attacker make the withdraw `undelegateAmountFromAddress`, As we remember, `_deposit.delegate` is **Victim** now, so after this call contract `VoteCheckpoints` will have this state: `_delegates[msg.sender][delegatee] == 0`.
    
<img width="717" alt="Screenshot 2024-11-06 at 10 30 50" src="https://github.com/user-attachments/assets/a4c759b8-10d5-4674-827d-9d057bab9600">

# Issue explanation: token with LTV = 0, could abuse the users in the protocol

https://x.com/SpearbitDAO/status/1658556015762190340

**Core problem**:

The problem that the tokens with the LTV = 0 could be added to the account balance on the lending protocol. If the user has such kind of tokens, he can’t withdraw/transfer this tokens, but they still will affect the health factor of the account. So, what can be done is that the attacker send some funds to the victim address? The following will happen!

<img width="499" alt="Screenshot 2024-11-06 at 10 32 02" src="https://github.com/user-attachments/assets/a3dbf838-32e4-4aec-85f2-a8ea44dab4ff">


**How to find it next time**:

We always need to think in the way how to abuse the protocol or user, it is our main goal. During the zetaChain audit i haven’t thought about abusing the user, which forced me to miss some bugs! Here, we have a token which prevent some important function., and what if we could send to the balance of the user? → Bug, think in a way how to abuse the user.
