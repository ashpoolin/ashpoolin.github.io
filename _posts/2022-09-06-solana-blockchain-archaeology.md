---
layout: post
---

## Deep Dive on Blockchain Archaeology: Accounting in Solana 

### Disclaimer (!)
**_I am not any sort of professional, this article shall not be taken as legal, tax, or accounting advice. It is merely a demonstration of data querying, sorting, and arithmetic. Do not take any of the demonstrated methods as legal or tax advice—seek the counsel of a professional! I am not liable for any mistakes, financial or personal liability you incur as a result of reading this article. This is merely to raise questions and issues that I encountered, and how I went about dealing with them. Every person’s experience is different, so again, I plead with you to seek the advice of technical support and tax professionals when preparing your own documents. I am merely a CT anon, and this is my experience..._**

### Introduction
I didn’t intend to become an expert in this. In all of the excitement of a new blockchain, Solana, I was a kid in a candy shop. I wanted to try all of the new protocols, and see how they worked. I created money-losing bots just for the satisfaction of watching my code transact autonomously. And I created traffic. Holy hell did I do a lot of transactions—more than 10,000 in a year. Now 2022, I found myself between a rock and a hard place: in need of someone whose skills were somewhere along the lines of blockchain archaeologist, and forensic accountant, in order to finish my taxes. In short, I’d need to figure out wtf I did.

### Self Sovereignty
Here’s the ugliest truth about defi: with self-sovereignty of your funds comes the requirement for self-sufficiency in record keeping, and accounting, for doing your taxes. Let’s presume you are a law abiding citizen who doesn’t want to go to jail… We do our taxes, and we do them as correctly as humanly possible. But transacting with blockchains using machines, we also need a machine’s help. We will walk through some tricks to help you collect and process all of your transaction information on Solana. Last year I made my bed, and there was no turning back, so this is a pile of broken glass I had to walk through. But I want to caution anyone who is not up to the job: if you can’t run down all of your trades, or keep meticulous (nearly 100% accurate) records as you go, then your best bet is to keep your coins on a supported exchange, and link the exchange account to a coin tracking / tax product to generate your reports at the end of the year. You’ll also have technical support if you run into trouble. Again, self-sovereign money is for the truly ambitious, stupidly self-reliant, and not for the faint of heart!

### Skills
To be able to sleuth on-chain, the proficient defi user must be some blend of these:
* **Trader**. Can you identify the types of trades you did on-chain? What were the instruments used, what was the position like at the time (e.g. long/short, delta-neutral, et cetera)?
* **Accountant**. Not really, but you should absolutely have taken a college-level accounting course and understand the basics of double-entry bookkeeping. Seriously. If you need a tip on where to look, try this: [Coursera - Wharton Accounting](https://www.coursera.org/learn/wharton-accounting?specialization=finance-accounting) 
* **Data Science/Developer**. Not completely, but having some scripting know-how, using JSON API calls, Javascript or Python, and having some command line (BASH/Linux) chops will work wonders. Familiar with comma-delimited files (these are my lifeblood). General “data munging” capability.
* **Excel**. You better know your way around a spreadsheets application such as Excel, or LibreOffice Calc. That includes using formulas with simple if/and/or statements and vlookup, for your own sanity. 

### Why do I need to know how to do this?
I’m optimistic that cryptocurrency tax software will get better, but currently these are incomplete products, imo. Coin tracking software often does not support every chain, protocol, trade type, and believe it or not, it often makes mistakes! You need to be self-reliant if you are doing anything on-chain. You can’t go to a public blockchain and ask customer support to help you file your taxes. So get out your machete, we’re going into the jungle!

### Limitations of Coin Tracking Software
To go a bit further, here’s an example of some of the trades that will probably not be well supported by off-the-shelf coin tax software:

* **_Transactions involving rent_** (on Solana) and the creation of an account. This muddies the overall transaction, so a transfer may look like a trade (where you spent 0.00204 SOL, and received an SPL token = wrong!), or just makes it indecipherable to your software (gets flagged as “UNKNOWN”). Either way, you’re SOL (shit-outta-luck) and will have to correct it manually.
* **_Liquidity Providing (LPing):_** as is often the case with “pool two” AMMs, or even fancier pools, you often provide token to be retrieved at a later time, plus some yield, less impermanent (now permanent) loss. Importantly, when closing a position, balances of either of the tokens has changed, and you need to record that. The temporal distance from depositing and withdrawing an LP position makes it difficult to calculate the net effect of it on your profit & loss statement (P&L). That, and the opening/closing of these positions will probably be flagged as a “transfer” rather than any sort of “trade,” thereby not counting towards your P&L without manual intervention.
* **_Any trade involving an escrow-like function._** Examples are trades on limit order books (Serum), or NFT marketplaces (Magic Eden). Here, the issue is (historically) that the coins are physically removed from your wallet before a trade/match can take place. Again, the issue is that your coins leave your wallet as a transfer, and reappear back in your wallet as something else, when they are settled. Again, identifying an executed limit order is more difficult than an atomic swap via an AMM, since there’s a time delay between placing an order and execution. All told, I doubt your coin tax software will correctly identify these types of trades, so you’ll have to hunt them down yourself, then report them manually.
* **_Short sales / margin trades._** What happens when you borrow coins, then sell them to buy them back cheaper? Well, most commercial software will see that you never bought the coins, but now they magically appeared in your wallet, and then you sold them. So, they will assign the proceeds as having a $0 cost basis, and record the sale as 100% profit. That’s not good for your end-of-year tax liability, and plus it is incorrect. I’ve yet to see a “short sale” toggle mode in any software, which would be a desirable feature. So If you are not careful with this, and have a borrow that you swap for something similar (e.g. borrow sol, stake as msol), you now have a short sale on your hands, which will cause your P&L to explode, for a seemingly trivial amount of yield. That is, if you don’t label and handle the transaction correctly.
 
### Solana Account Model

Despite some difficulty in retrieving historical transaction data, the Solana account model lends itself nicely to an important accounting primitive: [double-entry bookkeeping](https://en.wikipedia.org/wiki/Double-entry_bookkeeping)! Never thought I’d get excited about this, but since each SPL token gets its own account, it’s a logical way to partition transaction data for the purpose of getting your financial history in order.

![dual-entry table](https://raw.githubusercontent.com/ashpoolin/solana-archaeology/main/images/dual-ledger.png "dual-entry trade table")

Using an `account model`-based approach, we sidestep the absolute ADD that comes from trying to process your entire wallet in one go. It’s too difficult, unless you’re some wonky polymath, in which case you don’t need my help. But for the normies out there, we need to focus on one account, one coin at a time. This allows us to see what’s going on with your coins in a given account, and make sure that you get the correct balance at the end.

### The Procedure

You will need:
* This **repository**: [solana-archaeology](https://github.com/ashpoolin/solana-archaeology)
* **Linux / bash command line environment** – should work with Windows via WSL2 or VirtualBox, Mac OS, or any vanilla Linux distro
* **csvkit** – command line utility for dumping JSON API result to comma-delimited (.csv) files
* **Node** – installed
* **Typescript** – Basic ability to compile (transpile) Typescript files, and work with TS/JS scripts and Node

To get an understanding for how we can **attack our transaction history on a per-token-account (ATA) basis,** we lay out a model that allows us to be as thorough and methodical as possible. 

1. Find all of your known wallets, and collect their addresses (base 58 “public key”)
2. Gather a list of all the SPL tokens you believe you traded, and their mint IDs (base 58 “public key”). More scientific means for gathering this list of mint IDs is possible, but usually most people just trade in a handful of coins, which comprises the bulk of the volume, and profit and loss. 
3. Modify the `index.ts` script to include a list of desired wallet addresses and coin mint IDs to search. `npx tsc` to compile, then `node index.js > all_ata_search_space.csv` to generate all associated token account addresses as a .csv.
4. Next, I like to **split up all of the SPL account-based transactions by coin.** This allows you to get a comprehensive picture of what’s going on with the whole balance of the coin, across many wallets, et cetera. 
  * Create a bunch of new directories for each: 
  * `mkdir ETH BTC SOL MSOL WSOL USDC USDT ...`
  * copy the `looper.sh` and `getSignaturesForAddress.sh` scripts (found in the [repo](https://github.com/ashpoolin/solana-archaeology) under `./scripts/`) into each of the coin directories.
  * Next, copy-paste each of the associated token account addresses into a plain text file for each coin. Ex:
    ```bash
      nano ETH/ETH_ATA.txt # paste only the base58 ATA for each related wallet account here, no header
    ```
  * Run the `looper.sh` script to generate a .csv file of each of the ATA in the list. It will use an RPC to make JSON API calls and convert each of the JSON outputs to their own <associated_token_account>.csv file. Each of these files contains various information about each transaction, including whether it was successful (if not, ignore), the blocktime (can be converted to a human-legible timestamp), and the transaction signature.
  * Go through each coin and list of ATA generating all of the raw transaction files. You will have a lot of them. But congratulations, you’ve just mined the Solana blockchain, reaping your “ore” / raw material that you will use to search for all of your trades and notable activity!
  * Let’s start to refine, a bit. We want to remove the files that don’t actually have any activity in them, to save time. You could script this, but I just found an easy method is to just check the line counts of each file, move the ones you want into an INSPECT folder, and delete the rest. Although, you may want to keep them for posterity, you could sort them into another folder called NULL:
  ```bash
   wc -l *csv | sort -bh # tx with more lines appear at the bottom
   mv xxxx.csv INSPECT/  # …
   mv yyyy.csv NULL/ # line count = 1 means no data → safe to ignore
   ```
5. **Inspect / Analyze Trades.** Now, all of your meaningful token accounts that should be inspected exist in directories like this: `<coin name>/INSPECT/<associated_token_account>.csv`. You must now go through the painful process of evaluating each of the signatures. You have a couple of choices:
  * Copy-paste each tx id (signature) into a useful explorer like [solscan.io](https://solscan.io) 
  * Use a tool like [Helius](https://helius.xyz/) Transaction History to parse you transactions, and do the evaluation for you [DOCS](https://docs.helius.xyz/api-reference/enhanced-transactions-api/transaction-history). Here, you just search the ATA for all txs, and it will do some analysis for you. An example script that dumps JSON follows:
    ```javascript
      // helius-tx-history.js
      const api_key = "<insert your API key>"
      const address = "<insert associated token address>"
      const axios = require('axios')
      // get transaction history
      const url = `https://api.helius.xyz/v0/addresses/${address}/transactions?api-key=${api_key}`
      const parseTransactions = async () => {
        const { data } = await axios.get(url)
        const myJson = JSON.stringify(data);
        console.log('{ "transactions": ',myJson, '}')
      }
      parseTransactions()
    ```
    While calling the script, you can convert to a .csv file using csvkit:
      ```bash
        node helius-tx-history.js | in2csv --format json --key transactions > transactions.csv 
      ```
  * Use [stake.tax](https://stake.tax) to provide some analysis for you. I have found this tool is very convenient and helpful, but I fear that it may not be actively maintained or improved in the future. If you like it, please consider making a donation to support it.
6. Okay, now you’re in the thick of it. You’re working through each token account, for each coin. When you finish a “coin” (completely analyzed a set of token accounts), I find it’s most helpful to separate anything that is a taxable event from things that are personal transfers. So I would separate or remove things like transfers (deposits / withdrawals) and focus on trades (profit and loss), LP gain/losses, staking yields, or generally income. Note that you may want to keep the deposit / withdrawal history for your coin tracking software, since it can help tell you where coins are in wallets or exchanges. For me, it always seemed to mess things up, though.
7. **Combine.** Combine your profit and loss generating activity into a single .csv, .xlsx, or .ods file. Sort it chronologically (ascending). Note that if you just have the raw `blocktime` and not a timestamp, this Unix timestamp can be converted to a `datetime` (typically in UTC) that is more agreeable to human forms: `datetime = date(1970, 1, 1) + blocktime / 24 / 3600`. Note this is a dirty conversion, don’t go putting it into smart contracts (use a library instead!). But should be okay for spreadsheets or personal use. _Another word on timing: it is important that you have as granular of data as you can. For instance, if you paid or were paid interest on something hourly, maybe you don’t need your transaction history to be collected hourly, but definitely aggregate it daily and create a tx for it. Lumping all of your interest together for a year is a) probably not tax-compliant, and b) while the coins may balance at the end, your real, working balance will be off, which can lead to the problem below. You want your running balance to be as accurate as fucking possible. Full stop._
8. With all of your P&L activity now in order, in a spreadsheet, you can now do things like **calculate your `running balance`, and `final balance`.** Two important things to note:
    * if your `running balance` of the coin ever goes negative (<0) any coin tracking software will probably count this as a zero cost-basis trade (no matching “buy”), and this will be marked as 100% profit. You are either missing the matching buy, or this was made on margin (borrowed coins = functionally, a “short”). If the coins were borrowed, this will be a pain in the ass, and you’ll probably need to label these tx’s as a margin profit/loss to prevent the issue above. A helpful tool is to create a column in your spreadsheet to flag when the balance has run negative, and/or create an x-y scatterplot (line chart) to show when the a coin’s running balance has dipped negative. Doing this helps you identify trouble areas and should allow you to evaluate what's going on. Example below:

    ![alt text](https://raw.githubusercontent.com/ashpoolin/solana-archaeology/main/images/running-balance.png "X-Y scatterplot showing running balance vs. time")
           
    

    * Your `final balance` should reflect the current state (balance) of all of your wallets. 
      ```
        final balance = sum(buys, income) – sum (sells, losses)
      ```
      E.g. if you bought and sold everything, then it follows that:
      ```
        0 = sum(buys, income) – sum (sells, losses) 
        => sum(buys, income) = sum(sells, losses)
      ```
       If it doesn’t balance, you’re missing data, and may need to look for additional wallet addresses, or ATAs. It may also not balance because you may have coins on exchanges, or other blockchains, of course. We’ll address that next.
       
       _Note about Serum transactions:_ the combination of placing an order in one account, and settling in another, means that you may not be able to find the `place order` or `settle funds` transaction to correctly assemble a complete trade. This is horribly painful. For this, rather than scrolling through endless transactions in your main wallet on solscan, or obtaining a full wallet file from stake.tax (which can take minutes), it’s probably easier to just use the my bash script to get all of your signatures + blocktime:
       
       `./getSignaturesForAddress.sh <your main wallet address>`
       
       Now you can load that CSV file into Excel, and search (ctrl + F) for the transaction you’re missing a match for. It should be just a few transactions away, either up or down, depending on whether you are looking for an `place order` or `settle trade` transaction.
       
9. **Upload.** With all of your tx history balanced at the coin level, you’ll want to upload it to your coin tracking software of choice. There are many good options, but it sounds like these are the most popular: cointracker, cointracking, taxbit, koinly, tokentax, and zenledger. I don’t have experience with more than a couple of these, so I can’t say which is the best. So uploading your solana transaction history (at the coin level) combines it with all of your exchange transactions, and should give the correct, global state for your crypto holdings.
10. **Your Solana balance.** SOL is not an SPL token so the system above won’t work. That said, any on-chain trade including an SPL token will be picked up by the coin-by-coin method I've explained, but will obviously miss things like staking rewards. So either way, you probably want to search for SOL txs explicitly. Probably just use [stake.tax](https://stake.tax) or your wallet import function from your coin tracking software of choice on the main wallet address, then delete the txs that touch any SPL tokens. These will be added back by your careful analysis, and I almost guarantee you they will be more accurate than the crude import functions that have the problems (enumerated earlier in the article) with trades that involve limit orders or escrow, for example.  

### Conclusion
This concludes a primer in Solana blockchain archaeology, for the purposes of personal accounting. Woof. But there you have it! A system for carefully, and methodically evaluating you Solana on-chain transactions. This is isn’t the easy route, but then again, the alternative is more grim. I foundered for six months looking at my whole wallet transaction history, completely overwhelmed, and scarcely making any progress on my P&L. I had zero real success until I finally decided to adopt the `account model` or "coin-by-coin" method for processing my transactions. I have provided you with ideas and simple scripts to comprehensively collect and organize you entire SPL (associated token account, or ATA) history. Finally, we took a look at some of the technical considerations when analyzing transactions, although without presenting examples. We formalized our method by providing a formula that says that the current balance must be equal to the sum of all buys/profits and all sales/losses. To get all of your coins to balance across all token accounts is an exercise for the reader, or the heroic accountant.

### Future Work
Is there a way to package this as a tool or product that a more general audience could use? Yeah, probably. If there’s enough interest in this maybe I’ll take a crack at automating pieces of it for you. That, or you could develop it yourself, and I’d gladly pay you for the privilege. Until then, this is what I’ll likely be doing.

Thanks for reading. 