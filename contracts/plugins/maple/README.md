# Maple Pool Collateral (FDT)

## Summary

> contracts/plugins/maple/

[Maple Docs](https://maplefinance.gitbook.io/maple/)<br>
[Technical Docs](https://github.com/maple-labs/maple-core/wiki/Background)

Maple is a decentralized corporate credit market. Maple provides capital to institutional borrowers through globally accessible fixed-income yield opportunities.

In short, it allows normal users to lend money to corporate entities.

Liquidity Providers (LPs) deposit funds into a Liquidity Pool in order to fund loans and earn yield. In return, they receive an LP token representing their share of the pool.

This LP token (also called FDT) is our tok.

Specifically, I am using Maple's [M11 USDC Credit Pool](https://app.maple.finance/#/earn/pool/0x6f6c8013f639979c84b756c7fc1500eb5af18dc4).

## Implementation

|  `tok`  | `ref` | `tgt` | `UoA` |
| :-----: | :---: | :---: | :---: |
| PoolFDT | USDC  |  USD  |  USD  |

The maple pool contract basically stores the liquidity (USDC) provided to it in a LiquidityLocker contract. Then counts the borrowed amount in the `principalOut` storage variable, if it makes profit through interest it stores that in `interestSum` storage var and any losses in the `poolLosses` storage var, in the Pool contract.

The `interestSum` and `poolLosses` variables are also updated in case of an withdrawal by a user (as he withdraws his share of the profit or takes his share of the loss during the withdrawal).

The pool may concur a loss if any of the loans given out by the pool defaults. In such a case, if the collateral provided by the defaulted borrower and the [pool cover by BPT stakers](https://github.com/maple-labs/maple-core/wiki/Pool-Cover) is not enough to cover the loss, this loss is passed on to the LPs. This is the situation where we might think about changing our collateral status to DISABLED or maybe IFFY.

The FDT tokens owned by the user are not rebasing and stay constant in case of any profit or loss of the protocol.

For more details of the Maple Pool accounting you can refer [here](https://github.com/maple-labs/maple-core/wiki/Pool-Accounting).

### refPerTok

So refPerTok here will be simply the total usdc owned by the contract including the usdc sent out to loans divided the total supply of the fdt tokens. (Since in case of a profit, the usdc owned increases, and in case of a loss the `principalOut` or the usdc sent out to loans decreases)

### refresh

We would normally want the collateral status to be DISABLED when the pool incurs a big default on one of its loans.

But what if the pool has been making huge profits on interest and one day just makes a small loss? Would we want to default then? No. We may want to change the collateral status to IFFY though (maybe)

The `poolLosses` and `interestSum` are moving variables and keep on changing not only on profits and loss but also on user withdraws. So if at any time `poolLosses` > `interestSum`, the net profits of the pool will be negative and we would want to DISABLE in such a scenario.

But we dont need to track the poolLosses and interestSum for that. According to the maple pool accounting <img width="703" alt="Screenshot 2022-11-08 at 6 04 36 PM" src="https://user-images.githubusercontent.com/47485188/200564974-88fb4943-5488-4e6f-8761-35cc6478f13b.png">
So we can just keep track of the liquidityLockerBal+principalOut or in short we can keep track of the refPerTok.

if poolLosses > interestSum => refPerTok will be less than 1 (or 1e18 in our case). So in such an scenario we would directly want to DISABLE.

But in case of a small loss, where the **poolLosses < interestSum** and **refPerTok > 1e18**, we can introduce a threshold.

So suppose if the refPerTok is above 1 but below our threshold (suppose 1.01) then we can change the status to IFFY. If its above 1.01 then even after a loss we can be pretty sure that its a small loss, not much to worry about and we can be SOUND.

Including that I have also added priceFeed checks to mark the status as IFFY if the ref token **USDC** depegs or if the priceFeed stops working.

## Notes

### prevRefPerTok > currerntRefPerTok

One another thing is in maple specially we cannot mark status as DISABLED if current refPerTok is less than previous refPerTok.

- because in the withdrawfunds methods the MaplePool withdraws the interest to the user
- but doesn't burn the user's FDT tokens
- so the ref tokens decrease with the FDT tokens being constant
- this will reduce the refPerTok Value
- even though the protocol has not suffered any loss

### MPL REWARDS

Including interest from lending the lenders also earn MPL rewards.
As per the documentation, we dont have to call any method for earning rewards.
The rewards will be deposited directly to the lender's account at the end of each month.
https://maplefinance.gitbook.io/maple/protocol/maple-token-holders/how-can-you-earn-mpl

### [Excel Sheet](https://docs.google.com/spreadsheets/d/16rY9nHSd32Rf3FKmVDE3ipDHFho17-eb9SVjwxe31Ag/edit?usp=sharing)

This one is mostly rough calculations that I used to review the maple pool accounting. May come of some use so I will just keep it here.

## Tests

Detailed mainnet fork tests at

> test/plugins/maple/MapleCollateral.fork.test.ts
