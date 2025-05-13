# First Flight #5: Santa's List - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. anyone can buy present using an address with santatokens as input](#H-01)
    - ### [H-02. Users can collect present more than once in SantasList.collectPresent()](#H-02)
    - ### [H-03. Anyone can call SantasList.CheckList due to missing implementation of onlySanta modifier](#H-03)

- ## Low Risk Findings
    - ### [L-01. Using `_mintAndIncrement()` in SantasList.buyPresent sends tokens to msg.sender not to present reciever](#L-01)
    - ### [L-02. Unused Variable in SantaList.sol](#L-02)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #5

### Dates: Nov 30th, 2023 - Dec 7th, 2023

[See more contest details here](https://www.codehawks.com/contests/clpba0ama0001ywpabex01hrp)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 3
   - Medium: 0
   - Low: 2


# High Risk Findings

## <a id='H-01'></a>H-01. anyone can buy present using an address with santatokens as input            



## Summary
The vulnerability lies in the` SantasList.buyPresent` function, where an attacker can wrongfully burn tokens from the `presentReceiver` and mint additional presents. This manipulation leads to an unauthorized transfer of presents, impacting user balances and compromising the integrity of the contract's present purchase mechanism

## Vulnerability Details
In the vulnerable `buyPresent` function, an attacker can exploit it by leveraging an extra nice user's address who has tokens to initiate the purchase of a present through the `buyPresent` function. This results in the burning of tokens from the unsuspecting user, followed by the unauthorized minting of additional presents. As a consequence, the attacker gains control over the present purchase mechanism, leading to a misalignment of user balances and compromising the intended functionality of the contract.

## Impact
The identified vulnerability poses a significant impact on the smart contract's functionality and security. By exploiting the flaw in the `buyPresent` function, an attacker can wrongfully burn tokens from the intended recipient `(presentReceiver)` and initiate the unauthorized minting of additional presents. 

## POC

In the test file add this to the state variable :
`address reciever = makeAddr("reciever");`
Then add this function to test the exploit:
```  
        function testAttackBuyPresent() public {
        vm.startPrank(santa);
        santasList.checkList(user, SantasList.Status.EXTRA_NICE);
        santasList.checkTwice(user, SantasList.Status.EXTRA_NICE);
        vm.stopPrank();

        vm.warp(santasList.CHRISTMAS_2023_BLOCK_TIME() + 1);

        // an extra nice user collects present
        vm.startPrank(user);
        santasList.collectPresent();
        vm.stopPrank();

        // buy present using extra nice user address
        vm.startPrank(reciever);
        santasList.buyPresent(user);
        vm.stopPrank();

        assertEq(santasList.balanceOf(user), 1);
        assertEq(santaToken.balanceOf(user), 0);

        assertEq(santasList.balanceOf(reciever), 1);
    }
```
run this command to test only this function and observe the exploit
`forge t --mt testAttackBuyPresent -vvvvv`

## Tools Used
Foundry & Manual Review

## Recommendations
In the implementation of the [i_santaToken](https://github.com/Cyfrin/2023-11-Santas-List/blob/6627a6387adab89ae2ba2e82b38296723261c08a/src/SantasList.sol#L173) burn function the `presentReciever` could be changed to `msg.sender` so the burn function burns tokens from whoever calls the function
## <a id='H-02'></a>H-02. Users can collect present more than once in SantasList.collectPresent()            



## Summary
The `collectPresent` function in the provided code introduces a vulnerability in the smart contract Which allows users to be able to collect presents more than once.

## Vulnerability Details
Any user that has been checked twice and eligible to collect present can collect the present and transfer to another address then collect again because of the check on line [151](https://github.com/Cyfrin/2023-11-Santas-List/blob/6627a6387adab89ae2ba2e82b38296723261c08a/src/SantasList.sol#L151).

## Impact
The vulnerability exposes a flaw in the contract's conditional checks, enabling an attacker to undermine the intended security and functionality of the collectPresent function.

## POC
in the test file add this line of code as a state variable:
` address reciever = makeAddr("reciever");`
then add this block of code to the file:
```
  function testAttackCollectPresent() public {
        vm.startPrank(santa);
        santasList.checkList(user, SantasList.Status.EXTRA_NICE);
        santasList.checkTwice(user, SantasList.Status.EXTRA_NICE);
        vm.stopPrank();

        vm.warp(santasList.CHRISTMAS_2023_BLOCK_TIME() + 1);

        vm.startPrank(user);
        santasList.collectPresent();
        // after collecting present transfer to another address
        santasList.transferFrom(user, reciever, 0);

        // collect present again
        santasList.collectPresent();

        assertEq(santasList.balanceOf(user), 1);
        assertEq(santasList.balanceOf(reciever), 1);
        assertEq(santaToken.balanceOf(user), 2e18);
    }
```
after that run this command to confirm the exploit
`forge t --mt testAttackCollectPresent`

## Tools Used
Foundry & Manual Review

## Recommendations
A mapping could be implemented to mitigate it, say using a mapping of `address => bool` and calling it `hasClaimed` so if a user has passed all the checks in the function has claimed can be set to true then a require statement would be needed to check if the user has claimed before or not.
## <a id='H-03'></a>H-03. Anyone can call SantasList.CheckList due to missing implementation of onlySanta modifier            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Santas-List/blob/6627a6387adab89ae2ba2e82b38296723261c08a/src/SantasList.sol#L121

## Summary
The discovered security issue allows unauthorized users to access the `SantasList.checklist` function and assign a status to any address of their choice. This creates a serious danger because it allows unauthenticated users to modify the status assignments, potentially resulting in illegal access or harmful activity within the contract.

## Vulnerability Details
 Any user, without proper authentication, can call this function and assign a chosen address any status, presenting a risk of unauthorized access and potential misuse. at [Checklist()](https://github.com/Cyfrin/2023-11-Santas-List/blob/6627a6387adab89ae2ba2e82b38296723261c08a/src/SantasList.sol#L121) function

## Impact
The contract is susceptible to unauthorized manipulation through the checklist function. 

## POC
add the following block of code to the test file:
```
  function testAttackCheckList() public {
        vm.prank(user);
        santasList.checkList(user, SantasList.Status.EXTRA_NICE);
        assertEq(
            uint256(santasList.getNaughtyOrNiceOnce(user)),
            uint256(SantasList.Status.EXTRA_NICE)
        );
    }
```
then run the command below to check it out:
`forge t --mt testAttackCheckList`
## Tools Used
Foundry

## Recommendations
In the function, including the onlySanta modifier could solve the issue:

```
  function checkList(address person, Status status) external onlySanta{
        s_theListCheckedOnce[person] = status;
        emit CheckedOnce(person, status);
    }

```
		


# Low Risk Findings

## <a id='L-01'></a>L-01. Using `_mintAndIncrement()` in SantasList.buyPresent sends tokens to msg.sender not to present reciever            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Santas-List/blob/6627a6387adab89ae2ba2e82b38296723261c08a/src/SantasList.sol#L174

## Summary
The vulnerability in the `buyPresent` function is rooted in the misalignment of token transfers. The use of `_mintAndIncrement()` mistakenly sends tokens to the function invoker `(msg.sender)` instead of the intended recipient `(presentReceiver)`.

## Impact
The impact of this vulnerability is significant, as it introduces the risk of unauthorized token transfers and disrupts the expected behavior of the present purchase mechanism. 

## Tools Used
Manual Review

## Recomendations
Using `mintAndIncrement()` in the function should be changed and be dynamic in the sense that it should accept a parameter of address to be minted to, say:
```
    function buyPresent(address presentReceiver) external {
        i_santaToken.burn(presentReceiver);
        _mintAndIncrement(presentReciever);
    }
```
So the original function could be tweaked to accept an address parameter or a new function that handles that could be created.
## <a id='L-02'></a>L-02. Unused Variable in SantaList.sol            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-11-Santas-List/blob/6627a6387adab89ae2ba2e82b38296723261c08a/src/SantasList.sol#L88


## Vulnerability Details
In santalist.sol the PURCHASED_PRESENT_COST variable was declared but never used anywhere in the contract

## Tools Used
Manual Review



