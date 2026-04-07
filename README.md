# Vesting-Contract-with-Cliff-and-Linear-Release
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v5.0.0/contracts/token/ERC20/IERC20.sol";
import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v5.0.0/contracts/access/Ownable.sol";
import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v5.0.0/contracts/security/ReentrancyGuard.sol";

contract TokenVesting is Ownable, ReentrancyGuard {
    IERC20 public token;
    uint256 public totalVested;
    uint256 public cliffDuration;
    uint256 public vestingDuration;

    struct Vest {
        address beneficiary;
        uint256 amount;
        uint256 startTime;
        uint256 released;
    }

    mapping(address => Vest) public vests;

    event VestingCreated(address beneficiary, uint256 amount);
    event TokensReleased(address beneficiary, uint256 amount);

    constructor(address _token, uint256 _cliffDays, uint256 _vestingDays) Ownable(msg.sender) {
        token = IERC20(_token);
        cliffDuration = _cliffDays * 1 days;
        vestingDuration = _vestingDays * 1 days;
    }

    function createVesting(address beneficiary, uint256 amount) external onlyOwner {
        require(vests[beneficiary].amount == 0, "Already vested");
        vests[beneficiary] = Vest(beneficiary, amount, block.timestamp, 0);
        totalVested += amount;
        emit VestingCreated(beneficiary, amount);
    }

    function release() external nonReentrant {
        Vest storage vest = vests[msg.sender];
        require(vest.amount > 0, "No vesting");

        uint256 vestedAmount = vestedAmountOf(msg.sender);
        uint256 releasable = vestedAmount - vest.released;

        require(releasable > 0, "Nothing to release");

        vest.released += releasable;
        token.transfer(msg.sender, releasable);

        emit TokensReleased(msg.sender, releasable);
    }

    function vestedAmountOf(address beneficiary) public view returns (uint256) {
        Vest memory vest = vests[beneficiary];
        if (block.timestamp < vest.startTime + cliffDuration) return 0;

        if (block.timestamp >= vest.startTime + cliffDuration + vestingDuration) {
            return vest.amount;
        }

        uint256 timePassed = block.timestamp - (vest.startTime + cliffDuration);
        return (vest.amount * timePassed) / vestingDuration;
    }
}
