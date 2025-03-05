### My Findings of the [2025-01-DivaWrapper](https://codehawks.cyfrin.io/c/2025-01-diva)
# Aave DIVA Wrapper - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## Low Risk Findings
    - ### [L-01. Constructor Parameter Mismatch in AaveDIVAWrapper and AaveDIVAWrapperCore Contracts Leading to Failing Function Calls](#L-01)


# <a id='contest-summary'>Contest Summary</a>

### Sponsor: [Walodja1987](https://x.com/Walodja1987)

### Dates: Jan 24th, 2025 → Jan 31st, 2025

[View Full Report Here](https://codehawks.cyfrin.io/c/2025-01-diva/results?lt=contest&page=1&sc=reward&sj=reward&t=report)

# <a id='results-summary'>Results Summary</a>

### Number of findings:
- High: 0
- Medium: 0
- Low: 1



    


# Low Risk Findings

## <a id='L-01'></a>L-01. Constructor Parameter Mismatch in AaveDIVAWrapper and AaveDIVAWrapperCore Contracts Leading to Failing Function Calls            



## Description

The order of parameters in `AaveDIVAWrapper` does not match the expected order in `AaveDIVAWrapperCore`, leading to incorrect assignment of contract addresses and potential malfunction of the deployed contract.

The constructor in `AaveDIVAWrapper` is structured as:

```solidity
constructor(address _aaveV3Pool, address _diva, address _owner) AaveDIVAWrapperCore(_aaveV3Pool, _diva, _owner) {}
```

But the 'AaveDIVAWrapperCore\` has its constructor structured as:

```solidity
constructor(address diva_, address aaveV3Pool_, address owner_) Ownable(owner_) {}
```

This leads to improper initialization of DIVA and Aave V3 Pool addresses in the core contract.
DIVA is initialized to Aave V3 Pool address and vice versa.

## Impact

* Misconfiguration of contract dependencies, leading to function calls failing due to incorrect addresses.

* Unexpected behavior during contract execution, requiring redeployment to fix.

## Tools Used

Manual Review\
Foundry

## Recommendations

In `AaveDIVAWrapper` change the order of the variables passed to the constructor of `AaveDIVAWrapperCore`

```diff
- constructor(address _aaveV3Pool, address _diva, address _owner) AaveDIVAWrapperCore(_aaveV3Pool, _diva, _owner) {}
+ constructor(address _aaveV3Pool, address _diva, address _owner) AaveDIVAWrapperCore(_diva, _aaveV3Pool, _owner) {}
```



