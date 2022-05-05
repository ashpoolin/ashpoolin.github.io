---
layout: post
---

## How the f*** do you use the serum-ts client? (part 2)

In [part one](https://ashpoolin.github.io/how-tf-do-you-use-serum-ts-client) of this blog series, we mainly
discussed the various accounts and actions that were required to use the serum-ts API. If you are new to 
the library, and haven't read it, please go back as it will make dealing with accounts in serum much easier.

### The Making of a Market Maker

This next section is more like *cool, now what can we do with it?* It suffices to say that we need a project.
As I'm not one for taking large directional bets, it seemed appropriate to start with a simple market making bot.
A market maker provides liquidity to the market, and generates profit by setting prices such that there is a spread. 
The spread is the difference between the lowest ask and the highest bid prices. Another key feature of market makers
is that they try to avoid accumulating a net long (short) position in an asset, and thereby take on unwanted risk.
The goal is to stay market *neutral*, and is one of the ways a market making strategy differs from other algorithmic trading
strategies such as statistical arbitrage, wherein the trader may try to express a directional bet through their strategy.
The final dimension of good market making is for the maker to avoid "adverse selection," or a situation where it buys
(sells) to more informed sellers (buyers), thereby taking their *toxic flow* and essentially just getting the worst of it,
and reducing gains, or even turning a loss.

Launching into a full-featured mm bot is an ambitious affair. But how do you eat an elephant? One bite at a time. The idea 
here is to build up small components one-by-one, and then refine them as we identify problems with the design. Since 
I don't know how to do it any other way, we will 100% test in prod.

### Bot Concept

For our prototype, we'll be building something I call the "Greedy Bot"-- a naive market maker that elbows its way to the front of 
the order book, and tries to stay there. BEWARE THAT THERE IS NO PROOF THAT THIS IS AN INFORMED STRATEGY, OR EVEN A WISE THING TO DO.
It probably loses money, because it's just too damn simple. But this bot just wants to trade! 
And who knows, maybe it is effective in getting frequent, small-margin fills on markets for staked tokens (e.g. mSOL/SOL, stSOL/SOL, ...).

So here's how this simple bot works:
1. Pulls down the order book
2. Computes the spread, and retrieves the `minimum ask price (minAsk)` and `maximum bid price (maxBid)`
3. Sneakily increases (decreases) the price by one tick, and places an order to buy (sell) with *all* of the available tokens in the base or quote token accounts
4. Loops through, checking if its orders are at the front of the book. If they are not, it cancels and replaces them


### Setup 

You'll need to download the entire serum-ts repo, and if I recall you need to `yarn build` or `npm install` both the base directory,
as well as the [serum-ts/packages/serum](https://github.com/project-serum/serum-ts/tree/master/packages/serum) module's directory in order
for it all to work. I used yarn, it seemed to have less dependency issues and I think I got it to work quicker. You'll just have to play
around if you are having issues.

To avoid the time and additional costs it takes to settle tokens from the "unsettled" funds to your wallet's main address or SPL token accounts, 
the bot just keeps all of the tokens in the unsettled funds accounts. From the previous article, the unsettled balances can be found in the `base` and `quote` token accounts (more info on how to get that later). 

## Writing the Code

### Basic Program Structure
The outline for our naive maker bot is shown below. After examining the flow of the program, we'll go through each of the features in detail.

```
// --- IMPORTS --- //

// --- FUNCTIONS - TBD --- //

// --- MAIN LOOP ---//
async function main () {

    // --- INITIALIZATION & HOUSEKEEPING, SEE ABOVE --- //

    while(true) {
        // STEPS:
        // (1) CHECK SPREAD
        // (2) CANCEL STALE ORDERS
        // (3) FIND AVAILABLE TOKENS TO TRADE
        // (4) PLACE ORDERS
        // (5) TIME DELAY
    }
}
main();

```
You may find it bizarre or redundant to wrap the while loop in this main function. However, a fun quirk of Typescript is that for modules (any script that includes an import or export), calls to async functions (e.g. `await function(params) {...}` must not be done so at the top level of the program. So this C-inspired main function will encapsulates all of those async calls, and will enable us to port this code over to Typescript later. 

### Environment Initialization

First, we'll need to create the imports, and configure the connection, market, and wallet that we want to use. Something like this:
```
import {Keypair, Connection, PublicKey } from '@solana/web3.js';
import { Market } from '@project-serum/serum'; 
import fs from 'fs';

// --- FUNCTIONS - TBD --- //

async function main () {

    // Select an RPC
    // const connection = new Connection('https://solana-api.projectserum.com');
    
    // select the market... found in src/markets.json
    let marketAddress = new PublicKey('5cLrMai1DsLRYc1Nio9qMTicsWtvzjzZfJPXyAoF4t1Z'); // MSOL/SOL 
    let programAddress = new PublicKey('9xQeWvG816bUx9EPjHmaT23yvVM2ZWbrrpZb9PusVFin'); // serum v3 program ID
    const market = await Market.load(connection, marketAddress, {}, programAddress);
    
    // misc info market params
    const MIN_ORDER_SIZE = market.minOrderSize;
    const TICK_SIZE = market.tickSize;
    
    // load up your keypair 
    const owner = Keypair.fromSecretKey(
      Uint8Array.from(
        JSON.parse(
          fs.readFileSync(
            process.env.KEYPAIR,
            'utf-8',
          ),
        ),
      )
    );

    while(true) {
        // STEPS: 
        ...
    }
}
main();

```
Very good. A word about the keypair. You can do it several ways, but the easiest is just to load it from an environment variable
as I have done above. To do you can go to the console and just type `$ export KEYPAIR=/path/to/myKeypair.json` or alternately, 
do the following:
``` 
$ echo 'export KEYPAIR=/path/to/myKeypair.json' >> ~/.bashrc
$ source ~/.bashrc
```
Adding it to the `.bashrc` or `.bash_profile` files will ensure that your script can load the wallet for each session.

### Getting the Date

Most bots will need a concept of time, so that we can check the log file to see how and when we went horribly wrong. So here's some 
boiler plate code I nabbed from the interwebs, sorry, I don't recall the source. This will get added to the program above the main() function definition.
```
// date retrieval and formatting
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

// to use it:
const niceDate = formatDate(new Date());

```


### Finding the Spread

Now we're actually starting to build the bot. We'll need a way to get the state of the order book. 
For this we'll create an asynchronous function that pulls down the orderbook, and spits out the minimum ask, maximum bid, and their respective sizes. 
Remember, this is the "Greedy Bot," and it doesn't care much about the contours of the orderbook, just wants to be first in line. So here we go:

```
// returns the min_ask and max_bid values as an array
async function findSpread(market, connection) {
    console.log("retrieving prices...")
    
    let bids = await market.loadBids(connection);
    let asks = await market.loadAsks(connection);

    let minAskPrice = asks.getL2(1)[0][0];
    let minAskSize = asks.getL2(1)[0][1];
    let maxBidPrice = bids.getL2(1)[0][0];
    let maxBidSize = bids.getL2(1)[0][1];
    let spreadPercent = (minAskPrice - maxBidPrice) / maxBidPrice * 100;
    console.log(formatDate(new Date()) + ' ' + 'ask: ' + minAskPrice + ' ' + minAskSize + ' bid: ' + maxBidPrice + ' ' + maxBidSize + ' spread(%): ' + spreadPercent);

    return [minAskPrice, minAskSize, maxBidPrice, maxBidSize]
}

// to call it:
let [minAskPrice, minAskSize, maxBidPrice, maxBidSize] = await findSpread(market, connection);
```

So now we have a clumsy function that grabs computes the spread, but more importantly, returns the maximum bid and lowest ask (front of order book) values. We need this info so we can budge in line!

### Cancelling Stale Orders

Next up, we want to check if we have any open orders, and if they are not at the front of the order book, to cancel them. Here's the function:

```
// delete stale open orders if necessary
async function cancelStaleOrders(market, connection, owner, minAskPrice, minAskSize, maxBidPrice, maxBidSize) {
    try {
      console.log(formatDate(new Date()) + ' ' + 'finding open orders...')
      let myOrders = await market.loadOrdersForOwner(connection, owner.publicKey); 
      for (let order of myOrders) {
        console.log(formatDate(new Date()) + ' ' + 'order: ' + order.orderId + ' ' + order.side + ' ' + order.size + ' ' + order.price);
        
        // only cancel order if an order larger than yours is in front of you.
        if (order.side === 'buy' && order.price < maxBidPrice && order.size < maxBidSize) {
          console.log(formatDate(new Date()) + ' ' + "trying to cancel open buy order w/ size/price = " + order.size + '/' + order.price);
          const sigCancel = await market.cancelOrder(connection, owner, order);
          console.log(formatDate(new Date()) + ' ' + sigCancel);
        }
        else if (order.side === 'sell' && order.price > minAskPrice && order.size < minAskSize) {
          console.log(formatDate(new Date()) + ' ' + "trying to cancel open sell order " + order.size + ' ' + order.price);
          const sigCancel = await market.cancelOrder(connection, owner, order);
          console.log(formatDate(new Date()) + ' ' + sigCancel);
        }
      }
      console.log(formatDate(new Date()) + ' ' + 'delaying for 15 seconds to allow tokens to settle...');
      await new Promise(resolve => setTimeout(resolve, 15000));
      console.log(formatDate(new Date()) + ' ' + 'cancelOrders routine complete.');
    } catch (error) {
      console.error(error);
    }
  }

// call it like this:
await cancelStaleOrders(market, connection, owner, minAskPrice, minAskSize, maxBidPrice, maxBidSize);
```
Here, since it's plausible that that the `market.canceOrder(...)` call fails, we have wrapped the function in try-catch statements. Granted, there are safer ways to do this, and ways to retry with more determinism, I'm very lazy, so I've just opted to let the function completely fail and just loop back to try again later. You do you.

The key functionality of this function is that it takes in the details we retrieved from the order book in `findSpread(...)` and compares them against our open orders.
It says "if I have a buy (sell) order and the price is lower (higher) than the maxBidPrice (minAskPrice) from the order book, and its size is less than the order in front of it, then cancel the order." The reason we do it this way is that you may not want to cancel your order, if there's another tiny order in front of you, potentially with much worse bid (ask) price. So, the function evaluates all of your open orders, and cancels those that do not fit the criteria. Finally, because we want to place trades this cycle of the while loop, we insert a short, 15 seconds delay to give some time for the accounts to update.  

### Find Available Balances to Trade
Here, we want to know how many tokens are available to trade. The function just looks into the open order account and sees how many 'base' or 'quote' tokens are free  to trade. With that, the function returns an nested array, with each sub-array telling you what type of token balance it is (base or quote), the publicKey (format = BN) for the account, and the amount that is free. 

```
// return open order account public keys and corresponding free token balances
async function findAvailableBalances(market, connection, owner, MIN_ORDER_SIZE) {
  try {
    //spl-token accounts to which to send the proceeds from trades
    const base = await market.findBaseTokenAccountsForOwner(connection, owner.publicKey, true);
    const baseTokenAccount = base[0].pubkey;
    const quote = await market.findQuoteTokenAccountsForOwner(connection, owner.publicKey, true);
    const quoteTokenAccount = quote[0].pubkey;

    let availableBalances = []; 
    for (let openOrders of await market.findOpenOrdersAccountsForOwner(
      connection,
      owner.publicKey,
    )) {
      if (openOrders.baseTokenFree > MIN_ORDER_SIZE) {
        console.log(formatDate(new Date()) + ' ' + 'free base tokens: ' + openOrders.baseTokenFree)
        availableBalances.push(['base', baseTokenAccount, openOrders.baseTokenFree]);
      }
      if (openOrders.quoteTokenFree > MIN_ORDER_SIZE) {
        console.log(formatDate(new Date()) + ' ' + 'free quote tokens: ' + openOrders.quoteTokenFree)
        availableBalances.push(['quote', quoteTokenAccount, openOrders.quoteTokenFree]);
      }
    }
    return availableBalances;
  } catch (error) {
    console.error(error);
  }
}

// call it like this:
const freeTokenBalances = await findAvailableBalances(market, connection, owner, MIN_ORDER_SIZE);
```

It should be realized that there's no control over the order size, which is not really realistic for a good MM bot. But again, Greedy Bot just wants to trade, and if it has chips available, it will shove them onto the table.

### Place Orders
Now things get interesting. Our stale orders should have been cancelled, we know how much money is in our pocket, and we are in a place where we have enough info to create orders. So let's do it. We have just a basic function for placing the order:

```
async function placeOrders(market, connection, params) {
  try {
    console.log(formatDate(new Date()) + ' ' + 'placing order to ' + params.orderType + ' ' + params.side + ' ' + params.size + ' at ' + params.price);
    const sig = await market.placeOrder(
      connection,
      params,
    );
    console.log(formatDate(new Date()) + ' ' + sig);
  } catch (error) {
    console.error(error);
  }
}

// call it:
placeOrders(market, connection, params);
```
Not too bad, or too interesting. Now, in the main section of the while loop, we'll do some processing of the values, in order to set the payer account, get order side, size, and price all correct. Here's the sequence:

```
// (4) PLACE ORDERS
for (let account of freeTokenBalances) {
    if (account[0] === 'base') {
      const ask = minAskPrice - TICK_SIZE;
      const freeBaseTokens = account[2].toNumber() / 1000000000; // convert type BN to decimal here SOL = / 1E9, ETH = / 1E8, etc
      console.log(formatDate(new Date()) + ' ' + 'free base tokens: ' + freeBaseTokens);
      const size = Math.round(freeBaseTokens / MIN_ORDER_SIZE) * MIN_ORDER_SIZE;
      if (size >= MIN_ORDER_SIZE) {
        const params = {
          owner: owner,
          payer: account[1],
          side: 'sell',
          price: ask,
          size: size,  
          orderType: 'limit',
          selfTradeBehavior: 'decrementTake', 
        };
        await placeOrders(market, connection, params);
      }
      else {
        console.log(formatDate(new Date()) + ' ' + 'no base tokens available for trading.')
      }
    }
    else if(account[0] === 'quote') {
      const bid = maxBidPrice + TICK_SIZE;
      const freeQuoteTokens = new Number(account[2].toNumber() / 1000000000);
      console.log(formatDate(new Date()) + ' ' + 'free quote tokens: ' + freeQuoteTokens);
      const size = Math.round((freeQuoteTokens / bid) / MIN_ORDER_SIZE) * MIN_ORDER_SIZE;
      if (size >= MIN_ORDER_SIZE) {
        const params = {
          owner: owner,
          payer: account[1],
          side: 'buy',
          price: bid,
          size: size, 
          orderType: 'limit',
          selfTradeBehavior: 'decrementTake', 
        }; 
        await placeOrders(market, connection, params);
      }
      else {
        console.log(formatDate(new Date()) + ' ' + 'no quote tokens available for trading.')
      }
    }
}
```
There's a lot going on here, but we'll go through it step-by-step. Using the nested arrays returned from our `findAvailableTokens(...)` function, we can form orders to use the tokens in those accounts. We start with the base tokens (the left half of the pair, e.g. MSOL in MSOL/SOL). 

#### Bid / Ask price
`const ask = minAskPrice - TICK_SIZE;`

Narrating the process, it says if we have free token in a `base` account, create an `ask` price equal to the `minAskPrice` [previously obtained from the order book] minus the `TICK_SIZE`. So we drop our order right below theirs. Similarly a `bid` will be `maxBidPrice` plus the `TICK_SIZE`, and it's a tick above the current best bid price.

#### freeBaseTokens / freeQuoteTokens
`const freeBaseTokens = account[2].toNumber() / 1000000000;`
The serum-js library returns the amounts in these token accounts as a BN, and there is an offset. So we first convert the BN to a number, then divide it by the offset/multiplier.
For SOL it is x 1,000,000,000, for ETH it is 100,000,000. For MSOL I think it's also 1E9, and I don't really know about other SPLs. it's not written down anywhere, as far as I can tell. You'll just have to summon the `openOrders.baseTokenFree` and compare it against an account value that is known to you in order to calculate the multiplier. Anyway, that's what's going on here.

#### size
`const size = Math.round(freeBaseTokens / MIN_ORDER_SIZE) * MIN_ORDER_SIZE;`

Here we are creating an order with a size that is compatible with the MIN_ORDER_SIZE that is determined by the market. We retrieved this information earlier, by calling `const MIN_ORDER_SIZE = market.minOrderSize;`. We are doing a crude rounding routine, wherein we cleave off significant digits that are below the `MIN_ORDER_SIZE` by first dividing by it to get a scaled up number, rounding off the remaining decimals, and then multiplying again by `MIN_ORDER_SIZE` to get an acceptable value for `size`. Finally, there's a if/else check that ensures that we don't try to place $0 value orders, that looks like this `if (size >= MIN_ORDER_SIZE) {...}`.

#### params
We have created each of the dimensions of our order, and now we must assemble them. These get placed into a structure called `params`:
```
const params = {
  owner: owner,
  payer: account[1],
  side: 'sell',
  price: ask,
  size: size,  
  orderType: 'limit',
  selfTradeBehavior: 'decrementTake', 
};
```
First up is `owner`. This is a `Keypair` type. Let it be know that serum-ts library was constructed using the deprecated `Account` type, so you'll probably need to include a `//@ts-ignore` above the `params` definition while using Typescript. Since we are using unsettled `base` and `quote` and quote token accounts to source the funds, the `payer` is merely the base or quote token accounts accompanying the correct side of the order; e.g. free base tokens are used for a `'sell'` order with accompanying `ask` price, free quote token used for a `buy` with accompanying `bid` price. We are only doing limit `orderType` as this is a market-maker bot, after all. FYI I don't really know what `selfTradeBehavior` does, but let's just presume it's not going to happen to us (hopefully). You can probably omit it, too.

#### placeOrder
That's it! We've constructed our params, and the last thing we need to do is call it: `placeOrders(market, connection, params);`.

### Delay / Wait
Using an on-chain CLOB is different than just calling a CEX's API, because there are transaction ["gas"] costs to create / cancel orders, even if those order aren't filled. Additionally, CEX APIs have near-immediate response times, while it may take awhile for a transaction to be confirmed by the validators for a blockchain. Because we want to limit how often we trade, and reduce our cost, we set a delay here, but this is totally up to the user to decide what to do. Here's a simple call that I've been using to force the program to wait: `await new Promise(resolve => setTimeout(resolve, 60000));`

### Complete Script
Here's the completed program for copy-pasta purposes:
```
import {Keypair, Connection, PublicKey } from '@solana/web3.js';
import { Market } from '@project-serum/serum'; 
import fs from 'fs';

// date retrieval and formatting
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

// returns the minAskPrice and maxBidPrice values as a tuple
async function findSpread(market, connection) {
    console.log("retrieving prices...")
    
    let bids = await market.loadBids(connection);
    let asks = await market.loadAsks(connection);

    let minAskPrice = asks.getL2(1)[0][0];
    let minAskSize = asks.getL2(1)[0][1];
    let maxBidPrice = bids.getL2(1)[0][0];
    let maxBidSize = bids.getL2(1)[0][1];
    let spreadPercent = (minAskPrice - maxBidPrice) / maxBidPrice * 100;
    console.log(formatDate(new Date()) + ' ' + 'ask: ' + minAskPrice + ' ' + minAskSize + ' bid: ' + maxBidPrice + ' ' + maxBidSize + ' spread(%): ' + spreadPercent);

    return [minAskPrice, minAskSize, maxBidPrice, maxBidSize]
}

// delete stale open orders if necessary
async function cancelStaleOrders(market, connection, owner, minAskPrice, minAskSize, maxBidPrice, maxBidSize) {
    try {
      console.log(formatDate(new Date()) + ' ' + 'finding open orders...')
      let myOrders = await market.loadOrdersForOwner(connection, owner.publicKey); 
      for (let order of myOrders) {
        console.log(formatDate(new Date()) + ' ' + 'order: ' + order.orderId + ' ' + order.side + ' ' + order.size + ' ' + order.price);
        
        // only cancel order if an order larger than yours is in front of you.
        if (order.side === 'buy' && order.price < maxBidPrice && order.size < maxBidSize) {
          console.log(formatDate(new Date()) + ' ' + "trying to cancel open buy order w/ size/price = " + order.size + '/' + order.price);
          const sigCancel = await market.cancelOrder(connection, owner, order);
          console.log(formatDate(new Date()) + ' ' + sigCancel);
        }
        else if (order.side === 'sell' && order.price > minAskPrice && order.size < minAskSize) {
          console.log(formatDate(new Date()) + ' ' + "trying to cancel open sell order " + order.size + ' ' + order.price);
          const sigCancel = await market.cancelOrder(connection, owner, order);
          console.log(formatDate(new Date()) + ' ' + sigCancel);
        }
      }
      console.log(formatDate(new Date()) + ' ' + 'delaying for 15 seconds to allow tokens to settle...');
      await new Promise(resolve => setTimeout(resolve, 15000));
      console.log(formatDate(new Date()) + ' ' + 'cancelOrders routine complete.');
    } catch (error) {
      console.error(error);
    }
  }

  // return open order account public keys and corresponding free token balances
async function findAvailableBalances(market, connection, owner, MIN_ORDER_SIZE) {
    try {
      //spl-token accounts to which to send the proceeds from trades
      const base = await market.findBaseTokenAccountsForOwner(connection, owner.publicKey, true);
      const baseTokenAccount = base[0].pubkey;
      const quote = await market.findQuoteTokenAccountsForOwner(connection, owner.publicKey, true);
      const quoteTokenAccount = quote[0].pubkey;
  
      let availableBalances = []; 
      for (let openOrders of await market.findOpenOrdersAccountsForOwner(
        connection,
        owner.publicKey,
      )) {
        if (openOrders.baseTokenFree > MIN_ORDER_SIZE) {
          console.log(formatDate(new Date()) + ' ' + 'free base tokens: ' + openOrders.baseTokenFree)
          availableBalances.push(['base', baseTokenAccount, openOrders.baseTokenFree]);
        }
        if (openOrders.quoteTokenFree > MIN_ORDER_SIZE) {
          console.log(formatDate(new Date()) + ' ' + 'free quote tokens: ' + openOrders.quoteTokenFree)
          availableBalances.push(['quote', quoteTokenAccount, openOrders.quoteTokenFree]);
        }
      }
      return availableBalances;
    } catch (error) {
      console.error(error);
    }
  }

async function main () {

    // Select an RPC
    // const connection = new Connection('https://solana-api.projectserum.com');
    const connection = new Connection('https://ssc-dao.genesysgo.net/');

    // select the market... found in src/markets.json
    let marketAddress = new PublicKey('5cLrMai1DsLRYc1Nio9qMTicsWtvzjzZfJPXyAoF4t1Z'); // MSOL/SOL 
    let programAddress = new PublicKey('9xQeWvG816bUx9EPjHmaT23yvVM2ZWbrrpZb9PusVFin'); // serum v3 program ID
    const market = await Market.load(connection, marketAddress, {}, programAddress);

    // misc info market params
    const MIN_ORDER_SIZE = market.minOrderSize;
    const TICK_SIZE = market.tickSize;

    // load up your keypair 
    const owner = Keypair.fromSecretKey(
      Uint8Array.from(
        JSON.parse(
          fs.readFileSync(
            process.env.KEYPAIR,
            'utf-8',
          ),
        ),
      )
    );

    while(true) {
        // STEPS:
        // (1) CHECK SPREAD
        let [minAskPrice, minAskSize, maxBidPrice, maxBidSize] = await findSpread(market, connection);
        
        // (2) CANCEL STALE ORDERS
        await cancelStaleOrders(market, connection, owner, minAskPrice, minAskSize, maxBidPrice, maxBidSize);

        // (3) FIND AVAILABLE TOKENS TO TRADE
        const freeTokenBalances = await findAvailableBalances(market, connection, owner, MIN_ORDER_SIZE);

        // (4) PLACE ORDERS
        for (let account of freeTokenBalances) {
            if (account[0] === 'base') {
              const ask = minAskPrice - TICK_SIZE;
              const freeBaseTokens = account[2].toNumber() / 1000000000; // convert type BN to decimal here SOL = / 1E9, ETH = / 1E8, etc
              console.log(formatDate(new Date()) + ' ' + 'free base tokens: ' + freeBaseTokens);
              const size = Math.round(freeBaseTokens / MIN_ORDER_SIZE) * MIN_ORDER_SIZE;
              if (size >= MIN_ORDER_SIZE) {
                const params = {
                  owner: owner,
                  payer: account[1],
                  side: 'sell',
                  price: ask,
                  size: size,  
                  orderType: 'limit',
                  selfTradeBehavior: 'decrementTake', 
                };
                await placeOrders(market, connection, params);
              }
              else {
                console.log(formatDate(new Date()) + ' ' + 'no base tokens available for trading.')
              }
            }
            else if(account[0] === 'quote') {
              const bid = maxBidPrice + TICK_SIZE;
              const freeQuoteTokens = new Number(account[2].toNumber() / 1000000000);
              console.log(formatDate(new Date()) + ' ' + 'free quote tokens: ' + freeQuoteTokens);
              const size = Math.round((freeQuoteTokens / bid) / MIN_ORDER_SIZE) * MIN_ORDER_SIZE;
              if (size >= MIN_ORDER_SIZE) {
                const params = {
                  owner: owner,
                  payer: account[1],
                  side: 'buy',
                  price: bid,
                  size: size, 
                  orderType: 'limit',
                  selfTradeBehavior: 'decrementTake', 
                }; 
                await placeOrders(market, connection, params);
              }
              else {
                console.log(formatDate(new Date()) + ' ' + 'no quote tokens available for trading.')
              }
            }
        }

        // (5) TIME DELAY
        console.log(formatDate(new Date()) + ' ' + 'Routine complete. Delaying for 60 seconds...');
        await new Promise(resolve => setTimeout(resolve, 60000));
    }
}

main();
```

### Running the Program
Now we have a completed program. you can run it directly from the serum directory (`serum-ts/packages/serum`) using `$node greedy.js`. Inevitably, your bot may crash. I have a lazy (I think clever solution for that). We accomplish this, in addition to creating a process in the background and a logfile at `nohup.out`, using this simple shell script I've concocted:
```
$ nano launcher.sh

### COPY PASTE THIS INTO launcher.sh

#!/bin/bash
while :
do
    status=`ps -ef | grep greedy.js | grep node | wc -l`
    echo "bot status is: ${status}"

    if [ $status -eq 0 ]; then
        echo "starting greedy.js"
        nohup node ironman.js & 
    else
        echo "process is healthy."
    fi
    
    sleep 10;

done

### ctrl + C, then Y + enter to save and exit

### make the script executable
$ chmod +x launcher.sh

### use the launcher to manage the script, and restart it if it fails!

$ ./launcher.sh

### follow along by reading the tail of the nohup.out logfile
$ tail -f nohup.out
### ctrl + C to exit

```
With our `launcher.sh` shell script, we can automatically kickstart our bot if it happens to go offline. Since state is entirely housed on the blockchain, there is no risk in losing state from your script, and I believe that this is a perfectly sane and simple thing to do.

### User Interface
Having a bot is all well and good, but sometimes you need a separate set of eyes your system, and surroundings. For that there used to be a serum front-end but it seems like they took it down. [https://dex.raydium.io/](https://dex.raydium.io/) may be useful to you. There are probably others (open serum?). Just navigate to the market that you are using and should be all set. 

### Closing Thoughts 
What a journey! You'd think such a dumb bot would have fewer lines of code, and be simpler. Again, I want to warn you that this bot very likely will not make money--there are even more factors to consider which we haven't addressed, which are not limited to:
- "true" asset prices
- inventory management
- adverse selection / toxic order flow
- code efficiency / timing analysis and optimization

Albeit limited, this "bot" should hopefully give you a greater idea of how to move around the serum-ts library, interact with the CLOB, and express your own trading ideas.

This is part 2 of a blog series I call "How the f*** do you use the serum-ts client?" If this was helpful to you, please go ahead and follow me on Twitter, and let me know that you liked it. Thanks, and happy trading!