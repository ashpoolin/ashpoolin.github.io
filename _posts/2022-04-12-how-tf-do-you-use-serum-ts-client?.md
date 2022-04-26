---
layout: post
---

## How the f*** do you use the serum-ts client? (part 1)

### Resources
Link to required source code / tools: 
- [serum-ts/packages/serum](https://github.com/project-serum/serum-ts/tree/master/packages/serum)
- [Solana CLI](https://docs.solana.com/cli)

### Disclaimer
Legal: Serum order book is an open order book that enables trading of digital assets. This article is produced as educational content, and an exploration of the technical features of the order book. It is not an inducement to invest or buy digital assets or cryptocurrencies. Please abide by all applicable laws. 

Liability: The article is bound to be riddled with mistakes, please help correct them using a PR through github, or sending a message. There are no warranties express or implied for this content, and the author will claim no responsibility for any person's loss of funds while using information here, or while trading. 

### Background
I've been in the solana ecosystem for a bit of time now,
and one of the first things that drew me into the space was the idea of
creating bots and interacting with the Serum order book directly
through the crappy scripts that I write. As it would turn out, unfortunately,
it was anything but easy for me. I've had to get over a general
learning curve in the solana programming model, understanding accounts, etc, 
as well as becoming hunt-and-peck proficient with Typescript. And despite
its widespread use, there doesn't seem to be any documentation on how it works,
besides a brief example in the README file, and the code itself.
That should not deter you from experimenting with the serum order book, however, as
being able to use it directly from the command line is a most empowering skill.

### What is Project Serum? 
Besides the token, it is--in my opinion--the most novel and
interesting piece of infrastructure in defi around. Seriously. Solana's high transaction rate (tx/sec, or TPS)
would not be nearly as interesting without it. Serum is a central limit order book (CLOB) that 
is the substrate for nearly all trading on Solana. Because it is open by design, it allows 
users and programs to interact directly with it, and countless protocols use it
as the infrastructure to create markets (trading pairs), access liquidity, and handle
the gory details behind custodying orders, and settling funds. In a word, Serum defines 
*composability* for Solana. Integrations with the Serum platform continue to climb,
and if you include all the DEXes built on top of it, it's near the top 5 by daily volume.
To summarize, Serum is not to be ignored, and trying to master it is an honorable goal, and 
100% worth your time. Ok, enough preamble.

### How does serum work? 
To my knowledge (yeah, there's going to be some mistakes here), "Serum"
is an assemblage of on-chain programs that each comprise a market. There's a
market address, and a program address, that represent a single trading pair (e.g. SOL/USDC, RAY/SRM).
The market data can be found in the current  `serum-ts/packages/serum/src/markets.json` file, and each
look like this:

```
...
  {
    "address": "7xMDbYTCqQEcK2aM9LbetGtNFJpzKdfXzLL5juaLh4GJ", // market address
    "deprecated": true, // I don't know why so many say they are deprecated --> they still work
    "name": "SOL/USDC",
    "programId": "EUqojwWA2rd19FZrzeBncJsm38Jm1hEhE3zsmX3bRc2o" // program address
  },
...
```

#### Configure your client for a Market
Preparing your environment for a market is quite straightforward:

```
// Create a connection. Note: choose your RPC node carefully, don't be a dick!
let connection = new Connection('https://api.mainnet-beta.solana.com');

let marketAddress = new PublicKey('7xMDbYTCqQEcK2aM9LbetGtNFJpzKdfXzLL5juaLh4GJ'); // sol-usdc
let programAddress = new PublicKey('EUqojwWA2rd19FZrzeBncJsm38Jm1hEhE3zsmX3bRc2o'); // sol-usdc

const market = await Market.load(connection, marketAddress, {}, programAddress);
```

#### Accounts - General

At its core, Serum is a system that allows trading of SPL tokens (Solana's version of an ERC-20, more or less).
In addition to the market-related accounts named above, Serum relies on a set of accounts that follow:
- owner
- payer
- SPL token account
- base token account
- quote token account
- open order accounts

According to my understanding, we'll enumerate each of these accounts now.

#### Owner
This is the main address of your solana wallet. With the [Solana CLI](https://docs.solana.com/cli) tools installed, you can create a wallet if you don't already have one:
```
$ solana-keygen new --outfile serum.json --no-bip39-passphrase
```
Then grab the address:
```
$ solana address --keypair serum.json
``` 
The base58 public key is your `owner` address.

We must load the owner account first, before we can sign any transactions, e.g. place/cancel orders, or settle funds. Some methods involve creating a .env file and reading that in, but I have found this method works too. Let's do that now:
```
import fs from 'fs';

// load up your keypair
const owner = Keypair.fromSecretKey(
  Uint8Array.from(
    JSON.parse(
      fs.readFileSync(
        '/home/<username>/.config/solana/id.json',
        'utf-8',
      ),
    ),
  )
);
```
For key safety, maybe use a different path, and/or follow someone else's example ;)

#### Sidebar: settled versus unsettled funds
Serum makes use of intermediate accounts (`base token account` and `quote token account`) while your funds are in use, but not currently allocated to an open order. If you have coins in one of these accounts, e.g. after you have had a fill on an order, your balance will be `unsettled`. Coins that are in your wallet are `settled` and the act of "settling funds" takes your coins from these intermediate accounts and either deposits them in an SPL account or at your main SOL address (`owner`).  

#### Payer
The payer is the source of the funds sold (e.g. sell USDC to buy SOL in SOL/USDC 'buy' order). But there is actually a lot of ambiguity around *which* account is the payer, for a given circumstance. This is because Serum makes use of intermediate accounts to hold unsettled balances. For instance, if you place an order and it is filled, it will be placed into one of these unsettled accounts. These accounts are the `base token account` and `quote token account` (explained below). So what happens is, depending on the source of the funds could be any of these:
* owner
* SPL token account
* base token account (unsettled)
* quote token account (unsettled) 

Trust me, if you get the source wrong, then placing an order will fail, and reply with a cryptic "custom program error" for which there is no further explanation. So we have to get this right!

So here's a table on which account to use as the payer:

<table>
<thead>
<tr class="header">
<th>market</th>
<th>order side</th>
<th>payer coin type</th>
<th>status</th>
<th>payer</th>
</tr>
</thead>
<tbody>
<tr>
  <td>XXX/SOL</td>
  <td>buy</td>
  <td>SOL</td>
  <td>settled</td>
  <td>owner</td>
</tr>
<tr>
  <td>SOL/XXX</td>
  <td>sell</td>
  <td>SOL</td>
  <td>settled</td>
  <td>owner</td>
</tr>
<tr>
  <td>XXX/SPL</td>
  <td>buy</td>
  <td>SPL token (incl. wSOL)</td>
  <td>settled</td>
  <td>SPL token address</td>
</tr>
<tr>
  <td>SPL/XXX</td>
  <td>sell</td>
  <td>SPL token (incl. wSOL)</td>
  <td>settled</td>
  <td>SPL token address</td>
</tr>
<tr>
  <td>XXX/SOL, XXX/SPL</td>
  <td>buy</td>
  <td>SPL token</td>
  <td>unsettled</td>
  <td>quote token address</td>
</tr>
<tr>
  <td>SPL/SOL, SPL/XXX</td>
  <td>sell</td>
  <td>SPL token</td>
  <td>unsettled</td>
  <td>base token address</td>
</tr>

</tbody>
</table>

#### SPL Token account
This is a settled token address in your wallet. It's the address of an SPL token that you wish to trade for something else, which can also be wrapped SOL SPL tokens. In order to get the address of this token, you can use solana CLI to find it:
```
# this will show all mint IDs for SPL tokens in your wallet
$ spl-token accounts  
...
# get the relevant account Address for the selected SPL token, wSOL shown below
$ spl-token account info So11111111111111111111111111111111111111112
...
```
It should be noted that Serum seems to accept both SOL or wrapped SOL ("wSOL" for simplicity), which is an SPL token that represents an equal amount in SOL, and has the mint ID `So11111111111111111111111111111111111111112`. To my knowledge, you can use either the SOL `owner` address (your main account), or the wSOL SPL token address (derived from your wallet) as the `payer`, but whatever the case, you will need to have at least as much SOL or wSOL in the account as you would like to transact.

IIRC, if the SPL token account for a pair you are quoting doesn't exist at the time you are placing an order, a small amount of SOL (0.002xx) will be deducted as rent to create the account.

#### Base Token Account
This is an unsettled token account used by Serum. By forex trading conventions, "base" represents the first token listed in a trading pair (e.g. RAY in the pair RAY/SOL). This is essentially the source token account that will provide money (the `payer`) for a 'sell', or receive tokens during a 'buy'. 

These accounts may be accessed in the following way:
```
const base = await market.findBaseTokenAccountsForOwner(connection, owner.publicKey, false);
console.log(base);
const baseTokenAccount = base[0].pubkey;
```

Note: this account is returned as a serialized BN, so it will not be readily human-readable. Neverless, if used as the `payer` for `markets.placeOrder(...)` or as the base account in the `market.settleFunds(...)` function, it should work. 

#### Quote Token Account
This is an unsettled token account used by Serum. By forex trading conventions, "quote" represents the second token listed in a trading pair (e.g. SOL in trading pair RAY/SOL). It is the token that denominates the amount that is "quoted" when trying to buy. For clarity, the quote token is what you trade away during a 'buy', or what you get in return when you 'sell'.

These accounts may be accessed in the following way:
```
const quote = await market.findQuoteTokenAccountsForOwner(connection, owner.publicKey, false);
console.log(quote)
const quoteTokenAccount = quote[0].pubkey;
```
Note: this account is returned as a serialized BN, so it will not be readily human-readable. Neverless, if used as the `payer` for `markets.placeOrder(...)` or as the quote account in the `market.settleFunds(...)` function, it should work. 

#### Open Order Accounts
These are Solana data accounts that contain information about an order to buy or sell. They exist as long as an order is open (e.g. an out-of-the-money limit order). The data can be accessed a few ways. All orders on the order book can be accessed in the following way:
```
let bids = await market.loadBids(connection);
let asks = await market.loadAsks(connection);

console.log("bids: \n")
for (let order of bids) {  // or asks
  console.log(
    order.orderId,
    order.price,
    order.size,
    order.side, // 'buy' or 'sell'
  );
}
```

We see that each order has a few defining features: `orderId` (type: BN), `price` (number) `size` (number), `side` (string).

A user can access the order data of their own *open order account(s)* by first finding them, and then iterating through them:

```
//Retrieving open orders by owner
let myOrders = await market.loadOrdersForOwner(connection, owner.publicKey); 

myOrders.map(order => console.log(order.side + ', ' + order.size + ', ' + order.price));
```

#### Misc Market Parameters 
To place an order on the order book, you additionally need to understand the `tickSize` and `minOrderSize`. The definitions follow:
* `tickSize` is the lowest significant figure of the price (denominated in the quote pair token) that you can buy or sell. 
* `minOrderSize` is the minimum amount of a token that you can buy or sell. 

These values vary by market. So you will need to know what they are in order to be able to make trades reliably:
```
// misc info market params
const TICK_SIZE = market.tickSize;
console.log("tick size: " + market.tickSize.toString());

const MIN_ORDER_SIZE = market.minOrderSize;
console.log("minimum order size: " + market.minOrderSize.toString());
```

### Using Serum
Up until this point, I've just been providing background, because this honestly was the hardest part for me to get my head around. The nuances explained thus far is the implicit context you need in order to operate the client. Much of these details are stored in devs' heads, and never seem to have been put to paper. So that's a major goal for these posts: *to demystify Serum*.

With our new understanding of various Serum trivia, we are now ready to attempt to place an order.

#### Placing an Order
Revisiting the SOL/USDC market, let's say we want to place a limit order for 1 SOL for 100 USDC. To do so, we first assemble the market data as previously described:
```
// Create a connection. Note: choose your RPC node carefully, don't be a dick!
let connection = new Connection('https://api.mainnet-beta.solana.com');

let marketAddress = new PublicKey('7xMDbYTCqQEcK2aM9LbetGtNFJpzKdfXzLL5juaLh4GJ'); // sol-usdc
let programAddress = new PublicKey('EUqojwWA2rd19FZrzeBncJsm38Jm1hEhE3zsmX3bRc2o'); // sol-usdc

const market = await Market.load(connection, marketAddress, {}, programAddress);
```
Next, we must assemble the accounts. Remember, selecting the `payer` is tricky, but starting new, let's assume we have some USDC in our wallet:
```
// owner was previously loaded using the serum.json keypair file
const payer = new PublicKey('<your USDC SPL token address>') // do not use a mint address here!
```
And place the order:
```
// assemble order parameters
// @ts-expect-error
const params = {
  owner: owner,
  payer: payer,
  side: 'buy',
  price: 100,
  size: 1, 
  orderType: 'limit',
  selfTradeBehavior: 'decrementTake', // don't worry about it
 };
 
// place the order
const sig = await market.placeOrder(
  connection,
  params,
);

// print out signature as confirmation
console.log(sig);
```
Success!

Beware that there is a `@ts-expect-error` or `@ts-ignore` that's required for this to work, if using Typescript. That's because the function is expecting a type `Account` while `owner` is type `Keypair`. Functionally, they appear the same, but without this, you'll get an error.


#### Viewing open orders:
Once you have an order on the book, you may want to retrieve its info. Do that like so (already described this above):
```
//Retrieving open orders by owner
let myOrders = await market.loadOrdersForOwner(connection, owner.publicKey); 

myOrders.map(order => console.log(order.side + ', ' + order.size + ', ' + order.price));
``` 

#### Cancelling an order
You may want to cancel an order, you can do so like this:
```
const cancel_sig = await market.cancelOrder(connection, owner, order);
console.log(cancel_sig);
```

#### Observing filled orders
Did your order fill? Check using the following:
```
// Retrieving fills
console.log("retrieving order fills:")
for (let fill of await market.loadFills(connection)) {
  console.log(fill.orderId, fill.price, fill.size, fill.side);
}
```

#### Settling Funds
Finally, you want to shut down and close out of this market. The following will allow you to settle your funds:
```
//Settle funds

// retrieve the base and quote token account addresses 
const base = await market.findBaseTokenAccountsForOwner(connection, owner.publicKey, false);
const baseTokenAccount = base[0].pubkey;
const quote = await market.findQuoteTokenAccountsForOwner(connection, owner.publicKey, false);
const quoteTokenAccount = quote[0].pubkey;

for (let openOrders of await market.findOpenOrdersAccountsForOwner(
  connection,
  owner.publicKey,
)) {
  if (openOrders.baseTokenFree > 0 || openOrders.quoteTokenFree > 0) {
    await market.settleFunds(
      connection,
      owner,
      openOrders,
      baseTokenAccount,
      quoteTokenAccount,
    );
  }
}
```
Your tokens should appear in your wallet (settled). Note that sometimes if you are receiving tokens from a market with SOL as part of the pair, it may be returned to you as wrapped SOL. So if you don't see the balance in your wallet, a) check that the transaction is confirmed, and b) use the `$ spl-token accounts` CLI command to see if it has been returned as wSOL.

### Summary
This part 1 of "How the F*** do you Use the Serum-TS Client" primarily discussed Serum's implementation, and the high level aspects of interacting with it. I make no claims that this is error-free, but I will do my best to correct errors as I discover them. Consider this a first draft on explaining the inner workings of the Serum client, and please correct me either in public or private, as this will improve my own understanding of the protocol. With this foundational stuff out of the way, we will explore more about what you can do with the client in subsequent parts. Thanks for reading.  