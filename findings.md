### [M-#] looping through players array to check for duplicates in `puppyRaffle::enterRaffle`is a potentil denial(DoS) of service attack,imcrementing gas cost for future entrants.

**Description:** The `puppyRaffle::enterRaffle` function loops through `players` array to check for duplicates. However the longer the `puppleRaffle::players` array is, the more checks a player will have to make. This means the gas cost for players who enter right when the lower starts will be drammatically lower than thoes who enter later.
Every additional address in the `players` array, is an additional check the loop will have to make.

**Impact ** The gas cost for raffle entrants will greatly increase as more players enter the raffle. Discouraging later users from entering, and causing a rush at the start of a raffle t be one of the first entrants in the queue. 

An attacker can make the `puppyRaffle::entrants` so big that no one else enters guranteeing themselves the win.


** Proof of concept ** If we have 2 sets of 100 players enter, the gas cost will be as such:
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

 function enterRaffle(address[] memory newPlayers) public payable {
        //what if it's 0?
        require(msg.value == entranceFee * newPlayers.length, "PuppyRaffle: Must send enough to enter raffle");
        for (uint256 i = 0; i < newPlayers.length; i++) {
            players.push(newPlayers[i]);
        }

        // Check for duplicates
        //@audit DoS
        }
        emit RaffleEnter(newPlayers);
    }

    /// @param playerIndex the index of the player to refund. You can find it externally by calling `getActivePlayerIndex`
    /// @dev This function will allow there to be blank spots in the array
    function refund(uint256 playerIndex) public {
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");

        payable(msg.sender).sendValue(entranceFee);

        players[playerIndex] = address(0);
        emit RaffleRefunded(playerAddress);
    }

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



