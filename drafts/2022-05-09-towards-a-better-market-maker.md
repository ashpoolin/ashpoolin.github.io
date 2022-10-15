---
layout: post
---

## How the f*** do you use the serum-ts client? (part 4)

We recently introduced an extremely naïve maker bot strategy [(in Part 2 of this series)](https://ashpoolin.github.io/the-making-of-a-market-makerer) that we called the "Greedy Bot." In this episode, we will return to the concept of a market maker bot, but implement a slightly more informed strategy that addresses certain risks including volatility and inventory management.

### DISCLAIMER
Legal: Serum order book is an open order book that enables trading of digital assets. This article is produced as educational content, and an exploration of the technical features of the order book. It is not an inducement to invest or buy digital assets or cryptocurrencies. It is your responsibility to abide by all applicable laws. 

Liability: The article is bound to be riddled with mistakes, please help correct them using a PR through github, or sending a message. There are no warranties express or implied for this content, and the author will claim no responsibility for any person's loss of funds while applying information obtained from here, or while trading. 

### Greedy Bot
Our previous incarnation of the market maker was the Greedy Bot, a bot which jumped to the front of the order book and merely sloshed coins back and forth while hoping to capture the minimum spread (difference between the lowest ask and highest bid). The plan may work for some types of trading pairs that are tightly coupled--such as staked versus native tokens--but it was incredibly uninformed, and anyone using such a scheme is bound to be rekt from adverse selection against good traders, or in stable-quoted markets. The problems are many-fold:
1. *Inventory Risk* - We had no concept of inventory or position size. Whatever tokens were free to trade, we shoved them into the order book. This allowed the strategy to accumulate too much of either the base or quote asset, thereby putting the portfolio at risk of significant losses, should there be a divergence in the prices of the assets. 
2. *Adverse Selection* - All market makers are at risk of trading against more-informed traders, who may have insight into the direction an asset may move. To compensate for that, market makers have been historically been compensated via the bid-ask spread, fee reductions for significant trading volume, and even direct payments for ensuring liquidity (good prices and volume) at all times, even when conditions are unfavorable. Some crypto exchanges introduce the concept of *liquidity mining* which is the same sort of compensation, but paid in the form of a token. Crypto enables many of the vehicles for supporting market makers, which should be considered, since the specter of *adversity* is ever present, and is capable of wiping out earnings in a matter of minutes if care is not taken. 
3. *Volatility* - Similar to adverse selection, when prices are on the move, it is inappropriate to quote tight spreads. Volatility is mathematically identical to root-mean-squared error (RMSE), which is essentially a measure of *uncertainty*. It follows that if there is high uncertainly in a price, you need to widen your spreads because you don't have enough information to quote them correctly. The chances of offering a bad quote, and being picked off by savvy traders is just too high. To counteract (but not eliminate) this risk, wider spreaders force these traders to pay a worse price.  

As you can see, there are many risks for market makers that the previous design did not address, which makes it wholly unsafe to use with any material amount of money.

### Better Bot
To try to address the risks enumerated above, we introduce a new prototype we call the "Better Bot." First, rather than just following around the order book, we use the Pyth Network's oracle data to give us a "true North" on the actual price data. Next, this design implements a new, home-rolled metric that I call the `inventory ratio (IR)` that enforces a passive control mechanism for keeping the inventory balanced. We introduce the idea of tolerances for managing stale orders. e.g. how far off the optimal bid or ask prices our offers can deviate before we cancel them. Additionally, the bot takes into consideration the current market's *volatility* to modulate the optimal spread (e.g. increase spread in high volatility). Finally, we provide firm guardrails via setting minimum and maximum spreads, as well as a bot "circuit breaker" that forces it to cancel all offers and sit out when the maximum spread is exceeded, in order to wait until volatility has subsided. While the design is far from perfect, and only addresses the issue of adverse selection by proxy (via modulating spreads vs. volatility), it is a significant improvement from the greedy bot, and gets closer to how a "good" market maker bot should behave.

### Better Bot Code and Implementation

We'll keep a lot of our boiler plate code: the imports, markets, date module. All that crap is shown here:
```
import {Cluster, Keypair, Connection, PublicKey } from '@solana/web3.js';
import { Market } from '@project-serum/serum'; 
import fs from 'fs';

import { PythConnection } from './PythConnection.js';
import { getPythProgramKeyForCluster } from './cluster.js';

// date shit
export function padTo2Digits (num) {
  return num.toString().padStart(2, '0');
}

export function formatDate (date) {
  return (
    [
      date.getFullYear(),
      padTo2Digits(date.getMonth() + 1),
      padTo2Digits(date.getDate()),
    ].join('-') +
    ' ' +
    [
      padTo2Digits(date.getHours()),
      padTo2Digits(date.getMinutes()),
      padTo2Digits(date.getSeconds()),
    ].join(':')
  );
}
```

Before we hop into each of the functions we write, there are some structural changes that we're going to implement. Now, because we want to keep an eye on what the market is doing, we'll need to include our hookup to the Pyth Network, as we previously wrote about [here](https://ashpoolin.github.io/taking-a-pythstop). We are also now using a pseudo-Typescript code, since we need to transpile the code from TS to JS, as some of the Pyth code we want to use (imported into `src/`) is written in Typescript. Other items we described as changing are that we wrap the whole thing in a C-like `async function main() {...}` because Typescript will not allow async calls (e.g. `await ...`) in the top level of modules. Within our `main()` function we'll have the pyth price updates and calculations happening asynchronously from a `while` loop, where the actual trading happens. See below:  

```
// betterbot.ts

async function main() {

    // 1a. initialization stuff
    // 1b. import keypair
    // 1c. set trader config details
  
    pythConnection.onPriceChange((product, price) => {
        // 2. pyth calculations and moving average
    })  
    pythConnection.start()

    while(true) {
        // 3a. calculate inventory ratio (IR)
        // 3b. calculate optimal bid and ask prices, and the min/max bid and ask values (a range) for cancelling orders
        // 3c. cancel stale orders (if applicable), wait for tokens to settle
        // 3d. check balances: see if we have free tokens to trade
        // 3e. check back to see if there are still open orders. If yes --> skip placing orders this round
        // 3f. compute volatility-dependent optimal spread, and recompute optimal bid and ask prices (keep them fresh)
        // 3g. if the optimal spread is too high, do not place orders, else place fresh orders
    }
}
main();
```

There's a lot to go through, but it's not very complicated. Let's just hit them one at a time:

#### 1a. initialization stuff

```
async function main() {
  // Select an RPC
  const connection = new Connection('https://solana-api.projectserum.com/');

   // select the market
  let marketAddress = new PublicKey('7gtMZphDnZre32WfedWnDLhYYWJ2av1CCn1RES5g8QUf'); // ETH/SOL
  let programAddress = new PublicKey('9xQeWvG816bUx9EPjHmaT23yvVM2ZWbrrpZb9PusVFin'); // Serum V3 program ID 
  const market = await Market.load(connection, marketAddress, {}, programAddress);

  // misc info market params
  const MIN_ORDER_SIZE: number = market.minOrderSize;
  const TICK_SIZE: number = market.tickSize;
```

#### 1b. import keypair
Pretty straightforward. Just set an environment variable (e.g. `$ KEYPAIR=/path/to/keypair.json`), and the program will find it, load it into this Keypair data type:

```
  // load up your keypair
  const owner: Keypair = Keypair.fromSecretKey(
    Uint8Array.from(
      JSON.parse(
        fs.readFileSync(
          //@ts-ignore
          process.env.KEYPAIR,
          'utf-8',
        ),
      ),
    )
  );
}
```

#### 1c. set trader config details
The following items we are configuring the bot to have some parameters or rules to trade within.
```
  // bot config parameters
  const MAX_SIZE_BASE: number = 0.1; // in ETH
  const SPREAD_COEFF_OPTIMAL: number = 0.005; // default value; gets overwritten quickly
  const OFFER_TOLERANCE: number = 0.2; // ratio (e.g. 20%) for how far off the optimal prices before cancelling orders
  
  // number conversion constants
  const BASE_BN_OFFSET: number = 100000000; // eth is x 1E-8
  const QUOTE_BN_OFFSET: number = 1000000000; // sol is x 1E-9

  console.log(formatDate(new Date()) + ' ' + "MAX_SIZE_BASE = " + MAX_SIZE_BASE);
  console.log(formatDate(new Date()) + ' ' + "SPREAD_COEFF_OPTIMAL = " + SPREAD_COEFF_OPTIMAL);
  console.log(formatDate(new Date()) + ' ' + "OFFER_TOLERANCE = " + OFFER_TOLERANCE);

```
Much of this you will have seen before, if you read my previous examples. A few items worth spelling out, though:
1. MAX_SIZE_BASE = the amount of inventory that you want to trade for each order. This is a constant, and there will only be one bid and one ask open at a time (versus a more sophisticated mm bot that uses ladders, for example).
2. SPREAD_COEFF_OPTIMAL = the difference between your bid and ask prices (e.g. 0.005 = 0.5%) that we are targeting. Later in the program we will compute this using volatility, right now just initializing it.  
3. OFFER_TOLERANCE = the amount we will allow our open orders to deviate from the current optimal bid and ask prices, before we cancel them. (0.2 = 20%). This is a design choice to me made by the engineer.
4. BASE_BN_OFFSET = this is a number for converting the BN values in the serum market to a decimal. For ETH, you divide the BN (big number) by 1E8 (100_000_000) to get the decimal amount.
5. QUOTE_BN_OFFSET = this is a number for converting the BN values in the serum market to a decimal. For SOL, you divide the BN (big number) by 1E9 (1_000_000_000) to get the decimal amount.


#### Buffer Size: Volatility Sampling Window
On Solana, there are about 5760 slots per hour, or 1.6 slots per second. Pyth is designed to update prices every slot. So that means that if we are taking samples from Pyth on each update, we should expect our "sampling time" to match those of Pyth. Consequently, when targeting a specific time frame for calculating volatilty, say we want a 5-minute window, we'd need a buffer size of 5 min x 60 sec x 1.6 samples[slots] per sec = 480 samples. 

#### Safety Features: Control and Shut-Off by Volatility
As you can see, the optimal bid and ask prices are determined by the current market's volatility. So, 

#### Bot Warm-up and Operation
Depending on how wide you set the 
