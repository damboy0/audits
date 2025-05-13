# First Flight #3: Thunder Loan - Findings Report

# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
  - ### [H-01. ThunderLoan flashloan function can be manipulated and used to extract pool funds.](#H-01)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #3

### Dates: Nov 1st, 2023 - Nov 8th, 2023

[See more contest details here](https://www.codehawks.com/contests/clocopz26004rkx08q1n61wnz)

# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 1
- Medium: 0
- Low: 0

# High Risk Findings

## <a id='H-01'></a>H-01. ThunderLoan flashloan function can be manipulated and used to extract pool funds.

## Vulnerability

In the `ThunderLoan.sol` contract the `FlashLoan()` function can be manipulated since it does a check that requires the balance of the pool be the greater than the borrowed amount + fees.

## Impact

Calling `flashLoan()` function and `deposit()` can be exploited to drain the pool.

## Proof of Concept

In `ThunderLoan.flashloan` we check that the balance of the pool after executing operation is greater than the balance of the initial borrowed amount.

```sh
        uint256 endingBalance = token.balanceOf(address(assetToken));
        if (endingBalance < startingBalance + fee) {
            revert ThunderLoan__NotPaidBack(
                startingBalance + fee,
                endingBalance
            );
        }
```

With this an attacker can borrow all the funds in the pool by using `flashloan()` then in the `execute operation` the attacker deposits back the borrowed funds + fees using the contract's `deposit()` function, the attacker then gets minted reciept token in accordance to the depost function's logic:

```sh
assetToken.mint(msg.sender, mintAmount);
```

Then becomes the highest Lp provider. After wards the attacker can call the `redeem()` function and since the redeem function requires that the caller has the reciept tokens before releasing the equivalent amount of tokens to the caller. The attacker will pass this check completely and drain the pool.

To simulate the attack, add the provided codes in the gist file to the `2021-11-ThunderLoan` codebase as follows:

- in the `/test/mocks` create a file and paste the code [here](https://gist.github.com/DevPelz/357419f3c8625b6663a5665b4aeaf049), this is the flashLoan reciever.

- then in the `/test/unit` create another file and paste the code [here](https://gist.github.com/DevPelz/ba24abc6ab60756567533a1d746a904a), this is the test to run the attack.

Finally you can run the tests using `forge t` and confirm the exploit.

## Recommended Mitigation Steps

Use the same check used in the `repay()` function to lock the `deposit()` when flasloan is active:

```
if (!s_currentlyFlashLoaning[token]) {
     revert ThunderLoan__CurrentlyFlashLoaning();
}
```

The above could be added and the `deposit()` would reevert if flashloan is active.
