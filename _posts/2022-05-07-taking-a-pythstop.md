---
layout: post
---

## How the f*** do you use the serum-ts client? (part 3): Taking a Pythstop

No bot strategy is complete without having access to market data. For someone trading on-chain, the question is: "where do I get market data?" Fortunately, there's an app for that, and for Solana, it comes from the [Pyth Network](https://pyth.network/). In this part of the series, we'll show how to get Pyth set up, and how to make it play nice with the serum-ts library.

### What is Pyth?

First, what is an oracle? My limited knowledge on the subject says that an oracle is an aggregator of data, bringing the "true" value of assets on-chain. In Pyth's case, it is described as a two-sided marketplace, bringing together both publishers and consumers of data. The publishers provide data that they have the rights to, and are rewarded for it. The outcome is that consumers are able to access this fast, live data, for free! The system is designed to be open and available to everyone, which is a departure from much of the tradfi and high-frequency trading world. While it may be perceived that the largest market makers publishing pricing data could be a conflict of interest, Pyth brings together many adversarial firms who are each publishing their own data, and the network itself uses staking and slashing to punish publishers who produce slow, incomplete, or inaccurate data. This "Byzantine data" publishing scheme is one, if not the first of its kind, and is quite remarkable. With each publisher incentivized to contribute for both rewards, and to prevent Sybil-type attacks from rivaling firms against their own investments, it is likely that these prices do converge on an honest result. What this means for the data consumer is that the data obtained from the oracle should be as close to "true" as one could hope. That said, it's important to DYOR, so I will be looking into the data aggregation methods, and if my opinion changes on this I will be sure to make my opinion known.

For more info, please have a look at the docs: [How Pyth Works](https://docs.pyth.network/how-pyth-works)

### For Consumers

There are many protocols both on Solana and Terra that are actively using Pyth data feeds. To check out your options, I'd recommend going here: (Consume Data)[https://docs.pyth.network/consume-data/solana].

While I have designs on writing future systems using Rust (which Pyth has a library for), I'm taking the easy route here and we're going to use it's [JS/TS Client](https://www.npmjs.com/package/@pythnetwork/client).

### Our Project

Since I intend to use this with the serum-ts orderbook client, I've gone ahead and navigated to the directory `serum-ts/packages/serum`. I have written about how to get this client running in previous blog posts, but you can get the code from the Project Serum Github [here](https://github.com/project-serum/serum-ts).

Next, we're just going to add the Pyth package to our project:
```
# npm

$ npm install --save @pythnetwork/client

# Yarn

$ yarn add @pythnetwork/client

```

I've opted to use Yarn over npm, since it just seems to work better for me, and I'm a touch superstitious. 

I think that while the above adds the various dependencies you'll need to use Pyth, I think that you'll want the components from the Pyth examples, so go ahead and grab them from the `src/` folder of the Pyth client's [github repo](https://github.com/pyth-network/pyth-client-js).

### Typescript, ugh!

What you quickly see from the pyth source files in `src/` is that, although the client is "javascript-friendly" it is written in Typescript. This puts the onus on us to reconcile Typescript compiler hell. While the repo in itself isn't a problem if you clone it and use it out of the box, I found that there were conflicts between serum and pyth in that serum expects to use `"type" : "module"` in the `package.json` file, while pyth implicitly used `"commonjs"` (feel free to correct me if I am wrong). Now, I am no expert on typescript but it appears I am found a configuration that gets the two clients to play nice, so I'll share them here.

In `package.json` (from the `serum-ts/packages/serum`) project folder, we modify the file as follows:

```
{
  "name": "@project-serum/serum",
    ...
  "type": "module",
  },
  "scripts": {
    "build": "tsc",
    "build_modules": "tsc --module es6",
    ...
  },
  "devDependencies": {
    ...
    "typescript": "^4.7.0-dev.20220425"
  },
  "dependencies": {
    "@babel/types": "^7.17.0",
    "@project-serum/anchor": "^0.11.1",
    "@project-serum/serum": "^0.13.64",
    "@pythnetwork/client": "^2.6.1",
    ...
    "@solana/web3.js": "^1.39.1",
  },
  ...
}
```

Again, here we are using `"type" : "module"` to play nice with Serum, which we want to use. 

Next, we update the `tsconfig.json` file:

```
{
  ...
  "compilerOptions": {
    "moduleResolution": "nodenext",
    "outDir": "./lib",
    "target": "es6",
    "module": "esnext",
    "lib": ["es2019"],
    "allowJs": true,
    "checkJs": true,
    "declaration": true,
    "declarationMap": true,
    "noImplicitAny": false,
    "resolveJsonModule": true,
    "sourceMap": true,
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true
  },
  "include": ["./src/**/*"],
  "exclude": ["./src/**/*.test.js", "node_modules", "**/node_modules"]
}

```

I've included it in its entirety because I don't recall exactly which options caused problems. We've significantly relaxed to compiler options, to tolerate a lot of sloppy behavior, not limited to allowing implicit `any` type as well as allowing vanilla javascript. Now, once we've written our program, we can compile ("transpile", ackshually) by running `yarn build` and we should get working code (beware that there are errors, which I'll explain below).

A new issue crops up in that the latter versions of `es6` or `esnext` possibly reltaed to adding the option `"moduleResolution: "nodenext"` required me to go back through some of the source files and update the imports for explicit paths. This was a little troubling, but resolved rather quickly by hitting `yarn build` and the just tracking down the broken paths related to imports as they cropped up. An example of this shows up in the pyth `src/index.ts` file, which you'll need to update the import as follows:
```
// /src/index.ts

// must change from:
import { readBigInt64LE, readBigUInt64LE } from './readBig'

// to:
import { readBigInt64LE, readBigUInt64LE } from './readBig.js'

```
This is just an example, and non-exhaustive, but sniff around for any other issues like this, and your program will be able to run safely soon. In the program portion of this write-up I'll try to use the imports correctly, for your sake.

### The Code

We took a circuitous path to get here, but the point is to get Pyth into a usable form so that we can interact with the serum CLOB. It would be easier to just clone the Pyth github if you want to see it work, out of the box. Regardless, we're finally ready to write some code. To invoke the Pyth oracle, what better way than to heavily plagiarize some examples?

Take a look at the [Readme.md](https://github.com/pyth-network/pyth-client-js/blob/main/README.md) file for Pyth to familialize yourself with the example code. I've copied the basic structure here:

```
const pythConnection = new PythConnection(solanaWeb3Connection, getPythProgramKeyForCluster(solanaClusterName))
pythConnection.onPriceChange((product, price) => {
  // sample output:
  // Crypto.SRM/USD: $8.68725 ±$0.0131 Status: Trading
  console.log(`${product.symbol}: $${price.price} \xB1$${price.confidence} Status: ${PriceStatus[price.status]}`)
})

// Start listening for price change events.
pythConnection.start()
```

For thoroughness, let's expand this, while also trying to obtain some useful pieces of information. For this example, we'll first calculate the price of mSOL/SOL, and then calculate the current premium (or discount). Then, we'll write another dumb script to calculate a simple moving average, to provide some smoothing to our data.

First, let's set up our connection to Pyth. Create a new file `src/pythy.ts`.

```
// imports
import {Cluster, Connection } from '@solana/web3.js';
import { PythConnection } from './PythConnection.js';
import { getPythProgramKeyForCluster } from './cluster.js'; //

// create a connection
let connection = new Connection('https://solana-api.projectserum.com/');

// pyth connection
const SOLANA_CLUSTER_NAME: Cluster = 'mainnet-beta'
const pythPublicKey = getPythProgramKeyForCluster(SOLANA_CLUSTER_NAME)
const pythConnection = new PythConnection(connection, pythPublicKey)
```

That's all the plumbing we'll need to get the data feed. Beware the import `... from ./cluster.js` which calls the module with a `.js` extension, as this is required for our compiler configuration (`es6` compatibility, I believe).

Now we're set up to receive data. Let's start simple:
```
let priceLatestMSOL: number = 100; // random initialization value

pythConnection.onPriceChange((product, price) => {
  if (price.price && price.confidence) {
    if (product.symbol === "Crypto.MSOL/USD") {
      
      // store the price in the variable for later
      priceLatestMSOL = price.price;
        
      // tslint:disable-next-line:no-console
      console.log(product.symbol + ", " + price.price + ", " + price.confidence);
    }
  } 
})
pythConnection.start()
```
That's all we need to get a price for our token (mSOL shown here)! I am aware of the impropriety of using number types for some things, while not explicitly labeling others. We set up our compiler to tolerate Javascript, and we are being lazy. I'm basically doing the minimum to prevent compilation errors, but not much else. For things like numbers, though, you can get all sorts of problems if you are using the incorrect type, so here I've labeled explicitly. I encourage you to be a strict as your style and skills allow. These scripts are for recreational use only!

Alright, so we have a price for mSOL. Now let's get it for SOL, and then calculate the mSOL premium:
```
const MSOL_PER_SOL: number = 1.04516;
let priceLatestMSOL: number = 100;
let priceLatestSOL: number = 95.66;
let msolPerSolMarket: number = priceLatestMSOL / priceLatestSOL;
let msolPremium: number = ((priceLatestMSOL / MSOL_PER_SOL) - priceLatestSOL) / priceLatestSOL * 100; // as a percentage 

pythConnection.onPriceChange((product, price) => {
  if (price.price && price.confidence) {
    if (product.symbol === "Crypto.MSOL/USD") {
      priceLatestMSOL = price.price;
      msolPerSolMarket = priceLatestMSOL / priceLatestSOL;
      msolPremium = ((priceLatestMSOL / MSOL_PER_SOL) - priceLatestSOL) / priceLatestSOL * 100;
      // tslint:disable-next-line:no-console
      console.log(msolPerSolMarket + ', ' + msolPremium + ', ' + product.symbol + ", " + price.price + ", " + price.confidence);
    }
    if (product.symbol === "Crypto.SOL/USD") {
      priceLatestSOL = price.price;
      msolPerSolMarket = priceLatestMSOL / priceLatestSOL;
      msolPremium = ((priceLatestMSOL / MSOL_PER_SOL) - priceLatestSOL) / priceLatestSOL * 100;
      // tslint:disable-next-line:no-console
      console.log(msolPerSolMarket + ', ' + msolPremium + ', ' + product.symbol + ", " + price.price + ", " + price.confidence);
    }
  } 
});

pythConnection.start()
```

Here, we've declared variables that are in the outer-most scope, so that they can be accessed within the function. There two basic formulas for calculating the market value of mSOL:SOL, then the discount/premium, and it just spits out comma-delimited text that can be saved to a `.csv` file. Pretty amateur stuff, but we are officially using Pyth!

To run the program, you just need to compile and run:
```
$ yarn build

$ node lib/pythy.js
```

Now, as you run this, you'll see that even after the initialization constants have been overwritten with real data, you can see our calculated rate for MSOL/SOL (`msoPerSolMarket`) and the premium (`msolPremium`) values are quite noisy. We are receiving tick-by-tick data (that is, every change of the price in either mSOL or SOL updates the calculated values), so as prices change it causes our figures to bounce around. So let's implement a simple moving average to clean it up--it really won't take a lot.

First a little algebra. The `msolPremium` value was previously calculated as:

`msolPremium = ((priceLatestMSOL / MSOL_PER_SOL) - priceLatestSOL) / priceLatestSOL * 100;` 

Dividing through by `priceLatestSOL`, we have:

`msolPremium = ( ((priceLatestMSOL/priceLatestSOL) / MSOL_PER_SOL) - 1 ) * 100;`

We see that `priceLatestMSOL/priceLatestSOL` is just `msolPerSolMarket` which we can substitute as:

`msolPremium = ( (msolPerSolMarket / MSOL_PER_SOL) - 1 ) * 100;`

Now, we only have to apply our simple moving average to `msolPerSolMarket` to receive effect of smoothing in our `msolPremium` calculation as well. The formula will look something like this:

`msolPremiumSMA = ( (msolPerSolMarketSMA / MSOL_PER_SOL) - 1 ) * 100;`

We're ready to create a function for the simple moving average. Here it is:
```
const reducer = (accumulator, curr) => accumulator + curr;

function simpleMovingAverage (prices: number[], newVal: number, arrLength: number) {
  if(prices.length >= arrLength) {
    prices.shift();
  }
  prices.push(newVal);
  let average: number = prices.reduce(reducer) / prices.length;
  return [prices, average];
}

```
From the function, we see we need to pass in an array of `prices`, a new value `newVal` to add to the prices array, and `arrLength`, the number of samples we want to store for the calculation. The logic is reasonably simple:
1. Look at the array, and if the length is greater than or equal to the desired length, we need to 'pop' a value out of it, before adding our new one. this is done by the `shift()` function.
2. Next we add the new price value to the buffer
3. Here's the tricky part: we calculate the average by using an anonymous helper function called `reducer` which merely sums all of the values in the array. It's quirky, and not immediately clear why it works, but this is apparently how you do that in Javascript land. Divide the sum by the length of the array to get the average.
4. Finally, we return the updated prices buffer, and the now updated simple moving average.

Now, with our shiny new SMA calculator, we can roll out the full implementation:

```

// initialize variables
const MSOL_PER_SOL: number = 1.04516;
let priceLatestMSOL: number = 100;
let priceLatestSOL: number = 95.66;
let msolPerSolMarket: number = priceLatestMSOL / priceLatestSOL;
let msolPremium: number = ((priceLatestMSOL / MSOL_PER_SOL) - priceLatestSOL) / priceLatestSOL * 100; // as a percentage
// moving average-related
const BUFFER_SIZE: number = 10
let msolPerSolMarketBuffer: number[] = [msolPerSolMarket];
let msolPerSolMarketSMA: number = msolPerSolMarket;
let msolPremiumSMA: number = msolPremium;


pythConnection.onPriceChange((product, price) => {
  if (price.price && price.confidence) {
    if (product.symbol === "Crypto.MSOL/USD") {
      priceLatestMSOL = price.price;
      msolPerSolMarket = priceLatestMSOL / priceLatestSOL;
      //@ts-ignore
      [msolPerSolMarketBuffer, msolPerSolMarketSMA] = simpleMovingAverage(msolPerSolMarketBuffer, msolPerSolMarket, BUFFER_SIZE);
      msolPremiumSMA = ( (msolPerSolMarketSMA / MSOL_PER_SOL) - 1 ) * 100;
        
      // tslint:disable-next-line:no-console
      console.log(msolPerSolMarketSMA + ', ' + msolPremiumSMA + ', ' + product.symbol + ", " + price.price + ", " + price.confidence);
    }
    if (product.symbol === "Crypto.SOL/USD") {
      priceLatestSOL = price.price;
      msolPerSolMarket = priceLatestMSOL / priceLatestSOL;
      //@ts-ignore
      [msolPerSolMarketBuffer, msolPerSolMarketSMA] = simpleMovingAverage(msolPerSolMarketBuffer, msolPerSolMarket, BUFFER_SIZE);
      msolPremiumSMA = ( (msolPerSolMarketSMA / MSOL_PER_SOL) - 1 ) * 100;
      
      // tslint:disable-next-line:no-console
      console.log(msolPerSolMarketSMA + ', ' + msolPremiumSMA + ', ' + product.symbol + ", " + price.price + ", " + price.confidence);
    }
  } 
});

pythConnection.start()
```
There you have it! We're now logging data using Pyth, and creating meaningful metrics to use in whatever.

### Summary
In this third article in the series "How the f*** do you use the serum-ts client?" it was all about using Pyth's on-chain data. We made some configuration modifications to allow the Pyth client to be added to the serum package of the serum-ts library, which will allow us to use the data in data collection, research, and live trading strategies. 

If you liked this, please follow me on Twitter and let me know. Your feedback, errata, and suggestions are greatly appreciated and encouraged. Thank you.

-Ash



