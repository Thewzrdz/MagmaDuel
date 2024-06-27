// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract MagmaDuel {
    struct Player {
        uint256 cardId;
        uint256 lives;
        bool isActive;
    }

    struct Card {
        uint256 id;
        uint256 value;
    }

    address public owner;
    Card[] public cards;
    mapping(address => Player) public players;
    address[] public playerAddresses;
    uint256 public buyInAmount = 0.009 ether;
    address public winner;
    uint256 public totalPot;

    event CardBought(address indexed player, uint256 cardId, uint256 value);
    event RoundWinner(address indexed winner, uint256 pot);
    event PotWithdrawn(address indexed owner, uint256 amount);

    modifier onlyOwner() {
        require(msg.sender == owner, "Only the owner can call this function");
        _;
    }

    modifier nonReentrant() {
        uint256 _guard = reentrancyGuard;
        require(_guard == 1, "ReentrancyGuard: reentrant call");
        reentrancyGuard = 2;
        _;
        reentrancyGuard = 1;
    }

    uint256 private reentrancyGuard = 1;

    constructor() {
        owner = msg.sender;
        // Initialize cards with their values
        cards.push(Card(0, 3));
        cards.push(Card(1, 69));
        cards.push(Card(2, 420));
    }

    function buyCard(uint256 cardId) external payable nonReentrant {
        require(msg.value == buyInAmount, "Incorrect buy-in amount");
        require(cardId < cards.length, "Invalid card ID");

        // Ensure player does not already have a card
        require(!players[msg.sender].isActive, "You already have a card");

        // Create a new player with the bought card
        players[msg.sender] = Player(cardId, 3, true);
        playerAddresses.push(msg.sender);

        totalPot += msg.value;

        emit CardBought(msg.sender, cardId, cards[cardId].value);
    }

    function playRound(uint256 cardId1, uint256 cardId2) external nonReentrant {
        require(players[msg.sender].isActive, "Player does not have a card");
        require(players[msg.sender].lives > 0, "Player has no lives left");

        require(cardId1 < cards.length && cardId2 < cards.length, "Invalid card IDs");

        uint256 value1 = cards[cardId1].value;
        uint256 value2 = cards[cardId2].value;

        if (value1 > value2) {
            players[msg.sender].lives -= 1;
            if (players[msg.sender].lives == 0) {
                winner = address(0); // No winner (all lives lost)
            }
        } else if (value2 > value1) {
            players[msg.sender].lives -= 1;
            if (players[msg.sender].lives == 0) {
                winner = address(0); // No winner (all lives lost)
            }
        } else {
            revert("Cards cannot have the same value");
        }

        // Emit round winner event if there is a winner
        if (winner != address(0)) {
            uint256 pot = totalPot;
            totalPot = 0;
            payable(winner).transfer(pot);
            emit RoundWinner(winner, pot);
        }
    }

    function getPlayerCard(address player) external view returns (uint256) {
        return players[player].cardId;
    }

    function getPlayerLives(address player) external view returns (uint256) {
        return players[player].lives;
    }

    function getCardsCount() external view returns (uint256) {
        return cards.length;
    }

    function withdrawPot() external onlyOwner nonReentrant {
        uint256 amount = totalPot;
        totalPot = 0;
        payable(owner).transfer(amount);
        emit PotWithdrawn(owner, amount);
    }

    // Fallback function to reject unexpected Ether transfers
    receive() external payable {
        revert("Direct transfers not allowed");
    }

    fallback() external payable {
        revert("Fallback function called");
    }
}
