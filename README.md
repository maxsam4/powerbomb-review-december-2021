# Powerbomb Finance Security Review

## Objective of the review

The review's focus is to verify that the smart contract system is secure, resilient, and working according to its specifications. The review activities can be grouped in the following three categories:

**Security**: Identifying security related issues within each contract and the system of contracts.

**Sound Architecture**: Evaluation of this system's architecture through the lens of established smart contract best practices and general software best practices.

**Code Correctness and Quality**: A full review of the contract source code.

This review is based on commit hash `8b6728a53a47ee115d97f8eb461f335c405c0933` of <https://github.com/Powerbomb-Finance/MVP>.

## Findings

During the review, 0 Critical, 3 Major, 4 Minor, and 11 Informational issues were found.

- A critical issue represents something that can be relatively easily exploited and will likely lead to loss of funds.
- A major issue represents something that can result in an unintended behavior of the smart contracts. These issues can also lead to loss of funds but are typically harder to execute than critical issues.
- A minor issue represents an oddity discovered during the review. These issues are typically situational or hard to exploit.
- An informational issue represents a potential improvement or thing of interest. These issues do not pose any practical security risks.

### Major

#### 1.1 Rewards are traded onchain without slippage protection

The protocol often converts the rewards earned by staking into other tokens. There is no slippage protection which can lead to loss of rewards due to sandwich attacks. However, the rewards are expected to be harvested frequently which will make it unprofitable to do a sandwich attack in normal conditions.

This shouldn't be a problem as long as harvests are done frequently and there is no sudden jump in rewards via an airdrop.

#### 1.2 Yield earned from AAVE is unfairly distributed

In `claimReward` of `PowerBombAvaxJoe.sol` , ibRewardTokenAmt is adjusted based on yield earned by the contract for everyone. The problem is that it isn't accounting for the fact that some users might have entered on day 0 vs others just now. People who have been in the contract longer should be owed a bigger share of the rewards earned till date but the function resets the rewards owed based on current balances every time the reward is harvested. This means, people who enter later get more rewards than they deserve and people who don't claim their rewards frequently can lose a portion of their rewards.

This issue is now fixed.

#### 1.3 Fee is uncapped

Owner can set `yieldFeePerc` and r`ewardWithdrawalFeePerc` to 100%, stealing all rewards from the users. Even worse, the owners can set the fee to greater than 100% which can make all withdrawal attempts revert and user funds will be held hostage. It is recommended to cap the `feePerc` to 20% or something similar.

This issue is now fixed, fee is capped to 20%.


### Minor

#### 2.1 Useless check

In `withdraw` function, there's a check that does not seem to be logically useful - `require(amountOutLpToken > 0 || user.lpTokenBalance >= amountOutLpToken, "Invalid amountOutLpToken to withdraw");`. Perhaps the `||` needs to be replaced with `&&`.

#### 2.2 Contract can be lead to believe it holds more money than it actually holds

The `getLpTokenPriceInAVAX` function of `PowerBombAvaxJoe.sol` can be manipulated to return high price by unbalancing the pool. The price is not used in anything critical but it's still recommended to use fair lp pricing - 
https://blog.alphafinance.io/fair-lp-token-pricing/

The price is only used to limit max TVL of the pool. By doing this manipulation, an attacker can make the pool think it has reached its TVL limit in which case new deposits will be rejected. Someone can do a sandwich attack using this attack to make new deposits fail. This is a Denial of Service attack but does not gain anything to the attacker. In fact, the attacker will lose significant money in fees for manipulating the pools.

#### 2.3 Hardcoded rewards for harvesting

Hardcoded `refundAmt` and min `_WAVAXAmt` in `harvest` function of `PowerBombAvaxJoe.sol` can become unsuitable if gas prices change drastically. This can make harvesting unprofitable for the harvester. However, the team can continue bearing the costs and keep doing harvests, if needed.

#### 2.4 SLP migration is not handled

Address of SLP is set as constant but it can change in masterchef (migration for LP to trident, for example). There should be a function that allows resyncing the SLP if that happens. Since the contracts are meant to be upgradable, this is not a major issue. The SLP address can be upgraded via a full contract upgrade.


### Informational

#### 3.1 Dead Code (some of them are already highlighted by devs in the code)
- https://github.com/Powerbomb-Finance/MVP/blob/04149616eaf8b98dfe718296b85eac253a67877c/hardhat/contracts/PowerBombAvaxCurve.sol#L87


#### 3.2 Possible arithmetic revert with no reason
- https://github.com/Powerbomb-Finance/MVP/blob/04149616eaf8b98dfe718296b85eac253a67877c/hardhat/contracts/PowerBombAvaxCurve33.sol#L294

#### 3.3 Lack of zero address check.
- Throughout, especially in initialized and setter functions.

#### 3.4 Unnecessary check

- `require(token == WAVAX, "Withdraw: only WAVAX");` is unnecessary in `withdrawAVAX` function of `PowerBombAvaxJoe.sol`.

#### 3.5 Magic constants

Magic numbers and addresses like `0xB674f93952F02F2538214D4572Aa47F262e990Ff` should be stored as constants.

#### 3.6 Use of tx.origin is discouraged

Instead of transferring refund money to `tx.origin` in of `PowerBombAvaxJoe.sol`, consider using `msg.sender` and then `msg.sender` can decide if it wants to forward the cash. Alternatively, make refund address a user input.

#### 3.7 Gas Savings

`uint gOHMAmt = token0.balanceOf(address(this));` in `harvest` can be moved to the if statement for gas savings. i.e. execute the logic only if the lp address matches.

#### 3.8 Upgradability can be bug

The contracts are meant to be upgradable. This implies that the team has the power to change the logic post launch. It allows the team to fix bug but can also lead to addition of bugs like it happened with Compound.

#### 3.9 Code duplicacy

There is a lot of code duplicacy among different vaults. It's recommended to create a base contract with the common logic and inherit it  in specific contracts.

#### 3.10 Code Simplification

Instead of taking btcSplit and ethSplit as inputs, just directly take btcAmount and ethAmount.

#### 3.11 Fees can be combined

Currently, there is a `yieldFeePerc` and `rewardWithdrawalFeePerc` but they are logically the same except for the time they are charged. They can be combined together for simplification.


## Disclaimer

This report is not an endorsement or indictment of any particular project or team, and the report does not guarantee the security of any particular project. This report does not consider, and should not be interpreted as considering or having any bearing on, the potential economics of a token, token sale or any other product, service or other asset. Cryptographic tokens are emergent technologies and carry with them high levels of technical risk and uncertainty. This report does not provide any warranty or representation to any Third-Party in any respect, including regarding the bugfree nature of code, the business model or proprietors of any such business model, and the legal compliance of any such business. No third party should rely on this report in any way, including for the purpose of making any decisions to buy or sell any token, product, service or other asset. Specifically, for the avoidance of doubt, this report does not constitute investment advice, is not intended to be relied upon as investment advice, is not an endorsement of this project or team, and it is not a guarantee as to the absolute security of the project. I owe no duty to any Third-Party by virtue of publishing these Reports.

The scope of my review is limited to a review of Solidity code and only the Solidity code noted as being within the scope of the review within this report. The Solidity language itself remains under development and is subject to unknown risks and flaws. The review does not extend to the compiler layer, or any other areas beyond Solidity that could present security risks. Cryptographic tokens are emergent technologies and carry with them high levels of technical risk and uncertainty.

This review does not give any warranties on finding all possible security issues of the given smart contracts, i.e., the evaluation result does not guarantee the nonexistence of any further findings of security issues. As one review cannot be considered comprehensive, I always recommend proceeding with several independent reviews and a public bug bounty program to ensure the security of smart contracts.
