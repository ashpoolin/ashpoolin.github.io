---
layout: post
---

## Wallet Watching With Solana Web3.js

### Purpose
The speed of transactions, and insane amount of data pouring out of Solana can be daunting at times. You may feel that it is hard to get a grasp of what's going on, with each wallet having seemingly dozens of token sub-accounts, and many wallets executing several transactions per second. Despite this, there is significant alpha in being able to ingest and process data on specific accounts. You can use this to keep tabs on a whale you are trying to copy-trade, a project's vault or treasury, or automate the following of a hacker; really whatever. How do we filter through all this activity? Today, I will show you one way, although I'm sure there are other, better ways. Anyway, let's begin.

### Overview
The method will use the basic `solana/web3.js` library in javascript to call a function called `onAccountChange`, upon which it will parse the current account data, and dump to a postgres database that you can sift and analyze as you please. We will configure this system to monitor the SPL token accounts that bots tend to trade (e.g. wSOL, USDC, and others).

### Setup
1. You'll need Node and to create a new javascript project. 
```
npm init watcher;
```
2. You'll want solana [command line tools](https://docs.solana.com/cli/install-solana-cli-tools) installed, or you can use [solscan.io](https://solscan.io) to scout for accounts of interest.

3. You will also need to install `postgresql` and set up a new, _**dedicated**_ user that can `SELECT` and `INSERT` into a database table installed on `localhost`. If you share accounts this script will lock it up, or fail to write to the db if you're already using it. And I will not show you how to do any postgres config  because I hate doing it. Google is a better place to inquire about such things.

### Accounts
You'll need a wallet to follow. For an example, it may be interesting to see what a successful arb bot is doing. We'll make the assumption that they are using a swap feature, probably through [Jupiter Exchange](https://jup.ag/), so that the transactions fail or the tokens settle immediately in the wallet. Serum orders get a little funny because SPL token balances will not change until a trader or bot settles their accounts, so we're not going to do that. But let's head on over to [Jito's MEV feed](https://jito.retool.com/embedded/public/7e37389a-c991-4fb3-a3cd-b387859c7da1) to see if there are any subjects to observe. Okay, seems like this guy is pretty busy, and using a Jupiter-based bot: [HGSXs82RMXbTULPwJmrBdB33zrSvn3qvhk7Nrjwb8BVA](https://solscan.io/account/HGSXs82RMXbTULPwJmrBdB33zrSvn3qvhk7Nrjwb8BVA).

Next, we'll want to grab all of the related token accounts. Two ways: 
1. Click the "Token Accounts" tab in solscan, or 
2. `$ spl-token accounts --owner HGSXs82RMXbTULPwJmrBdB33zrSvn3qvhk7Nrjwb8BVA`

Holy crap that's a lot of accounts! Let's just pretend that most of their volume is done on USDC-SOL, which it probably is. We grab the associated token account (ATA) for each: 
- USDC ATA = `GWAYpfYMUKrfcN6fJKTo5Q86shXULyPP8fYe9VnqTepZ`
- wrapped SOL ATA = `9knZSjwFKFFgoD9ua8uYnbesN5yEg9GNnjQXWtd2EUoE`

Okay, all set.

### The Script 

#### onAccountChange
Let's start off basic, by trying to 1) get our script to capture an update on changes to the ATA's listed above, and 2) do some basic parsing of the data.

Start by just getting the blockchain to call you back:
```javascript
//watcher.js
import { PublicKey, Connection } from '@solana/web3.js';

const pubkeyUsdc = new PublicKey("GWAYpfYMUKrfcN6fJKTo5Q86shXULyPP8fYe9VnqTepZ");

const connection = new Connection("https://ssc-dao.genesysgo.net/", "confirmed");

connection.onAccountChange(
    pubkeyUsdc,
    (updatedAccountInfo, context) => {
        const buffer = updatedAccountInfo.data;
        console.log(buffer),
        "confirmed"
    }
);
```
To run, just `$ node watcher.js`.

When the user's USDC account is changed in some way, it will now return a buffer filled with the SPL token account's data. That's great, but not human-readable. So we'll have to parse it. The SPL token program encodes all of the info about the token account (what it is, its balance, etc) before stuffing it into the account on-chain. To retrieve it, we refer to the struct's [definition](https://docs.rs/spl-token/latest/spl_token/state/struct.Account.html):
```rust
pub struct Account {
    pub mint: Pubkey, // 32 byte
    pub owner: Pubkey, // 32 byte
    pub amount: u64, // 8 byte
    pub delegate: COption<Pubkey>, // don't care ...
    pub state: AccountState,
    pub is_native: COption<u64>,
    pub delegated_amount: u64,
    pub close_authority: COption<Pubkey>,
}
```

Since `onAccountChange` returns an encoded buffer, we just want to nab the various byte "slices" and then convert to something we can read.

Where to grab the bytes, exactly? We know from tribal knowledge that a `PublicKey` data type is 32 bytes, and unsigned 64 bit integer `u64` is 8 bytes. Let's start with the token mint address (the base58 address of the "type" of coin that it is, such as USDC):
```javascript
const mint = (new PublicKey(buffer.slice(0,32))).toBase58();
```
Let's unpack a bit. First, we take a 32 byte slice of the buffer with `buffer.slice(0,32)`. Starts at the `0` position (inclusive), and ends at `32` (not inclusive). Next, we create a `new PublicKey(...)` from the slice, and finally, cast it all to a base58 string with the `(...).toBase58()` method that comes to us conveniently from the `PublicKey` type in `solana/web3.js`. We'll do the same for the owner `PublicKey`.

A word on the amount: it is an unsigned 64-bit integer, so what does that mean for a javascript type? Well, the correct type is a 64-bit bigint, that is packed little-endian. Wtf is that? Little-endian means the most-significant byte is on the right (last), while big-endian stores the MSByte on the left (first). Whatever, here's how you unpack it:
```javascript
const amount = (buffer.slice(64,72)).readBigUInt64LE();
```
_Note that this amount is in base units, so if you want the actual amount, usually you need to divide by 1,000,000 for most SPL tokens. For wSOL it may be 1,000,000,000 since the base unit may also be Lamports. I don't recall._

With that out of the way, we cue up our indices and unpack the data into our variable, then print to console:

```javascript
connection.onAccountChange(
    pubkeyUsdc,
    (updatedAccountInfo, context) => {
      const buffer = updatedAccountInfo.data;
      const mint = (new PublicKey(buffer.slice(0,32))).toBase58();
      const owner = (new PublicKey(buffer.slice(32,64))).toBase58();
      const amount = (buffer.slice(64,72)).readBigUInt64LE();
      console.log(`${mint},${owner},${pubkeyUsdc},${amount}`),
    "confirmed"
    }
  )
```

#### Multiple Accounts
That's all well and good, but you probably don't want to just read a single SPL token account in isolation. So let's add some more. Minor tweaking to our code above using an array of base58 strings for addresses, and the `array.map(...)` function, and we're good to go:
```javascript

const addressArray = [
"GWAYpfYMUKrfcN6fJKTo5Q86shXULyPP8fYe9VnqTepZ", // USDC
"9knZSjwFKFFgoD9ua8uYnbesN5yEg9GNnjQXWtd2EUoE", // wSOL
// ... add as many SPL token accounts as you want
];

console.log(`mint,owner,associated_token_account,amount`);

addressArray.map( address => 
  connection.onAccountChange(
    new PublicKey(address),
    (updatedAccountInfo, context) => {
      const buffer = updatedAccountInfo.data;
      const mint = (new PublicKey(buffer.slice(0,32))).toBase58();
      const owner = (new PublicKey(buffer.slice(32,64))).toBase58();
      const amount = (buffer.slice(64,72)).readBigUInt64LE();
      console.log(`${mint},${owner},${address},${amount}`),
    "confirmed"
    }
  )
);
```
Alright! Firing that up, you are now able to observe more than one account at a time!


#### Postgres
To get ready to pass the data to postgres, we'll want to add a compatible `date` to our code:
```javascript
// date stuff
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

// use it like this:
const rightNow = formatDate(new Date());

```
This produces dates in the format of `YYYY-MM-DD hh:mm:ss`, which corresponds with postgres data type `timestamp`.

Next, we'll use the npm package `pg`, so add it with `npm install pg`. Gonna speed run this part because it's dinner time... add the following, and modify our main function below:
```javascript

// add this
import pkg from 'pg'; //funny import: it's commonjs, but I'm using type = module
const { Client } = pkg;

// add this
// postgres client
const client = new Client({
  user: 'me',
  host: 'localhost',
  database: 'spellz',
  password: 'password',
  port: 5432,
});


client.connect(); // add this
addressArray.map( address => 
  connection.onAccountChange(
    new PublicKey(address),
    async (updatedAccountInfo, context) => {
      const buffer = updatedAccountInfo.data;
      const mint = (new PublicKey(buffer.slice(0,32))).toBase58();
      const owner = (new PublicKey(buffer.slice(32,64))).toBase58();
      const amount = (buffer.slice(64,72)).readBigUInt64LE();
      const insertText = 'INSERT INTO event_log(date, mint, owner, associated_token_account, amount) VALUES ($1, $2, $3, $4, $5)'; // add this
      await client.query(insertText, [date, mint, owner, insider, amount]); // add this
      console.log(`${formatDate(new Date())},${mint},${owner},${address},${amount}`),
    "confirmed"
    }
  )
);
```
What's happening? We create a new `Client` from the `pg` package, connected to it, and then passed our variables to a postgres table called `event_log` using an `INSERT` query. We added `await` and made this block `async`, to avoid concurrency issues w/ writing to the database. Evidently, we cannot write to a table that does not exist, but it's pretty easy to set one up. You can copy mine below:
```sql
create table event_log(
    date timestamp,
    mint varchar(255),
    mint varchar(255),
    mint varchar(255),
    amount bigint
);
```
Now, running your script will both print the info to the console as it occurs, but also place it neatly in a database, which is much more helpful when it comes time to do whatever analysis.

#### Final Script
```javascript
// watcher.js
import { PublicKey, Connection } from '@solana/web3.js';
import pkg from 'pg';
const { Client } = pkg;

// date stuff
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

// postgres client
const client = new Client({
  user: 'me',
  host: 'localhost',
  database: 'spellz',
  password: 'password',
  port: 5432,
});
client.connect();

// Solana stuff
// define your list of SPL tokens accounts of interest
const addressArray = [
"GWAYpfYMUKrfcN6fJKTo5Q86shXULyPP8fYe9VnqTepZ", // USDC
"9knZSjwFKFFgoD9ua8uYnbesN5yEg9GNnjQXWtd2EUoE", // wSOL
// ... add as many SPL token accounts as you want
];

// create connection through RPC node
const connection = new Connection("https://ssc-dao.genesysgo.net/", "confirmed");

// print a header
console.log(`date,mint,owner,associated_token_account,amount`);

// main callback function
addressArray.map( address => 
  connection.onAccountChange(
    new PublicKey(address),
    async (updatedAccountInfo, context) => {
      const buffer = updatedAccountInfo.data;
      const mint = (new PublicKey(buffer.slice(0,32))).toBase58();
      const owner = (new PublicKey(buffer.slice(32,64))).toBase58();
      const amount = (buffer.slice(64,72)).readBigUInt64LE();
      const insertText = 'INSERT INTO event_log(date, mint, owner, associated_token_account, amount) VALUES ($1, $2, $3, $4, $5)'; 
      await client.query(insertText, [date, mint, owner, insider, amount]); 
      console.log(`${formatDate(new Date())},${mint},${owner},${address},${amount}`),
    "confirmed"
    }
  )
);

```
There you have it! A simple script that can keep eyes on one to hundreds of SPL accounts and dump the token account balance into a database for your research. I won't explain it here, but using [postgres LAG function](https://www.postgresqltutorial.com/postgresql-window-function/postgresql-lag-function/) you can discern transfer size and direction (to/from) by subtracting a previous `amount` (the token balance) from the current one.   

### Misc Logging and Shell Stuff
I like to run my program in the background, and log both to postgres, and just a .csv file. To do this on the command line, you can use `nohup` or just send it to the background, as in:
```bash
$ nohup node watcher.js >> logfile.csv 
# OR
$ node watcher.js >> logfile.csv &
```
I also have a script to manage my script, and checks to see if the process has failed, and restarts it if it's dead. This is because I am lazy, there are probably better ways. Looks something like this:

```bash
#!/bin/bash
# launcher.sh
script_path="./watcher.js"
keyword="watcher"

while :
do
    status=`ps -ef | grep node | grep ${keyword} | wc -l`;
    echo "bot status is ${status}";

    if [ $status -eq 0 ]; then
        echo "starting... nohup node ${script_path}"
	node $script_path  >> logfile.csv &
    else
        echo "process is healthy."
    fi

    sleep 10;

done
```
With all of that out of the way, you should have a pretty durable setup that allows you to be scanning the Solana blockchain, mostly unsupervised.

### Conclusion
We covered a lot, honestly more than I thought we would. We learned how to get notified when any account of interest changed, exerted a little effort parsing the data encoded in SPL token accounts, and finally, figured out how to dump it all into a postgres database so that you can analyze it. 

Thanks for reading, and if you found this helpful, please consider sharing it.
_-Ash_




