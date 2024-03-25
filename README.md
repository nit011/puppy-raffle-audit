

# Puppy Raffle

This project is to enter a raffle to win a cute dog NFT. The protocol should do the following:

1. Call the `enterRaffle` function with the following parameters:
   1. `address[] participants`: A list of addresses that enter. You can use this to enter yourself multiple times, or yourself and a group of your friends.
2. Duplicate addresses are not allowed
3. Users are allowed to get a refund of their ticket & `value` if they call the `refund` function
4. Every X seconds, the raffle will be able to draw a winner and be minted a random puppy
5. The owner of the protocol will set a feeAddress to take a cut of the `value`, and the rest of the funds will be sent to the winner of the puppy.

- [Puppy Raffle](#puppy-raffle)
- [Getting Started](#getting-started)
  - [Requirements](#requirements)
  - [Quickstart](#quickstart)
    - [Optional Gitpod](#optional-gitpod)
- [Usage](#usage)
  - [Testing](#testing)
    - [Test Coverage](#test-coverage)
- [Audit Scope Details](#audit-scope-details)
  - [Compatibilities](#compatibilities)
- [Roles](#roles)
- [Known Issues](#known-issues)





# Usage

## Testing

```
forge test
```

### Test Coverage

```
forge coverage
```

and for coverage based testing:

```
forge coverage --report debug
```

# Audit Scope Details

- Commit Hash: e30d199697bbc822b646d76533b66b7d529b8ef5
- In Scope:

```
./src/
└── PuppyRaffle.sol
```

## Compatibilities

- Solc Version: 0.7.6
- Chain(s) to deploy contract to: Ethereum

# Roles

Owner - Deployer of the protocol, has the power to change the wallet address to which fees are sent through the `changeFeeAddress` function.
Player - Participant of the raffle, has the power to enter the raffle with the `enterRaffle` function and refund value through `refund` function.

# Known Issues

None



 *Concepts : Static analysis, Reentrancy, Weak RNG, Arithmetic issues, How to write a professional looking report.*

## Tooling: Static Analysis
 - [Web3 bugs machine vs human](https://github.com/ZhangZhuoSJTU/Web3Bugs)
 - Static Analysis
   - [Slither](https://github.com/crytic/slither)
   - [Aderyn](https://github.com/Cyfrin/aderyn)
 - [cloc](https://github.com/AlDanial/cloc)
 - [Solidity Metrics (audit estimation)](https://github.com/Consensys/solidity-metrics)
 - [Solidity Visual Developer](https://marketplace.visualstudio.com/items?itemName=tintinweb.solidity-visual-auditor)
## Scoping & Reconnaissance: Puppy Raffle
### Exploits: DoS (Denial of service)
  - Fixes:
    - Remove unnecessary loops
### Exploits: Reentrancy
  - [Case Study: DAO Hack](https://www.gemini.com/cryptopedia/the-dao-hack-makerdao)
    - [Still plagues us today](https://github.com/pcaversaccio/reentrancy-attacks)
  - [Exercises](https://github.com/Cyfrin/sc-exploits-minimized/tree/main/src/reentrancy)
    - [Search "reentrancy" in Solodit](https://solodit.xyz/)
  - Prevention:
    - CEI/CEII ( FREI-PI soon!)
    - NonReentrant modifiers
### Exploits: Weak RNG
  - [Case Study: Meebits](https://forum.openzeppelin.com/t/understanding-the-meebits-exploit/8281)
  - [Exercises](https://github.com/Cyfrin/sc-exploits-minimized/tree/main/src/weak-randomness)
    - [Search "RNG" in Solodit](https://solodit.xyz/)
  - Prevention:
    - [Chainlink VRF](https://docs.chain.link/vrf)
### Exploits: Arithmetic issues
   - Examples:
     - Under/Overflow
     - Rounding & Precision
   - [Exercises](https://github.com/Cyfrin/sc-exploits-minimized/tree/main/src/arithmetic)
     - [Search "overflow" in Solodit](https://solodit.xyz/)
   - Prevention:
       - Use newer versions of solidity 
       - Multiply before divide
### Exploits: Poor ETH Handling
  - Case study: [Sushiswap Miso](https://samczsun.com/two-rights-might-make-a-wrong/)
  - Exercises:
    - [Stuck ETH without a way to withdraw ](https://gist.github.com/tinchoabbate/99fbf7cbce47eb7c463212fd13f21149)
    - [Mishandling ETH](https://github.com/Cyfrin/sc-exploits-minimized/tree/main/src/mishandling-of-eth)
    - [Search "Stuck ETH" in Solodit](https://solodit.xyz/)
### Informational Findings
   - Stict Solc Versioning 
   - Supply Chain Attacks 
   - Magic Numbers 
### Gas Audits 
### Code Maturity 
   - Code coverage
### Static Analysis, follow up
## What is a Competitive Audit? 
  - [CodeHawks Docs](https://docs.codehawks.com/)
