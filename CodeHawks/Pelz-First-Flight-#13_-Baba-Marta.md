# First Flight #13: Baba Marta - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Incorrect Calculation of Collected Rewards in collectReward Function](#H-01)
    - ### [H-02. Lack of Excess Ether Refund Mechanism in buyMartenitsa Function](#H-02)
- ## Medium Risk Findings
    - ### [M-01. Unauthorized Reward Claiming by Producers via Token Transfers](#M-01)
    - ### [M-02. Lack of Delisting Mechanism for NFTs in Marketplace](#M-02)
    - ### [M-03. Incorrect Winner Announcement Handling in announceWinner Function](#M-03)
    - ### [M-04.  Gas Inefficiency and Potential Out-of-Gas Error in voteForMartenitsa Function](#M-04)
- ## Low Risk Findings
    - ### [L-01. Lack of Listing Check in makePresent Function](#L-01)
    - ### [L-02. Unnecessary onlyOwner Modifier in Constructor Function](#L-02)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #13

### Dates: Apr 11th, 2024 - Apr 18th, 2024

[See more contest details here](https://www.codehawks.com/contests/cluseb1bf0001s4tjl2rzajup)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 2
   - Medium: 4
   - Low: 2


# High Risk Findings

## <a id='H-01'></a>H-01. Incorrect Calculation of Collected Rewards in collectReward Function            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-04-Baba-Marta/blob/5eaab7b51774d1083b926bf5ef116732c5a35cfd/src/MartenitsaMarketplace.sol#L106

## Summary

The `collectReward` function in the `MartenitsaMarketplace.sol` contract enables users to claim rewards based on their token ownership. However, a vulnerability exists in the current implementation, allowing users to potentially receive more rewards than intended by claiming them incrementally. This vulnerability stems from the incorrect handling of previously claimed rewards, leading to an inflation of rewards and imbalance in the ecosystem.

## Vulnerability Details

The vulnerability arises due to the flawed calculation of rewards in the `collectReward` function. Users are entitled to one reward token for every three tokens they own. However, the function fails to accurately track previously claimed rewards, resulting in users potentially receiving excessive rewards when claiming them incrementally.

For example, if a user owns 12 tokens and claims rewards incrementally, they should receive 4 reward tokens in total. However, due to the oversight, they may receive 6 reward tokens instead, resulting in an unintended increase in the reward distribution.

A proof of concept demonstrating this vulnerability is available in the following gist: [Proof of Concept Gist](https://gist.github.com/DevPelz/46c5966db15c43e04e173360fd9f8977). It should be added to the MartenitsaMArketplace.t.sol file.

## Impact

This vulnerability enables users to exploit the reward system, potentially leading to an inflation of rewards and disrupting the ecosystem's balance. It may result in an increased circulating supply of reward tokens, affecting their value and utility within the platform. Moreover, it undermines the fairness and integrity of the reward distribution mechanism.

## Tools Used

manual code review.

## Recommendations

To mitigate this vulnerability and ensure the accurate distribution of rewards, it is imperative to implement proper tracking of previously claimed rewards. Adjust the `collectReward` function as follows:

```solidity
function collectReward() external {
  require(
    !martenitsaToken.isProducer(msg.sender),
    "You are a producer and not eligible for a reward!"
  );

  uint256 count = martenitsaToken.getCountMartenitsaTokensOwner(msg.sender);
  uint256 totalRewards = count / requiredMartenitsaTokens;
  uint256 newRewards = totalRewards - _collectedRewards[msg.sender]; // Calculate new rewards

  if (newRewards > 0) {
    _collectedRewards[msg.sender] += newRewards; // Increment previously claimed rewards
    healthToken.distributeHealthToken(msg.sender, newRewards); // Distribute new rewards
  }
}
```

With this fix, the `collectReward` function accurately tracks previously claimed rewards and ensures users receive rewards based on their total token ownership. This adjustment promotes fairness and consistency in the reward distribution process.

## <a id='H-02'></a>H-02. Lack of Excess Ether Refund Mechanism in buyMartenitsa Function            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-04-Baba-Marta/blob/5eaab7b51774d1083b926bf5ef116732c5a35cfd/src/MartenitsaMarketplace.sol#L63

## Summary
The `buyMartenitsa` function in the `MartenitsaMarketplace.sol` contract is vulnerable to a potential loss of funds due to an oversight in handling excess Ether sent by users.

## Vulnerability Details

In the `buyMartenitsa` function, there is a requirement that ensures the amount of Ether sent (`msg.value`) is greater than or equal to the listing price. 
` require(msg.value >= listing.price, "Insufficient funds");`
However, if a user accidentally sends more Ether than required, there is no mechanism in place to refund the excess Ether. As a result, the excess Ether will remain locked in the contract indefinitely, leading to a loss of funds for users.

## Impact

This vulnerability can lead to a loss of funds for users who accidentally send more Ether than required when purchasing a Martenitsa token. Over time, this accumulation of locked funds could have a negative impact on user trust and the reputation of the marketplace contract.

## Tools Used
manual code review.

## Recommendations

To address this vulnerability and ensure the safety of user funds, the following fix is recommended:

 **Implement a Refund Mechanism**: Modify the `buyMartenitsa` function to include a mechanism for refunding excess Ether to the sender. After deducting the required amount for the purchase, any excess Ether should be returned to the sender's address.

   Example:
   ```solidity
   // Calculate excess Ether
   uint256 excessAmount = msg.value - listing.price;

   // Refund excess Ether to sender
   if (excessAmount > 0) {
       (bool refunded, ) = msg.sender.call{value: excessAmount}("");
       require(refunded, "Failed to refund excess Ether");
   }
   ```

		
# Medium Risk Findings

## <a id='M-01'></a>M-01. Unauthorized Reward Claiming by Producers via Token Transfers            



## Summary

The `collectReward` function in the `MartenitsaMarketplace.sol` contract has a vulnerability where producers can bypass the eligibility check and claim rewards by transferring their already created tokens to another address they control. This circumvention undermines the intended restriction on producers claiming rewards and poses a risk to the fairness of the reward distribution system.

## Vulnerability Details

The vulnerability arises from the inadequate enforcement of eligibility criteria in the `collectReward` function. While the function includes a check to ensure that producers are not eligible for rewards, this check can be bypassed by producers transferring their tokens to other addresses they control. By transferring tokens to eligible addresses, producers can effectively circumvent the eligibility check and claim rewards improperly.

Proof of Concept (POC):

```solidity
// Producer is eligible for reward after creating 3 martenitsas
vm.startPrank(chasy);
martenitsaToken.createMartenitsa("bracelet");
martenitsaToken.createMartenitsa("bracelet");
martenitsaToken.createMartenitsa("bracelet");
martenitsaToken.approve(address(marketplace), 0);
martenitsaToken.approve(address(marketplace), 1);
martenitsaToken.approve(address(marketplace), 2);
marketplace.makePresent(bob, 0);
marketplace.makePresent(bob, 1);
marketplace.makePresent(bob, 2);
vm.stopPrank();

vm.startPrank(bob);
// Collect reward
marketplace.collectReward();
// Transfer reward back to chasy (producer)
healthToken.transfer(chasy, 10 ** 18);
vm.stopPrank();
assert(healthToken.balanceOf(chasy) == 10 ** 18);
```

The provided POC demonstrates how a producer (`chasy`) can create three Martenitsas and transfer them to another address (`bob`). Subsequently, `bob` can collect the rewards and transfer them back to `chasy`, effectively bypassing the restriction on producers claiming rewards.

## Impact

This vulnerability undermines the fairness and integrity of the reward distribution system in the marketplace. Producers can exploit this loophole to claim rewards improperly, potentially leading to unfair advantages and distortions in the distribution of rewards. Additionally, it may erode user trust in the platform and diminish the perceived value of rewards.

## Tools Used

manual code review.

## Recommendations

To mitigate this vulnerability, consider implementing stricter controls to prevent producers from transferring their tokens to other addresses for the purpose of claiming rewards. Additionally, review the reward distribution mechanism to ensure that rewards are distributed fairly and in accordance with the intended criteria.

One potential fix could be to restrict producers from making presents altogether, allowing only those who purchase NFTs to be eligible for rewards. This would help prevent producers from exploiting the reward system by transferring tokens to other addresses they control. However, it's important to consider the impact of such restrictions on the overall functionality and user experience of the marketplace.
## <a id='M-02'></a>M-02. Lack of Delisting Mechanism for NFTs in Marketplace            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-04-Baba-Marta/blob/5eaab7b51774d1083b926bf5ef116732c5a35cfd/src/MartenitsaMarketplace.sol#L49

## Summary

The `listMartenitsaForSale` function in the `MartenitsaMarketplace.sol` contract lacks a mechanism for users to delist their NFTs (non-fungible tokens) by setting the `forSale` flag to `false`. Currently, the function hardcodes the `forSale` flag to `true` when listing an NFT for sale, and there is no explicit way for users to remove their NFTs from the marketplace.

## Vulnerability Details

The vulnerability arises from the absence of a dedicated function for users to delist their NFTs. Once an NFT is listed for sale using the `listMartenitsaForSale` function, it remains listed indefinitely, with no option for the owner to remove it from the marketplace. This lack of control over listings can lead to a cluttered marketplace and may result in an undesirable user experience.

## Impact

The impact of this vulnerability is primarily on user experience and marketplace management. Without the ability to delist NFTs, users may have limited control over their listings, leading to potential frustration and confusion. Additionally, a cluttered marketplace with outdated or unwanted listings can diminish the overall usability and attractiveness of the platform.

## Tools Used

manual code review.

## Recommendations

To address this vulnerability and enhance user control over their listings, it is recommended to implement a separate function that allows users to delist their NFTs from the marketplace. This function should update the `forSale` flag to `false` and remove the NFT from the `tokenIdToListing` mapping. Additionally, consider implementing access controls to ensure that only the owner of the NFT can delist it.

Example Fix:

```solidity
function delistMartenitsa(uint256 tokenId) external {
  require(
    msg.sender == martenitsaToken.ownerOf(tokenId),
    "You do not own this token"
  );

  // Set the forSale flag to false
  tokenIdToListing[tokenId].forSale = false;

  // Remove the listing from the mapping
  delete tokenIdToListing[tokenId];

  emit MartenitsaDelisted(tokenId, msg.sender);
}
```

By implementing this fix, users regain control over their listings and can remove their NFTs from the marketplace as needed, enhancing the overall usability and user experience of the platform.

## <a id='M-03'></a>M-03. Incorrect Winner Announcement Handling in announceWinner Function            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-04-Baba-Marta/blob/main/src/MartenitsaVoting.sol#L57-L74

## Summary

The `announceWinner` function in the `MartenitsaVoting.sol` contract does not handle tie scenarios properly. In cases where multiple tokens receive the same maximum number of votes, only the first token encountered is announced as the winner, neglecting other tokens with the same vote count.

## Vulnerability Details

The vulnerability lies in the logic of the `announceWinner` function, which does not handle tie scenarios properly. In cases where multiple tokens receive the same maximum number of votes, only the first token encountered is announced as the winner, neglecting other tokens with the same vote count.

Proof of Concept (POC) to be added to MartenitsaVoting.t.sol:

```solidity
function testVoteForMartenitsaTwice() public listMartenitsa {
  vm.startPrank(chasy);
  martenitsaToken.createMartenitsa("bracelet2");
  marketplace.listMartenitsaForSale(1, 1 wei);
  vm.stopPrank();

  vm.prank(address(uint160(1)));
  voting.voteForMartenitsa(0);

  vm.prank(address(uint160(2)));
  voting.voteForMartenitsa(1);

  assert(voting.voteCounts(0) == 1);
  assert(voting.voteCounts(1) == 1);
}

function testAnnounceWinnerWrongly() public {
  testVoteForMartenitsaTwice();
  vm.warp(block.timestamp + 1 days + 1);
  vm.recordLogs();
  voting.announceWinner();

  Vm.Log[] memory entries = vm.getRecordedLogs();
  address winner = address(uint160(uint256(entries[0].topics[2])));
  assert(winner == chasy);
}
```

## Impact

This vulnerability can lead to incorrect determination of winners in tie scenarios, resulting in unfair outcomes and potentially undermining the integrity and credibility of the voting process. It may also impact user trust in the platform and the reliability of voting results.

## Tools Used

manual code review.

## Recommendations

To address this vulnerability and ensure fair determination of winners in tie scenarios, it is recommended to modify the `announceWinner` function to handle tie situations appropriately. Consider implementing logic to identify and announce all tied tokens as winners if they share the maximum vote count.

Example:

```solidity
function announceWinner() external onlyOwner {
  require(block.timestamp >= startVoteTime + duration, "The voting is active");

  uint256 maxVotes = 0;
  address[] memory winners;

  // Find the maximum number of votes
  for (uint256 i = 0; i < _tokenIds.length; i++) {
    if (voteCounts[_tokenIds[i]] > maxVotes) {
      maxVotes = voteCounts[_tokenIds[i]];
    }
  }

  // Identify all tokens with the maximum vote count as winners
  for (uint256 i = 0; i < _tokenIds.length; i++) {
    if (voteCounts[_tokenIds[i]] == maxVotes) {
      winners.push(_tokenIds[i]);
    }
  }

  // Distribute rewards to all winners
  for (uint256 i = 0; i < winners.length; i++) {
    list = _martenitsaMarketplace.getListing(winners[i]);
    _healthToken.distributeHealthToken(list.seller, 1);
    emit WinnerAnnounced(winners[i], list.seller);
  }
}
```

## <a id='M-04'></a>M-04.  Gas Inefficiency and Potential Out-of-Gas Error in voteForMartenitsa Function            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-04-Baba-Marta/blob/5eaab7b51774d1083b926bf5ef116732c5a35cfd/src/MartenitsaVoting.sol#L51

## Summary

The `voteForMartenitsa` function in the `MartenitsaVoting.sol` contract allows users to vote for a particular Martenitsa token. However, a potential vulnerability exists in the continuous pushing of token IDs into an array after each vote. This approach could lead to an out-of-gas error if the array grows too large, especially in scenarios where a large number of votes are cast for the same token ID consecutively.

## Vulnerability Details

The vulnerability lies in the continuous appending of token IDs to the `_tokenIds` array within the `voteForMartenitsa` function. Each time a user votes, the token ID is added to the array, potentially causing it to grow significantly over time. In scenarios where a large number of votes are cast for the same token ID in quick succession, the array could become excessively large, leading to an out-of-gas error during transaction execution.

This vulnerability could disrupt the voting process and affect the fairness and accuracy of the voting mechanism, potentially impacting the announcement of winners and overall system functionality.

## Impact

The impact of this vulnerability could be significant, potentially resulting in out-of-gas errors during transaction execution, disruption of the voting process, and inaccuracies in determining the winners of the voting process. It could undermine the integrity of the voting system and erode user trust in the platform.

## Tools Used

manual code review.

## Recommendations

To mitigate this vulnerability and ensure the stability and efficiency of the voting mechanism, it is recommended to revise the approach for storing token IDs. Consider alternative data structures or storage mechanisms that can handle large amounts of data without impacting gas costs significantly. Additionally, implementing gas-efficient coding practices and optimizing data storage can help prevent out-of-gas errors and ensure the smooth operation of the voting system.


# Low Risk Findings

## <a id='L-01'></a>L-01. Lack of Listing Check in makePresent Function            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-04-Baba-Marta/blob/main/src/MartenitsaMarketplace.sol#L90-L95

## Summary

The `makePresent` function in the `MartenitsaMarketplace.sol` contract allows token owners to gift tokens to others without checking if the token is already listed for sale. This oversight can disrupt the logic of the codebase and prevent the purchase of listed tokens. 

## Vulnerability Details

The `makePresent` function permits token owners to transfer tokens to others without verifying if the token is already listed for sale in the marketplace. Consequently, an owner can inadvertently gift a token that is listed, which could disrupt the marketplace's functionality and prevent potential buyers from purchasing the listed token.

Proof of the vulnerability is demonstrated in the following test case:
to be added to the : MartenitsaMarketplace.t.sol file.
```solidity
function testMakePresentOfListedItemsAndBuyFails() public {
    // List token for sale
    testListMartenitsaForSale();

    // Gift token
    vm.startPrank(chasy);
    martenitsaToken.approve(address(marketplace), 0);
    marketplace.makePresent(jack, 0);
    vm.stopPrank();

    // Assertions to verify the outcome of the gift transaction
    assert(martenitsaToken.ownerOf(0) == jack);
    assert(martenitsaToken.getCountMartenitsaTokensOwner(bob) == 0);
    assert(martenitsaToken.getCountMartenitsaTokensOwner(jack) == 1);

    // Check if listing still exists
    marketplace.getListing(0);

    // Attempt to buy the token and expect failure
    vm.expectRevert();
    vm.prank(bob);
    marketplace.buyMartenitsa{value: 1 wei}(0);
}
```

This test case demonstrates that the `makePresent` function successfully transfers a listed token (`tokenId = 0`) to another address (`jack`). Subsequently, an attempt to purchase the token (`tokenId = 0`) fails, indicating that the token remains listed for sale despite being gifted.


## Impact

This vulnerability could lead to disruption in the marketplace's functionality, as listed tokens could be inadvertently gifted, rendering them unavailable for purchase. As a result, buyers would be unable to acquire tokens that were intended for sale, potentially leading to frustration and loss of trust in the marketplace.

## Tools Used
 manual code review.

## Recommendations

To mitigate this vulnerability, it is recommended to implement a check to verify whether the token is listed for sale before allowing it to be gifted. The following fix is proposed:

 **Check Listing Status**: Modify the `makePresent` function to include a check to verify if the token is listed for sale before allowing it to be transferred as a gift. If the token is listed, the function should revert to prevent unintended disruption of the marketplace's logic.

   Example:
   ```solidity
   function makePresent(address presentReceiver, uint256 tokenId) external {
       require(msg.sender == martenitsaToken.ownerOf(tokenId), "You do not own this token");
       require(!tokenIdToListing[tokenId].forSale, "Token is listed for sale");
       
       // Rest of the function's logic...
   }
   ```



## <a id='L-02'></a>L-02. Unnecessary onlyOwner Modifier in Constructor Function            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-04-Baba-Marta/blob/5eaab7b51774d1083b926bf5ef116732c5a35cfd/src/MartenitsaEvent.sol#L22


## Summary

The constructor of the `MartenitsaEvent.sol` contract utilizes the `onlyOwner` modifier, which restricts its execution to the contract owner. However, this restriction appears unnecessary for a constructor function.

## Vulnerability Details

The use of the `onlyOwner` modifier in the constructor function imposes an unnecessary constraint, as constructor functions are automatically executed by the deploying address, which is typically the contract owner. The `onlyOwner` modifier is commonly employed in functions that require privileged access control, but applying it to a constructor adds no additional security or functionality.

## Impact

While this redundancy does not introduce a security vulnerability per se, it may contribute to code complexity and maintenance overhead. Unnecessary modifiers can obscure the intent of the code and increase the risk of errors during contract deployment or upgrades.

## Tools Used

manual code review.

## Recommendations

To streamline the contract and improve readability, it is advisable to remove the `onlyOwner` modifier from the constructor function. This action simplifies the codebase without compromising security or functionality.

Example:

```solidity
constructor(address healthToken) {
  _healthToken = HealthToken(healthToken);
}
```

By removing the unnecessary modifier, the constructor becomes more concise and aligns with common coding practices.



