# kraken-grid
A bot that extends grid trading once you use it to create a grid using orders with conditional closes.

This was developed with NodeJS running in the BASH shell provided by Windows 11.  I believe it's using "Windows Subsystem for Linux" and that there are some oddities because of this.  I don't see them as odd because I'm not familiar enough with Linux yet.

## Upgrading
This version stores your API key and secret using your password.  This information is stored without encryption in previous versions, but this one will read that file, use the data as default values when it prompts you for the API Key and secret, and replace it with the encrypted data.  I recommend finding your keys.txt file in your home directory and making a copy of it just in case.  If you forget your password, you must re-enter the API keys to reset it.

## Installation
1. Get your API key and secret from Kraken. Otherwise, you will have to go through this process again if you want to run kraken on a machine that doesn't have these keys yet, or if you forget your password.
   - 1.1 Click the box in the upper right corner on kraken.com after you log in that has your name in it.
   - 1.2 Click the "Security" item in the dropdown box.
   - 1.3 Click "API" in the list of options under Security.
   - 1.4 Choose the "Add Key" link.
   - 1.5 We recommend that you give your key a better description.
   - 1.6 We recommend that you record this set of codes (a key and a private key, called "Secret" in kraken-grid).
2. Install [NodeJS](https://nodejs.org/)
3. Run `npm g install kraken-grid` from a command line ("Command Prompt", "Terminal Window", or "Shell").
4. If Node installed kraken-grid successfully, you can now run `kraken-grid` from the command line on your computer.  It will ask you for a password.  If it has no previous record of a password being entered into kraken-grid, then it will also ask you for the API key and secret ("private key" is what Kraken calls the secret).  Your password is used to encrypt information that kraken-grid stores on your computer, like the API keys.

This software will save a file `keys.txt` to your home folder. 

## Usage

In this section, prospective documentaiton is marked with an asterisk* to indicate features that are being added.  If all is correctly updated, such asterisks will only ever appear in this readme file on branches, and those branches will be where the features are being developed.  This gives devs a handy way to find the specs for a new feature.

### Changing your password*
When you start the bot, it asks for a password.  If you enter the wrong password (or there is no password yet), it will assume that you want to set a new password and ask you to enter your API keys from Kraken again.  You can simply enter 'x' to start over if you want to keep your old password and think you mistyped it.

### Mistyped passwords*
If you think you mistyped your password, just enter x as described above.

### Command Line Interface
At the prompt that kraken-grid presents (>), you can enter one of these commands:

### Trading
#### buy
`buy [XMR|XBT|ETH|DASH|EOS|LTC|BCH] price amount closePrice`
If closePrice is not a number but evaluates to true, the code will create this buy with a conditional close at the last price it saw.  If it is 1, that might be because you want it to evaluate to true and close at the current price, or because you want to close at 1.  The bot plays it safe and closes at the current price.  To change that, you can use the `risky` command (see below).  If you don't want the code to place a trade with a conditional close, leave closePrice off or pass `false` for it.

#### sell
The semantics are the same as for `buy`

#### kill
`kill X`
X can be an Order ID from Kraken (recognized by the presence of dashes), a userref (which often identifies more than one order, and, importantly, _both_ the initial buy or sell, _and_ the series of sells and buys resulting from partial executions), or a `Counter` as shown from `list`.  This cancels the order or orders.  `list` will still show such orders, prefixed with `Killed`, until `report` runs again to update the internal record of open orders.

#### delev
`delev C`
C _must be_ a `Counter` as shown by executing `list`.  If the identified order uses leverage, this command will first create an order without any leverage to replace it, and then kill the one identified. The order that was killed will still be in the list, prefixed with `Killed:` ***NOTE: The new order often (or always?) appears at the top of `list` after this, so the `Counter`s identifying other orders may change.

#### addlev
`addlev C`
The semantics are the same as for `delev`.

### Information Gathering
#### assets*
`assets [F]`
This provides you with a list of assets in each account (see `asset` under bot management), a list of all all assets combined across accounts with your chosen allocation for each one, and if F is anything, all listed assets will contain F in their ticker.

#### list [SEARCH]
This simply prints out a list of all the open orders the code last retrieved (it does NOT retrieve them again, so...) It may have orders in it that have already been executed or which you canceled.  Each order is presented as:
`Counter K trade amount pair @ limit price [with A:B leverage] userref [close position @ limit price]`
...where:
* `Counter` gives you a handle to use for other commands like delev and kill.
* `K` is `Killed` if you used the kill command to cancel an order and the bot hasn't yet updated the list. For existing orders, `K` is missing.
* `Trade` is either `buy` or `sell`.
* `Amount` is the number of coins.
* `Pair` is symbol (see `buy`) with the 'USD' suffix.
* `Price` is the price for this trade.
* The corresponding bracketed items will be missing for an order with no leverage or without a conditional close.
* `userref` is a user-reference number created when you use the `buy` or `sell` command.  It starts with 1 for buys and 0 for sells (but since userrefs are integers, the 0 gets removed), followed by three digits that identify the cryptocurrency pair, and then the price without the decimal point and with leading zeroes.  Note that this causes collisions in very rare cases like a price of $35.01 and another price for the same crypto of $350.10.  I expect this to be too rare to fix at this time.

If you enter anything for [SEARCH], the list will only display lines that contain what you entered, except in one case, `C`.  If it's just the `C`, it will retrieve the last 50 orders that are no longer open (Filled, Cancelled, or Expired), but only list those that actually executed (Filled).  If you add the userref after `C`, then it will fetch only orders with that userref, which means the buys and sells between one set of prices. Use `set` for a list of the userrefs for the grid points.  Such orders also include the time at which the order filled completely.

#### margin
Whether or not you use this command, the bot will try to use leverage when there isn't enough USD or crypto.  Whether or not it succeeds, it will still be able to report how much you are long or short for each supported crypto.  Reporting that is all this command does.

#### report
This is the default command, meaning that if you don't enter a command, but you hit enter, this command will execute.  It does several things:
1. Retrieves balances for the cryptos listed under `buy` and reports the values in a table:
```
ZUSD    AMT       undefined
XXBT    AMT       PRICE
XLTC ...
...
```
2. Retrieves the list of open orders, which is immediately processed to:
   1.  replace conditional closes resulting from partial executions with a single conditional close which, itself, has a conditional close to continue buying and selling between the two prices, but only up to the amount originally specified, and _only_ for orders with a User Reference Number (such as all orders placed through this program).
   2.  fill out the internal record of buy/sell prices using the open orders and their conditional closes (see `set` and `reset`).
   3.  extend the grid if there are only buys or only sells remaining for the crypto identified in each order.
   4.  identify any orders that are gone or new using Kraken's Order ID and for new orders, it also describes them.

#### show
`show` displays some bot internals which will only be useful to you if you understand the code enough to figure out what passing a parameter to it does.

#### verbose
There is a little bit of logic in the code to spit out a lot more information when verbose is on.  It's off by default and this command just toggles it.

#### web
`web S` is used to turn on or off the web interface.  This interface is rudimentary at this point, but it's more convenient for me than the command line.  They are best used together.

### Bot Management
#### adjust
`adjust T A P`
This feature currently only affects your desired allocation by reading the price history of the selected asset (T, for ticker) and moving up to A percent from USD in your savings to the allocation for T based on where the current price is within the most current range that is P% wide.  As you go back in time through the price history, the ratio between the highest and lowest price (high over low) slowly increases, so that at some point, the high will be more than 100+P percent of the low.  The current price (let's say 100) is within that range, perhaps near the bottom ("relatively low"), let's say 25% of the way from the low to the high.  In this case, we'd want more of that asset, and this is how `adjust` affects the allocation.  At 25% of the way through the range, we want to move 75% of A from the original allocation of cash to the allocation of T.  At 10%, we'd want 90% in T.  For example, let's say the current price is 100 and the recent low is 95 and an earlier high was 200.  The range will be 95 - (95*1.2 =) 114, meaning the price is currently about 25% above the 95.  If A is 15, 75% of that is 11.25, so when the price is so low, the bot sets the allocation of T to 11.25% more than its original value.  As the price rises, this allocation falls back down to the original value.

#### alloc* (Not implemented)
`alloc S P [A]` allows you to specify how you want your savings allocated.

* `S` is a symbol and if it is found in the list of asset tickers from the exchange (only Kraken at the time of writing), the bot will record `P` as the percentage allocation target for your savings.
* `P` must be a number (any more than two decimal places will be ignored). It is interpreted as the percentage of your entire savings you would like this asset to be.  If such an allocation is already recorded (and `A` is not present and "false"), the bot will ask you to confirm that you want to update it.  Your default currency allocation will be adjusted so that the full allocation adds up to 100%.
* `A` is for Ask. (optional) It must be the string 'false' if it is present because it prevents the bot from asking before overwriting existing data.

#### allocate
`allocate` starts a process through which you can enter how you want your savings allocated (see [Balancing the Present](https://litmocracy.blogspot.com/2019/06/balancing-present.html), which is the motivation for this software).  It will ask if you want to erase your current allocation even if you don't have one.  When you have no allocation, it starts with the actual current allocation as a default and allows you to adjust it.  When you are satisfied with your allocation settings, enter N instead of a ticker to get back to the bot.

#### allocation
`allocation [F]` will display your current actual allocation across all the assets you have entered (see `asset` below) as well as the assets you have on the exchange, including any positions you hold.  F can be ? to review what the command does, or "fresh" to have the code get new prices from the exchange.

#### asset
`asset S U [L] [A]` allows you to describe your savings to the bot so that it can automatically trade without you entering any trades yourself.

* `S` is a symbol and if it is found in the list of asset tickers from the exchange (only Kraken at the time of writing), the bot will take into account that you hold U units of this asset, but not on the exchange.
* `U` is the number of units that you'd like the bot to record.  This information is saved to disk (as a JSON object) after being encrypted with your password.  If you call asset on the same symbol twice, the bot will ask you to confirm that you want to overwrite the old value.
* `L` is for Label. (optional) This string will be used as an account label.  If there is no such account, the bot will confirm that you mean to create one.  If you hold cryptos in two different wallets, W1 and W2, you can tell the bot how much is in each one and it will keep separate records for them.  THe default account will be used if `L is missing, and its label is "default".
* `A` is for Ask. (optional) It must be the string 'false' if it is present because it prevents the bot from asking before overwriting existing data.

#### balance
`balance TOL [TKR]`
TOL is the tolerable difference between your desired allocation and your current allocation.  If you don't specify TKR, the bot will identify the most out-of balance asset and propose a trade to balance it.  If TKR is present, it will create a trade to balance that ticker along with a limit buy and a limit sell at prices TOL percent (this is a number between 0.00 and 100.00) above and below the current price.  These trades will then be used by the bot (in `auto` mode) to keep your savings in balance.

#### limits
`limits AtLeast AtMost`
The bot starts out with no limits, using 0 as AtLeast and -1 as AtMost.  You can use this command to prevent it from trading unless the trade amount in USD (other fiat currencies will be supported soon) is inclusively between these two limits.

#### less
`less C A F`
C _must be_ a `Counter` as shown by executing `list`. This command reduces the amount of crypto in the limit order identified by C in list by amount, and if ALL (optional) is "all", update any other orders for the same crypto for which the current amount to be traded matches (to three decimal places, after rounding) the pre-adjusted amount of the identified trade.
Example: You have a limit sell order at 45000 for 0.015 BTC and another above that at 45900 for 0.015 BTC, each with its own conditional close. When you issue list they show up as numbers 3 and 6. You issue less 3 0.0025 all and that causes both orders (because of the "all" at the end) to be cancelled and replaced with new orders. The new orders have the same conditional closes, and the same prices, but their amounts are both 0.0125 (0.015 - 0.0025). If you issued `less 3 0.0025` without "all" at the end, then only the order numbered 3 would be replaced.

#### more
`more C A F`
Increase the amount of crypto to be traded. Otherwise, this command is the same as less.#### trade*
`trade T` tells the bot to create market trades if any asset is more than T% different from what your allocation says it should be across all of your savings.  We recommend that you use the `safe` instruction first so that the bot will only report what trades it would place without actually placing them.

#### set
`set R S P`
This lists the `userref`s and prices at which buys and sells have been (and will be) placed.
R _must be_ a userref, S _must be_ either `buy` or `sell`, and P is the price you want to add or replace) for that grid point.  If the bot fails to determine either the buy price or the sell price, it displays a ?, and this will prevent the creation of a new order as described under `refnum`.  This command allows you to fix that so that Step 2.1 under `report` will work properly as described under `refnum`. 

#### quit
`quit` terminates the program.

#### risky
`risky` is intended to give you access to 

#### reset
This erases the list of `userref`s and prices at which buys and sells will be placed, but that list gets immediately rebuilt because it performs the second step in `report`.

#### auto
`auto N`
This automatically and repeatedly executes the second step of `report` and then waits N seconds.  N defaults to 60 but when you call auto with a new value for it, it is updated. 

#### manual
This stops the automatic calling of `report`.  The bot will do nothing until you give it a new command.

#### refnum
`refnum C R`
C _must be_ a `Counter` as shown by executing `list`, and it must be an order that was entered without a userref.  It will cancel the existing order and create a new one with the specified userref `R`.  All orders added by the bot (automatically and manually) have a userref.  This function is to allow you to enter an order at trade.kraken.com using the same price and no conditional close so that the bot will include it into the existing grid point with the same userref (use `set` to make sure both the buy and sell prices are known for the userref) as R.  If you use `refnum` to assign the reference number of an order that is at a different price, the behavior is undefined.

#### safe
When the bot starts, it is in "safe" mode, which means that it will not __actually__ add or cancel any orders.  The idea is that it won't do anything, but instead just show you what it would do if __safe__ were off.  Your have to enter `safe` to turn this off so that the bot will actually do things.  It allows for startup with a lot less risk with a possibly buggy bot.

### Experimental features (Not Recommended and not well tested)
#### ws - EXPERIMENTAL
This connects to Kraken's WebSockets, which, I have to warn you, send you something about every second, and sometimes silently disconnects.

## Internals

### Userref
When you place an order through trade.kraken.com or through kraken.com, it will have a `userref` of zero.  It will be ignored for the purposes of grid trading.  When you place an order through the bot, it will have a userref.

### Partial Execution
Because grid orders have conditional closes (at a price I'll call C, for close, where I'll call the price on the opening order O), a new trade is created each time a partial execution occurs, but any such new trades do not have conditional closes (which would need to have C and O swapped).  These conditional closes all have the same userref as the order that produced them. The bot detects this, sums the amount executed at price O, cancels the new orders created by the partial executions, and creates a new order for the sum at price C using the same userref and with a conditional close of its own that uses price O (see how C and O are now swapped?).  Rarely, only part of an order will have executed (at price O) and the price will move back to C and cause the conditional close(s) to execute.  If they were combined and thus already have their own conditional close (at O), new orders will appear at O, in addition to the original.  At trade.kraken.com, this looks like it will be trading too much at O, but that is because the partial execution reduced the size of the original trade, and trade.kraken.com still shows the original trade amount.  You can click the trade on trade.kraken.com to verify that the sum of the new order at O and the origianl one add to the right amount.  You got a round trip on less volume than the bot was set to try, because the market didn't fully execute your original order. All is well.

## HELP!
This code is messy and monolithic.  It works for me and I didn't want to keep waiting until I cleaned it up to publish it.  I haven't yet put enough thought into how (and whether) I would break it up into smaller files with specific purposes, so I'd love to see proposals.  One of the major motivations I have for publishing it is that as more people use a strategy like "grid trader" to balance their savings, the prices of the cryptos with which they do it will become more stable.

All calls to @nothingisdead's [Kraken-API](https://github.com/nothingisdead/npm-kraken-api) (which I have copied and renamed to kraka-djs to add CancelAll) are made through a function I called `kapi` so that any other exchange could be used by updating that funtion to translate Kraken's APIs to those of other exchanges.
