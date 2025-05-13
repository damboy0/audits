# Eggstravaganza - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Randomness Logic Used in searchForEgg Function Allows the Random Number to Be Predictable](#H-01)
- ## Medium Risk Findings
    - ### [M-01. Lack of cooldown Mechnism Allows user to call SearchEgg Multuple Times till they eventually win](#M-01)



# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #37

### Dates: Apr 3rd, 2025 - Apr 10th, 2025

[See more contest details here](https://codehawks.cyfrin.io/c/2025-04-eggstravaganza)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 1
- Medium: 1
- Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. Randomness Logic Used in searchForEgg Function Allows the Random Number to Be Predictable            



## Summary

The [`searchForEgg`](https://github.com/CodeHawks-Contests/2025-04-eggstravaganza/blob/f83ed7dff700c4319bdfd0dff796f74db5be4538/src/EggHuntGame.sol#L65-L81) function in the [`EggHuntGame`](https://github.com/CodeHawks-Contests/2025-04-eggstravaganza/blob/main/src/EggHuntGame.sol) contract attempts to introduce game-like randomness by using a pseudo-random number generator based on on-chain parameters. However, the chosen method is insecure, making the random number **predictable** and therefore, **exploitable** by malicious users or bots to increase their chances of finding eggs.

## Vulnerability Details

```solidity
uint256 random = uint256(
    keccak256(abi.encodePacked(block.timestamp, block.prevrandao, msg.sender, eggCounter))
) % 100;

```

The above line is responsible for generating a pseudo-random number. The randomness relies on the following parameters:

* `block.timestamp`: Can be influenced by miners/validators within a reasonable window (e.g., ±15 seconds).

* `block.prevrandao`: Introduced after Ethereum’s Merge to replace `block.difficulty`, but it’s **not cryptographically secure** and can be biased under certain conditions.

* `msg.sender`: Fully known by the caller.

* `eggCounter`: A public or easily inferred state variable.

Since all of these values can be **predicted off-chain**, an attacker can simulate the exact outcome of `searchForEgg()` before calling it. If the `random < eggFindThreshold` condition is not met, they simply skip the transaction. If the result is favorable, they proceed — giving themselves a **clear advantage** over honest users.

## Impact

* **Game Manipulation**: Bots or malicious players can **consistently win** more eggs than intended.

* **Unfair Advantage**: Legitimate players are at a **disadvantage** due to manipulation by those simulating the RNG.

## Tools Used

* **Manual Code Review**

## Recommendations

1. **Avoid using block variables for randomness in on-chain logic** unless the outcome has **no economic impact** or **can tolerate manipulation**.

2. Replace the random generation logic with [\*\*Chainlink VRF](https://docs.chain.link/vrf) (Verifiable Random Function)\*\*

    
# Medium Risk Findings

## <a id='M-01'></a>M-01. Lack of cooldown Mechnism Allows user to call SearchEgg Multuple Times till they eventually win            



## Summary

The [EggHuntGame](https://github.com/CodeHawks-Contests/2025-04-eggstravaganza/blob/main/src/EggHuntGame.sol) contract lacks a per-user cooldown mechanism for attempting to discover or mint eggs. This omission allows users to interact with the egg-finding functionality unrestricted, potentially leading to brute-force exploitation of the pseudo-randomness logic. The vulnerability undermines the fairness and integrity of the game, especially in a competitive or reward-based setting.

## Vulnerability Details

The core logic responsible for determining whether a player "finds" an egg in the [`searchForEgg`](https://github.com/CodeHawks-Contests/2025-04-eggstravaganza/blob/f83ed7dff700c4319bdfd0dff796f74db5be4538/src/EggHuntGame.sol#L65-L80) function is as follows:

```solidity
uint256 random = uint256(
    keccak256(abi.encodePacked(block.timestamp, block.prevrandao, msg.sender, eggCounter))
) % 100;

if (random < eggFindThreshold) {
    eggCounter++;
    eggsFound[msg.sender] += 1;
    eggNFT.mintEgg(msg.sender, eggCounter);
    emit EggFound(msg.sender, eggCounter, eggsFound[msg.sender]);
}

```

This function lacks any type of **rate limiting** or **time-based restriction** per user. Since the function can be called repeatedly in rapid succession, an attacker (or even a determined player) can spam the function within the same block or across multiple blocks to:

* Attempt multiple pseudo-random draws quickly

* Increase their odds of finding an egg

**This is because:**

* `block.timestamp` changes each block (but is predictable within a range)

* `eggCounter` increases deterministically after a successful find

* No gas penalty or restriction is stopping the user from calling this over and over

Players are incentivized to run aggressive scripts to find eggs quickly without a cooldown, bypassing the intended "chance" mechanic.

## Impact

* **Fairness Violation:** Users with scripting or botting capabilities gain an unfair advantage over honest players who interact normally.

* **Brute-force Exploitability:** Attackers can call the function repeatedly to force random values below `eggFindThreshold`, especially if they understand how to manipulate the inputs.

* **Denial of Service Risk:** If there’s a limited supply of eggs, a spammer could deplete the egg pool before others get a chance.

* **Gas Inefficiency and Network Strain:** Repeated calls with no restriction can bloat transactions and lead to high gas consumption.

## Tools Used

* **Manual Review**

## Recommendations

* **Implement a Per-User Cooldown Mechanism** Add a `lastEggAttempt` mapping to track the timestamp of the last interaction and allow the owner to tune the `cooldownTime` to adapt game dynamics if needed.




