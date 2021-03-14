# RAI X UMA Challenge by Reflexer Labs


# <pre>**Challenge**</pre>
https://gitcoin.co/issue/reflexer-labs/geb/97/100024834

To build a synthetic asset tracking the
[Kovan RAI](https://github.com/reflexer-labs/geb-changelog/tree/master/releases/kovan/1.4.0/median/fixed-discount)
redemption rate movements using [UMA](https://docs.umaproject.org/build-walkthrough/build-process).

# <pre>**Overview**</pre>

UMA is a fast, flexible and secure way to create synthetic assets on Ethereum. UMA has defined a novel architecture
that enables anyone to create a synth asset that can track virtually anything from stock markets in Iran to gas 
fees on Ethereum safely and securely.

In this challenge I created a synth asset `RR-RAI-APR21` tracking RAI redemption rate.

Also a DApp (fork of UMAProtocol/emp-tools) to interact with my EMP and manage/create their positions.

https://emp-tools-2391flb9z-ashutoshvarma.vercel.app/


# <pre>**Solution**</pre>

## Pricing Model
`Redemption_Rate` is the rate at which RAI is being devalued or revalued, it can therefore be negative as well. It is stored as `redemptionRate` in `OracleRelayer` relayer contract. Mathematically,
```
redemptionRate = Redemption_Rate + 1
```
Therefore `redemptionRate` is bound in (0,2) which makes it a sensible candiadate to track our synthetic asset. But `redemptionRate` changes very slowly and is almost constant upto few decimals. For example, these are some consecutive value for `redemptionRate` :-
```
0.99999999984948877039244582
0.999999999848547411861463422,     // Starting 10 decimals are almost consatnt
0.999999999848547690004497233.
```

To mitigate this for somme extent we can use `annualizedRate` which is scaled version of redemptionRate.
```
annualizedRate = (redemptionRate) ^ (365 * 12 * 30 * 24 * 3600)
```

I avoided using complex scaling methods to keep implementaion as simple as possible. (In case of dispute, every UMA shareholder should be able to calculate correct price without any issue. Also same price value should be reproducable accross different programming languages like python, bash upto 18 decimals (Wei) incase someone decides to use different langauge)

Lastly, to prevent market manuplation due to flash loans and other factors which can make `Redemption_Rate` volatile for very short period (which can casue sudden liquidations) so we should take Time Weighted Average Price (TWAP) of `annualizedRate`.

### PriceFeed Implementation
Sources of data :- 
- https://subgraph-kovan.reflexer.finance/subgraphs/name/reflexer-labs/rai/
- `UpdateRedemptionRate` Event from [`RateSetter`](https://kovan.etherscan.io/address/0x0641C280B21A31daf1518a91A68Ad396EcC6f2f0#events) contract


`RaiRedemptionPriceFeed.js` calculates  TWAP (8 hours by default) of `annualizedRate`. While calculating TWAP timestamp of asset price should be accurate so for that purpose we will use timestamp of block in which `redemptionRate` is changed (or `UpdateRedemptionRate` event block time).

Since `redemptionRate` is updated every `updateRateDelay` (saved in [`RateSetter`](https://kovan.etherscan.io/address/0x0641C280B21A31daf1518a91A68Ad396EcC6f2f0#readContract)) seconds, for Kovan it is 3 Hrs so TWAP length is kept 8 hrs to make sure atleast 2 `annualizedRate` are used to calculate price.
Also sometimes there is delay of 15min in rate updation ( see https://discord.com/channels/698935373568540753/698936206691401759/818863474364514352 for full discussion).

There can be a delay of about 40-60s (or sometimes more) in indexing new events to subgraph, that can lead to wrong calculation of TWAP as there might be a new price which is not indexed in subgraph yet, so to prevent this we employ following logic

```
PRICES = QUERY_SUBGRAPH()                         // Fetch prices from subgraph (with block number to compute timestamp)

LATEST_SUBGRAPH_TIMESTAMP = PRICES[0].timestamp   // Subgraphs's latest price's (block) timestamp 

NEXT_RATE_UPDATE_TIME = LATEST_SUBGRAPH_TIMESTAMP + UPDATE_RATE_DELAY     // time when rate will be changed by RAI bots

IF CURRENT_TIME > NEXT_RATE_UPDATE_TIME:                                    // If this is true, that means subgraph might not have lastest price indexed.
    LATEST_PRICE = PRICE_FROM_MOST_RECENT_EVENT()                           // Try to get price from latest UpdateRedemptionRate event
    IF LATEST_PRICE && LATEST_PRICE.timestamp > LATEST_SUBGRAPH_TIMESTAMP:  // If price from event is newer than lastest subgraph 
        PRICES.push(LATEST_PRICE)                                           // add to prices list
    END
END

CURRENT_PRICE = TWAP(PRICES)
```

**Why don't just read `redemptionRate` from `OracleRelayer` ?**

Problem is that we need timestamp for a given price also inorder to calculate correct TWAP, reading the value from contract will not give us timestamp of price. 


### Code & Tests
The full implementation of price feed with unit tests is contained in UMA protocol repo's fork.
https://github.com/ashutoshvarma/protocol

**RaiRedemptionPriceFeed** - [here](https://github.com/ashutoshvarma/protocol/blob/master/packages/financial-templates-lib/src/price-feed/RAIRedemptionRatePriceFeed.js)

**Unit tests** - [here](https://github.com/ashutoshvarma/protocol/blob/master/packages/financial-templates-lib/test/truffle/RAIRedemptionRatePriceFeed.js)

Also default price-feed configuration has been added for bots to work with minimal configuration - [eacb633](https://github.com/ashutoshvarma/protocol/commit/eacb6338ab598d28e0a30fcf4050154087b159cd)
 

_UMA's `Networker` class does not support sending POST requests which was nesseary in order to query subgraphs. To add support for POST requests I made few small changes to it. Here is the PR_ https://github.com/UMAprotocol/protocol/pull/2691


# <pre>**Deployment**</pre>
**An EMP UMA Contract and Token** has been deployed to the Kovan testnet and a **UMA liquidation & disputer bot** is configured to use the `RaiRedemptionPriceFeed`.

## Setup Configuration
### 1. Collateral Currency - `RAI`
```
"symbol":           "RAI",
"name":             "Rai Reflex Index"
"address":          "0x76b06a2f6df6f0514e7bec52a9afb3f603b477cd",
"decimals":         18,
```
Added buy UMA team to `
AddressWhitelist`. through this [transaction](https://kovan.etherscan.io/tx/0x006f18d76ba32ae42e2ca73eea703c9c5574c835773b447342bd46e71964ae6f)

### 2. Price Indentifier Name - `RaiRedemptionRate`
Added by UMA Team to `IdentifierWhitelist` through this [transaction](https://kovan.etherscan.io/tx/0x5cc0ccb70a86480af46385105d1b3e6318554df7c503ee43c055de77a0fb2b9b)

### 2. Synth Token - `RR-RAI-APR21`
```
syntheticName:      "RAI Redemption Rate [RR April 2021]", 
syntheticSymbol:    "RR-RAI-APR21", 
```
Token Deployed (by EMP) at 
https://kovan.etherscan.io/token/0xcac5b5ac9f4af1a4b73a12cd007a64ba4dfa07c2

### 3. EMP Parameteres
```
Expiry date:            30/04/2021, 16:30:00 UTC
Price identifier:       RaiRedemptionRate
Collateral requirement: 1.25
Unique sponsors:        1
Minimum sponsor tokens: 100.0 RR-RAI-APR21
```
(This EMP will expire at the end of April 2021 and synth holders can redeem their token then.)

Deployed at

https://kovan.etherscan.io/address/0x08eA186755Ad743897c00AAfaEF7Fb9A7EcE8cf3

_While trying to deploy EMP using UMAProject/launch-emp scripts I faced some errors due to incompatibility between old ganache-cli version and node 14, I made a small PR for this also_ ,https://github.com/UMAprotocol/launch-emp/pull/14


# <pre>**Uniswap Pool - `RAI` & `RR-RAI-APR21`**</pre>
Create and add liquidity to Kovan Uniswap Pool of `RAI` and ``RR-RAI-APR21``.

https://app.uniswap.org/#/swap?outputCurrency=0xCaC5B5AC9F4af1A4b73a12CD007A64BA4DFa07C2

**Pool Address** - https://kovan.etherscan.io/address/0xff0ca43fc5444dc464177cbd82419d42db1af761#tokentxns

**NOTE**:- You might need to import `RAI` & `RR-RAI-APR21` tokens in uniswap.

# <pre>**LIVE DApp**</pre>

**Live** - https://emp-tools-2391flb9z-ashutoshvarma.vercel.app/

**Source** - https://github.com/ashutoshvarma/emp-tools

A Simple DApp to interact with EMP, manage position, deposit collateral, redeem synth and view positions or liquidations.
(Fork from [emp-tools](https://github.com/UMAprotocol/emp-tools/))

### Changes made to original `emp-tools` for supporting RAI EMP contract 
1. Disbale DevMining interfaces
2. Use updated EMP ABI.
3. Replace Old EMP ABI methods with updated one.
4. Add dummy price config for `RR-RAI-APR21` synth.

![image](https://user-images.githubusercontent.com/17181457/111082409-1e672c80-852e-11eb-8b11-5eddda7493cf.png)


## Refrence
### RAI

* **OracleRelayer**:- [Source](https://github.com/reflexer-labs/geb/blob/master/src/OracleRelayer.sol),
[Docs](https://docs.reflexer.finance/system-contracts/oracle-module/oracle-relayer)

* **RateSetter**:- [Kovan](https://kovan.etherscan.io/address/0x0641C280B21A31daf1518a91A68Ad396EcC6f2f0#code), [Events](https://kovan.etherscan.io/address/0x0641C280B21A31daf1518a91A68Ad396EcC6f2f0#events)

### UMA

#### Setup and Minting a Synthetic token

* Setup https://docs.umaproject.org/developers/setup
* Local install https://docs.umaproject.org/build-walkthrough/mint-locally
* EMP on net https://docs.umaproject.org/developers/emp-deployment
* Kovan addresses https://github.com/UMAprotocol/protocol/blob/master/packages/core/networks/42.json


#### Bots

* Bot parametrization https://docs.umaproject.org/developers/bot-param





