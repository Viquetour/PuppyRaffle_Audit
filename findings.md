### [H-1]

**Description:** The `PuppleRaffle::Refund` function doesnot follow CEI(checks
Effects Interactions) as a result enables participants to drain the contract balance

In the `PuppleRaffle::refund` function we, first make an external call to the
`msg.sender` address and only after making that external call do we update the
`puppleRaffle::players`

```javascript

  function refund(uint256 playerIndex) public {
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");

@>      payable(msg.sender).sendValue(entranceFee);

@>      players[playerIndex] = address(0);
        emit RaffleRefunded(playerAddress);
    }

```
a player who has entered the raffle could have a `fallback`or`recieve` function that calls a `puppyRaffle::refund` function again and claim another refund. they could continue the circle
till the contract balnce is drained.    


    
**Impact:** All fees paid by raffle entrants can be stolen by malicious participants.

**Prof of Concept**

1. User enters Raffle
2. Attacker sets up a contract witih a `fallback` that calls `puppyRaffle::refund`.
3. Attacker enters the raffle
4. Attacker calls `puppyRaffle::refund` from their attack contract, draining the contract balance

**Proof Of Code**

<details>
<summary>Code</summary>
place the following into `puppyRaffleTest.t.sol`

 function test_rentrancyRefund() public {
        // Enter initial players
        address[] memory players = new address[](4);
        players[0] = address(1);
        players[1] = address(2);
        players[2] = address(3);
        players[3] = address(4);
        puppyRaffle.enterRaffle{value: entranceFee * 4}(players);
    
        // Create and fund attacker contract
        ReentrancyAttacker attackerContract = new ReentrancyAttacker(puppyRaffle);
        vm.deal(address(attackerContract), 1 ether);
    
        uint256 startingAttackContractBalance = address(attackerContract).balance;
        uint256 startingContractBalance = address(puppyRaffle).balance;
    
        // Perform the attack
        attackerContract.attack{value: entranceFee}();
    
        console.log("Starting attacker Contract balance: ", startingAttackContractBalance);
        console.log("Starting Contract balance: ", startingContractBalance);
        console.log("Ending attacker Contract balance: ", address(attackerContract).balance);
        console.log("Ending PuppyRaffle Contract balance: ", address(puppyRaffle).balance);
    
        // Add assertions here to check if the attack was successful or prevented
    }
````
and this contract as well.

```javascript
contract ReentrancyAttacker {
    PuppyRaffle public puppyRaffle;
    
    constructor(PuppyRaffle _puppyRaffle) {
        puppyRaffle = _puppyRaffle;
    }
    
    function attack() external payable {
        address[] memory players = new address[](1);
        players[0] = address(this);
        puppyRaffle.enterRaffle{value: msg.value}(players);
        // The reentrancy would typically happen in the fallback or receive function
    }
    
    receive() external payable {
        if (address(puppyRaffle).balance >= msg.value) {
            puppyRaffle.refund(0); // Attempt to drain the contract
        }
    }
}


```

</details>

**Recomended Mitigation**To prevent this, we should have the `puppleRaffle::refund`
 function update the `players`array before making the externall call. additionally we should move the event emmision up as well.
 ```diff
 function refund(uint256 playerIndex) public {
        //written skipped --MEV attack
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");
+        players[playerIndex] = address(0);
+        emit RaffleRefunded(playerAddress);
        payable(msg.sender).sendValue(entranceFee);

_        players[playerIndex] = address(0);
_        emit RaffleRefunded(playerAddress);
    }
```
 function enterRaffle(address[] memory newPlayers) public payable {
        //what if it's 0?
        require(msg.value == entranceFee * newPlayers.length, "PuppyRaffle: Must send enough to enter raffle");
        for (uint256 i = 0; i < newPlayers.length; i++) {
            players.push(newPlayers[i]);
        }
        }
        emit RaffleEnter(newPlayers);
    } 
     ```
### [M-#] Looping through players array to check for duplicates in `puppyRaffle::enterRaffle`is a potentil denial(DoS) of service attack,imcrementing gas cost for future entrants.

**Description:** The `puppyRaffle::enterRaffle` function loops through `players` array to check for duplicates. However the longer the `puppleRaffle::players` array is, the more checks a player will have to make. This means the gas cost for players who enter right when the lower starts will be drammatically lower than thoes who enter later.
Every additional address in the `players` array, is an additional check the loop will have to make.

** Impact ** The gas cost for raffle entrants will greatly increase as more players enter the raffle. Discouraging later users from entering, and causing a rush at the start of a raffle t be one of the first entrants in the queue. 

An attacker can make the `puppyRaffle::entrants` so big that no one else enters guranteeing themselves the win.


## Proof of Concept ## If we have 2 sets of 100 players enter, the gas cost will be as such:
-First 100 players: ~6252128 gas
-Second 100 players: ~18068218 gas

this is more than 3x expensive for the second 100 players.

<details>
<summary>PoC</summary>
place the following test into `puppyRaffleTest.t.sol`;

```javascript

 function test_denialOfService() public {
        vm.txGasPrice(1);
        //Lets Enter 100 players
        uint256 playersNum = 100;
        address[] memory players = new address[](playersNum);
        for (uint256 i = 0; i < playersNum; i++) {
            players[i] = address(i);
        }
        //see how much gas it cost
        uint256 gasStart = gasleft();
        puppyRaffle.enterRaffle{value: entranceFee * players.length}(players);
        uint256 gasEnd = gasleft();
        uint256 gasUsedFirst = (gasStart- gasEnd) * tx.gasprice;

        console.log("The price for the first 100 players is: ", gasUsedFirst);

        //for the second 100 players

        vm.txGasPrice(1);
        //Lets Enter 100 players
        
        address[] memory playersTwo = new address[](playersNum);
        for (uint256 i = 0; i < playersNum; i++) {
            playersTwo[i] = address(i + playersNum);
        }
        //see how much gas it cost
        uint256 gasStartSecond = gasleft();
        puppyRaffle.enterRaffle{value: entranceFee * playersTwo.length}(playersTwo);
        uint256 gasEndSecond = gasleft();
        uint256 gasUsedSecond = (gasStartSecond- gasEndSecond) * tx.gasprice;

        console.log("The price for the first 100 players is: ", gasUsedSecond);

        assert(gasUsedFirst < gasUsedSecond);
    }
```
</details>
** Recomended Mitigation: ** There are a few recomendations.
1. consider allowing duplicates. Users can make new wallet addresses, so a duplicate check doesn't prevent the same person from entering multiple times, only the same wallet addresses.

2. consider using a mapping to check for duplicates. this will allow constant time lookup of whether a user has already entered.

```diff


    /// @notice a way to get the index in the array
    /// @param player the address of a player in the raffle
    /// @return the index of the player in the array, if they are not active, it returns 0
    function getActivePlayerIndex(address player) external view returns (uint256) {
        for (uint256 i = 0; i < players.length; i++) {
            if (players[i] == player) {
                return i;
            }
        }
        return 0;
    }

    /// @notice this function will select a winner and mint a puppy
    /// @notice there must be at least 4 players, and the duration has occurred
    /// @notice the previous winner is stored in the previousWinner variable
    /// @dev we use a hash of on-chain data to generate the random numbers
    /// @dev we reset the active players array after the winner is selected
    /// @dev we send 80% of the funds to the winner, the other 20% goes to the feeAddress
    function selectWinner() external {
        require(block.timestamp >= raffleStartTime + raffleDuration, "PuppyRaffle: Raffle not over");
       

```
Alternatively you can use [@openzeppelin's `EnumarableSet` library]
(https://docs.openzeppelin.com/contracts/3.x/api/utils)

### [1-1] solidity pragma should be specific not wide

Consider using a specific version of solidity in your contract instead of wide versions.
For example, instead of ` pragma solidity ^0.7.6 ` use `pragma solidity 0.8.0;`

### [1-2] Using outdated version of solidity is not advisable

pls try using a newer version like `pragma solidity ^0.8.18`

## Recomendations ##
solc versions 0.4.7-0.5.9 contain a compiler bug leading to incorrect ABI encoder usage.

Exploit Scenario:
contract A {
    uint[2][3] bad_arr = [[1, 2], [3, 4], [5, 6]];
    
    /* Array of arrays passed to abi.encode is vulnerable */
    function bad() public {                                                                                          
        bytes memory b = abi.encode(bad_arr);
    }
}
abi.encode(bad_arr) in a call to bad() will incorrectly encode the array as [[1, 2], [2, 3], [3, 4]] and lead to unintended behavior.

Recommendation
Use a compiler >= 0.5.10.

please see [slither](
    https://github.com/crytic/slither/wiki/Detector
    -Documentation#incorect-version-of-solidity
)

#L

##[L-1]`PuppleRaffle::getActivePlayerIndex` returns 0 for non exixtent playersand for players at index 0 causing a player at index 0, to incorrectly think they have not enterd the raffle.

**Description:** If a player is in the `puppleRaffle:`array at index 0 this will return 0,
but according to the natspec it will also return 0 if the player is not in the array.
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
**Impact:** A player at index 0 may incorrectly think they have enterd the raffle and attemp to enter the raffle again, wasting gas,

**Proof of Concept:**
1. User enters the raffle they're the first entrants.
2. `PuppyRaffle::getActivePlayer`Index Returns 0
3. User thinks they have not enetered correctly due to the function.


**Recomended Mitigation:** The easiest recomendation is to revert if the player is not in the array instead of returning to 0.

You could also reserve the 0th position in a competitive audit, but a better solution might be to reutrn an `int256` where the function returns -1 if the player is not active.

### G
### [G-1] Unchanged state varibles should be declared constant or immutable##

Instances
reading from a constant is much more expensive than reading from a 
constant or immutable variable.

-  `PuppleRaffle::raffleDuration ` should be `immutable`
-  `puppyRaffle ::commonImageUri` should be `Constant`
-  `PuppyRaffle::rareImageUri` should be `constant`
-  `PuppyRaffle::lagendryUri` should be `constant`

##[1-4] `PuppyRaffle::SelectWinner`Does not  follow CEI which is not a best practice.
it's best to keep code clean and follow CEI(Check Effect interaction)

```javascript
-       (bool success,) = winner.call{value: prizePool}("");
-        require(success, "PuppyRaffle: Failed to send prize pool to winner");
        _safeMint(winner, tokenId);
+       (bool success,) = winner.call{value: prizePool}("");
+         require(success, "PuppyRaffle: Failed to send prize pool to winner");
        _safeMint(winner, tokenId);
    }

```
### [1-5] Use of "magic" numbers is discouraged

It can be confusing to see number literals in a codebase, and it's much more
readable if the numbers are giving a name.

Example:
```javascript
        uint256 prizePool = (totalAmountCollected * 80) / 100;
        uint256 fee = (totalAmountCollected * 20) / 100;       
```
Instead, you could use:
```javascript
        
        //uint256 public constant PRIZE_POOL_PERCENTAGE = 80;
        //uint256 public constant FEE_PERCENTAGE = 20;
        //uint256 public constant POOL_PRECISION = 100;
```

# Additional findings not taught in course

##MEV




