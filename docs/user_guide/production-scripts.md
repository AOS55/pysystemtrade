# Scripts

Scripts are used to run python code which:

- runs different parts of the trading system, such as:
   - get price data
   - get FX data
   - calculate positions
   - execute trades
   - get accounting data
- fix any issues or basically interactively meddle with the system
- runs report and diagnostics, either regular or ad-hoc
- Do housekeeping duties, eg truncate log files and run backups

Script are then called by [schedulers](#scheduling), or on an ad-hoc basis from the command line.

## Script calling

I've created scripts that run under Linux, however these all just call simple python functions so it ought to be easy to 
create your own scripts in another OS. See [here](#scripts-under-other-non-linux-operating-systems) for notes about a 
method to create cross-platform executable scripts.

So, for example, here is the [run reports script](/sysproduction/linux/scripts/run_reports):

```
#!/bin/bash
. ~/.profile
. p sysproduction.run_reports.run_reports
```

In plain english this will call the python function `run_reports()`, located in `/sysproduction/run_reports.py` By convention all 'top level' python functions should be located in this folder, and the file name, script name, and top level function name ought to be the same.

Scripts are run with the following linux convenience [script](/sysproduction/linux/scripts/p) that just calls run.py with the single argument in the script that is the code reference for the function:

```
python3 run.py $1
```

run.py is a little more complicated as it allows you to call python functions that require arguments, such as [interactive_update_roll_status](/sysproduction/interactive_update_roll_status.py), and then ask the user for those arguments (with type hints).


## Script naming convention

The following prefixes are used for scripts:

- _backup: run a backup.
- _clean: run a housekeeping / cleaning process
- _interactive: run an interactive process to check or fix the system, avoiding diving into python every time something goes wrong
- _update: update data in the system (basically do one of the stages in the system)
- startup: run when the machine starts
- _run: run a regularly scheduled process.

Normally it's possible to call a process directly (eg _backup_files) on an ad-hoc basis, or it will be called regularly through a 'run' process that may do other stuff as well  (eg run_backups, runs all backup processes). Run processes are a bit complicated, as I've foolishly written my own scheduling code, so see [this section](#pysystemtrade-scheduling) for more. Some exceptions are interactive scripts which only run when called, and run_stack_handler which does not have a separate script.

## Run processes

These are listed here for convenience, but more documentation is given below in the relevant section for each script

- run_backups: Runs [backup_arctic_to_csv](#backup-arctic-data-to-csv-files), [backup state files](#backup-files): [mongo dump backup](#mongo-dump-backup)
- run_capital_updates: Runs [update_strategy_capital](#allocate-capital-to-strategies), [update_total_capital](#update-capital-and-pl-by-polling-brokerage-account): update capital
- run_cleaners: Runs [clean_truncate_backtest_states](#delete-old-pickled-backtest-state-objects), [clean_truncate_echo_files](#truncate-echo-files), [clean_truncate_log_files](#clean-up-old-logs): Clean up
- run_daily_price_updates: Runs [update_fx_prices](#get-spot-fx-data-from-interactive-brokers-write-to-mongodb-daily), [update_sampled_contracts](#update-sampled-contracts-daily), [update_historical_prices](#update-futures-contract-historical-price-data-daily), [update_multiple_adjusted_prices](#update-multiple-and-adjusted-prices-daily): daily price and contract data updates
- run_daily_fx_and_contract_updates: Runs [update_fx_prices](#get-spot-fx-data-from-interactive-brokers-write-to-mongodb-daily), [update_sampled_contracts](#update-sampled-contracts-daily).
- run_daily_update_multiple_adjusted_prices: Runs [update_multiple_adjusted_prices](#update-multiple-and-adjusted-prices-daily): daily price and contract data updates
- run_reports: Runs [all reports](#reports-1)
- run_systems: Runs [update_system_backtests](#run-updated-backtest-systems-for-one-or-more-strategies): Runs a backtest to decide what optimal positions are required
- run_strategy_order_generator: Runs [update_strategy_orders](#generate-orders-for-each-strategy): Creates trades based on the output of run_systems
- [run_stack_handler](#execute-orders): Executes trades placed on the stack by run_strategy_order_generator


## Core production system components

These control the core functionality of the system.

### Get spot FX data from interactive brokers, write to MongoDB (Daily)

Python:
```python
from sysproduction.update_fx_prices import update_fx_prices

update_fx_prices()
```

Linux script:
```
. $SCRIPT_PATH/update_fx_prices
```

Called by: `run_daily_fx_and_contract_updates`


This will check for 'spikes', unusually large movements in FX rates either when comparing new data to existing data, or within new data. If any spikes are found in data for a particular contract it will not be written. The system will attempt to email the user when a spike is detected. The user will then need to [manually check the data](#manual-check-of-fx-price-data).
.

The threshold for spikes is set in the default.yaml file, or overridden in the private config, using the parameter `max_price_spike`. Spikes are defined as a large multiple of the average absolute daily change. So for example if a price typically changes by 0.5 units a day, and `max_price_spike=6`, then a price change larger than 3 units will trigger a spike.


### Update sampled contracts (Daily)

This ensures that we are currently sampling active contracts, and updates contract expiry dates.

Python:
```python
from sysproduction.update_sampled_contracts import update_sampled_contracts
update_sampled_contracts()
```

Linux script:
```
. $SCRIPT_PATH/update_sampled_contracts
```

Called by: `run_daily_fx_and_contract_updates`


### Update futures contract historical price data (Daily)

This gets historical daily data from IB for all the futures contracts marked to sample in the mongoDB contracts database, and updates the Arctic futures price database.
If update sampled contracts has not yet run, it may not be getting data for all the contracts you need.

Python:
```python
from sysproduction.update_historical_prices import update_historical_prices
update_historical_prices()
```

Linux script:
```
. $SCRIPT_PATH/update_historical_prices
```

Called by: `run_daily_price_updates`

This will get daily closes, plus intraday data at the frequency specified by the parameter `intraday_frequency` in the defaults.yaml file (or overwritten in the private .yaml config file). It defaults to 'H: hourly'.

It will try and get intraday data first, but if it can't get any then it will not try and get daily data (this is the default behaviour. To change modify the .yaml configuration parameter `dont_sample_daily_if_intraday_fails` to False). Otherwise, there will possibly be gaps in the intraday data when we are finally able to get daily data.

It performs the following cleaning on data that it receives, depending on the .yaml parameter values that are set (defaults are shown in brackets):

- if `ignore_future_prices` is True (default: True) we ignore any prices with time stamps in the future (assumes all prices are sampled back to local time). This prevents us from early filling of Asian time zone closing prices.
- if `ignore_prices_with_zero_volumes`  is True (default: True) we ignore any price in a bar with zero volume; this reduces the amount of bad data we get.
- if `ignore_zero_prices` is True  (default: True) we ignore prices that are exactly zero. A zero price is usually erroneous.
- if `ignore_negative_prices`  is True (default: False) we ignore negative prices. Because of the crude oil incident in March 2020 I prefer to allow negative prices, and if they are errors hope that the spike checker catches them.

This will also check for 'spikes', unusually large movements in price either when comparing new data to existing data, or within new data. If any spikes are found in data for a particular contract it will not be written. The system will attempt to email the user when a spike is detected. The user will then need to [manually check the data](#manual-check-of-futures-contract-historical-price-data).
.

The threshold for spikes is set in the default.yaml file, or overridden in the private config .yaml file, using the parameter `max_price_spike`. Spikes are defined as a large multiple of the average absolute daily change. So for example if a price typically changes by 0.5 units a day, and `max_price_spike=8`, then a price change larger than 4 units will trigger a spike.

In order for this step to work, you'll need an active IB market data subscription for the instruments you wish to trade. I detailed 
my own market data subscriptions on [my blog](https://qoppac.blogspot.com/2021/05/adding-new-instruments-or-how-i-learned.html), 
reproduced here:

|            Name             | Cost per month |
|:---------------------------:|:--------------:|
|          Cboe One           |    USD 1.00    |
|        CFE Enhanced         |    USD 4.50    | 
|         Eurex Core          |    EUR 8.75    | 
|     Eurex Retail Europe     |    EUR 2.00    | 
|    Euronext Data Bundle     |    EUR 3.00    | 
|    Korea Stock Exchange     |    USD 2.00    | 
|     Singapore Exchange      |    SGD 2.00    |
|       Osaka Exchange        |    JPY 200     |

More details and latest prices on the [Interactive Brokers site](https://www.interactivebrokers.com/en/index.php?f=14193)

#### Set times when different regions download prices

To maximise efficiency, rather than doing a big end of day download, you might prefer to download different regions throughout the day.

To achieve this, add code like the following to your `private_control_config.yaml` file:

```
arguments:
  run_daily_prices_updates:
    update_historical_prices: # everything in this block is passed as **kwargs to this method
      download_by_zone:
        ASIA: '07:00'
        EMEA: '18:00'
        US: '20:00'

```

This will download Asian regional instruments at 7am, local machine time; Europe Middle East Africa at 6pm, and US at 8pm. Regions are set in the instrument configuration (provided .csv file which is then written to the database using interactive_controls, options to update configuration). 

You should also ensure that `run_daily_price_updates` has a start time set in `private_control_config.yaml` earlier than 7am, and is started by the crontab or other scheduler before 7am.


### Update multiple and adjusted prices (Daily)

This will update both multiple and adjusted prices with new futures per contract price data.

It should be scheduled to run once the daily prices for individual contracts have been updated, although you can also schedule it to run without any dependencies if you don't want to be affected by a slow download of individual prices.

Python:
```python
from sysproduction.update_multiple_adjusted_prices import update_multiple_adjusted_prices
update_multiple_adjusted_prices()
```

Linux script:
```
. $SCRIPT_PATH/update_multiple_adjusted_prices
```

Called by: `run_daily_update_multiple_adjusted_prices`


Spike checks are not carried out on multiple and adjusted prices, since they should hopefully be clean if the underlying per contract prices are clean.


### Update capital and p&l by polling brokerage account


See [capital](#capital) to understand how capital works. On a daily basis we need to check how our brokerage account value has changed. This will be used to update our total available capital, and allocate that to individual strategies.

Python:
```python
from sysproduction.update_total_capital import update_total_capital
update_total_capital()
```

Linux script:
```
. $SCRIPT_PATH/update_total_capital
```

Called by: `run_capital_update`


If the brokers account value changes by more than 10% then capital will not be adjusted, and you will be sent an email. You will then need to run `modify_account_values`. This will repeat the poll of the brokerage account, and ask you to confirm if a large change is real. The next time `update_total_capital` is run there will be no error, and all adjustments will go ahead as normal.


### Allocate capital to strategies

Allocates total capital to individual strategies. See [strategy capital](#strategy-capital) for more details. Will not work if `update_total_capital` has not run at least once, or capital has been manually initialised by `update_capital_manual`.

Python:
```python
from sysproduction.update_strategy_capital import update_strategy_capital
update_strategy_capital()
```

Linux script:
```
. $SCRIPT_PATH/update_strategy_capital
```

Called by: `run_capital_update`


### Run updated backtest systems for one or more strategies
(Usually overnight)

The paradigm for pysystemtrade is that we run a new backtest nightly, which outputs some parameters that a trading engine uses the next day. For the basic system defined in the core code those parameters are a pair of position buffers for each instrument. The trading engine will trade if the current position lies outside those buffer values.

This can easily be adapted for different kinds of trading system. So for example, for a mean reversion system the nightly backtest could output the target prices for the range. For an intraday system it could output the target position sizes and entry  / exit points. This process reduces the amount of work the trading engine has to do during the day.


Python:
```python
from sysproduction.update_system_backtests import update_system_backtests
update_system_backtests()
```

Linux script:
```
. $SCRIPT_PATH/update_system_backtests
```

Called by: `run_systems`

The code to run each strategy's backtest is defined in the configuration parameter in the control_config.yaml file (or overridden in the private_control_config.yaml file): `process_configuration_methods/run_systems/strategy_name/`. For example:

```
process_configuration_methods:
  run_systems:
    example:
      max_executions: 1
      object: sysproduction.strategy_code.run_system_classic.runSystemClassic
      backtest_config_filename: systems.provided.futures_chapter15.futures_config.yaml

```


The sub-parameters do the following:

- `object` the class of the code that runs the system, eg `sysproduction.strategy_code.run_system_classic.runSystemClassic` This class **must** provide a method `run_backtest` that has no arguments.
- `backtest_config_filename` the location of the .yaml configuration file to pass to the strategy runner eg `systems.provided.futures_chapter15.futures_config.yaml`

The following optional parameters are used only by `run_systems`:
- `max_executions` the number of times the backtest should be run on each iteration of run_systems. Normally 1, unless you have some whacky intraday system. Can be omitted.
- `frequency` how often, in minutes, the backtest is run. Normally 60 (but only relevant if max_executions>1). Can be omitted.

See [system runners](#system-runner) and scheduling processes(#process-configuration) for more details.

The backtest will use the most up to date prices and capital, so it makes sense to run this after these have updated.

### Generate orders for each strategy

Once each strategy knows what it wants to do, we generate orders. These will depend on the strategy; for the classic system we generate optimal positions that are then compared with current positions to see what trades are needed (or not). Other strategies may have specific limits ('buy but only at X or less'). Importantly these are *instrument orders*. These will then be mapped to actual [*contract level* orders](#positions-and-order-levels).

Python:
```python
from sysproduction.update_strategy_orders import update_strategy_orders
update_strategy_orders()
```

Linux script:
```
. $SCRIPT_PATH/update_strategy_orders
```

Called by: `run_strategy_order_generator`


The code to run each strategy's backtest is defined in the configuration parameter in the control_config.yaml file (or overridden in the private_control_config.yaml file): `process_configuration_methods/run_systems/strategy_name/`. For example:


```
  run_strategy_order_generator:
    example:
      object: sysexecution.strategies.classic_buffered_positions.orderGeneratorForBufferedPositions
      max_executions: 1
```

- `object` the class of the code that generates the orders, eg `sysexecution.strategies.classic_buffered_positions.orderGeneratorForBufferedPositions`. This must provide a method `get_and_place_orders` (which it will, as long as the class inherits from `orderGeneratorForStrategy`)

The following optional parameters are used only by `run_strategy_order_generator`:
- `max_executions` the number of times the generator should be run on each iteration of run_systems. Normally 1, unless you have some whacky intraday system. Can be omitted.
- `frequency` how often, in minutes, the generator is run. Normally 60 (but only relevant if max_executions>1). Can be omitted.

See [system order generator](#strategy-order-generator) and scheduling processes(#process-configuration) for more details.



### Execute orders

Once we have orders on the instrument stack (put there by the order generator), we need to execute them. This is done by the stack handler, which handles all three order stacks (instrument stack, contract stack and broker stack).

Python:
```python
from sysproduction.run_stack_handler import run_stack_handler
run_stack_handler()
```

Linux script:
```
. $SCRIPT_PATH/run_stack_handler
```

Notice that the stack handler only exists as a run process, as it's designed to run throughout the day.

The behaviour of the stack handler is extremely complex (and it's worth reading [this](#positions-and-order-levels) again, before reviewing this section). Here is the normal path an order takes:

- Instrument order created (by the strategy order generator)
- Spawn a contract order from an instrument order
- Create a broker order from a contract order and submit this to the broker
- Manage the execution of the order (technically done by execution algo code, but this is called by the stack handler), and note any fills that are returned
- Pass fills upwards; if a broker order is filled then the contract order should reflect that, and if a contract order is filled then an instrument order should reflect that
- Update position tables when fills are received
- Handle completed orders (which are fully filled) by deleting them from the stack after copying them to the historic order table

In addition the stack handler will:

- Check that the broker and database positions are aligned at contract level, if not then it will lock the instrument so it can't be traded (locks can be cleared automatically once positions reconcile again, or using [interactive_order_stack](#interactive-order-stack).
- Generate roll orders if a [roll status](#interactively-roll-adjusted-prices) is FORCE or FORCELEG
- Safely clear the order stacks at the end of the day or when the process is stopped by cancelling existing orders, and deleting them from the order stack.

That's quite a list, hence the use of the [interactive_order_stack](#interactive-order-stack) to keep it in check!

The stack handler will also periodically sample the bid/ask spread on all instruments. This is used to help with cost analysis (see the relevant [reports](#reports-1) section).

## Interactive scripts to modify data

### Manual check of futures contract historical price data
(Whenever required)

You should run these if the normal price collection has identified a spike (for which you'd be sent an email, if you've set that up).

Python:
```python
from sysproduction.interactive_manual_check_historical_prices import interactive_manual_check_historical_prices
interactive_manual_check_historical_prices(instrument_code)
```

Linux script:
```
. $SCRIPT_PATH/interactive_manual_check_historical_prices
```

The script will pull in data from interactive brokers, and the existing data. It will behave in the same way as `update_historical_prices`, except you have the option to change the relevant configuration items before you begin.


Next it will check for spikes. If any spikes are found, then the user is interactively asked if they wish to (a) accept the spiked price, (b) use the previous time periods price instead, or (c) type a number in manually. You should check another data source to see if the spike is 'real', if so accept it, otherwise type in the correct value. Using the previous time periods value is only advisable if you are fairly sure that the price change wasn't real and you don't have a source to check with.

If a new price is typed in then that is also spike checked, to avoid fat finger errors. So you may be asked to accept a price you have have just typed in manually if that still results in a spike. Accepted or previous prices are not spike checked again.

Spikes are only checked on the FINAL price in each bar, and the user is only given the opportunity to correct the FINAL price. If the FINAL price is changed, then the OPEN, HIGH, and LOW prices are also modified; adding or subtracting the same adjustment that was made to the final price. The system does not currently use OHLC prices, but you should be aware of this creating potential inaccuracies. VOLUME figures are left unchanged if a price is corrected.

Once all spikes are checked for a given contract then the checked data is written to the database, and the system moves on to the next contract.


### Manual check of FX price data
(Whenever required)

You should run these if the normal price collection has identified a spike (for which you'd be sent an email, if you've set that up).


Python:
```python
from sysproduction.interactive_manual_check_fx_prices import interactive_manual_check_fx_prices
interactive_manual_check_fx_prices(fx_code)
```

Linux script:
```
. $SCRIPT_PATH/interactive_manual_check_fx_prices
```

See [manual check of futures contract prices](#manual-check-of-futures-contract-historical-price-data) for more detail. Note that as the FX data is a single series, no adjustment is required for other values.


### Interactively modify capital values

Python:
```python
from sysproduction.interactive_update_capital_manual import interactive_update_capital_manual
interactive_update_capital_manual()
```

Linux script:
```
. $SCRIPT_PATH/interactive_update_capital_manual
```

See [capital](#capital) to understand how capital works.
This function is used interactively to control total capital allocation in any of the following scenarios:

- You want to initialise the total capital available in the account. If this isn't done, it will be done automatically when `update_total_capital` runs with default values. The default values are brokerage account value = total capital available = maximum capital available (i.e. you start at HWM), with accumulated profits = 0. If you don't like any of these values then you can initialise them differently.
- You have made a withdrawal or deposit in your brokerage account, which would otherwise cause the apparent available capital available to drop, and needs to be ignored
- There has been a large change in the value of your brokerage account. A filter has caught this as a possible error, and you need to manually confirm it is ok.
- You want to delete capital entries for some recent period of time (perhaps because you missed a withdrawal and it screwed up your capital)
- You want to delete all capital entries (and then probably reinitialise). This is useful if, for example, you've been running a test account and want to move to production.
- You want to make some other modification to one or more capital entries. Only do this if you know exactly what you are doing!




### Interactively roll adjusted prices
(Whenever required)

Allows you to change the roll state and roll from one priced contract to the next.

Python:
```python
from sysproduction.interactive_update_roll_status import interactive_update_roll_status
interactive_update_roll_status(instrument_code)
```

Linux script:
```
. $SCRIPT_PATH/interactive_update_roll_status
```

There are four different modes that this can be run in:

- Manually input instrument codes and manually decide when to roll
- Cycle through instrument codes automatically, but manually decide when to roll
- Cycle through instrument codes automatically, auto decide when to roll, manually confirm rolls
- Cycle through instrument codes automatically, auto decide when to roll, automatically roll

#### Manually input instrument codes and manually decide when to roll

You enter the instrument code you wish to think about rolling.

The first thing the process will do is create and print a roll report. See the [roll report](#roll-report-daily) for more information on how to interpret the information shown. You will then have the option of switching between roll modes. Not all modes will be allowed, depending on the current positions that you are holding and the current roll state.

The possible options are:

- No roll. Obvious.
- Passive. This will tactically reduce positions in the priced contract, and open new positions in the forward contract.
- Force. This will pause all normal trading in the relevant instrument, and the stack handler will create a calendar spread trade to roll from the priced to the forward contract.
- Force legs. This will pause all normal trading, and create two outright trades (closing the priced contract position, opening a forward position).
- Roll adjusted. This is only possible if you have no positions in the current price contract. It will create a new adjusted and multiple price series, hence the current forward contract will become the new priced contract (and everything else will shift accordingly). Adjusted price changes are manually confirmed before writing to the database.

Once you've updated the roll status you have the option of choosing another instrument, or aborting.

NOTE: Adjusted price rolling will fail if the system can't find aligned prices for the current and forward contract. In this case you have the option of forward filling the prices - it will make the roll less accurate, but at least you can keep that instrument in your system.

#### Cycle through instrument codes automatically, but manually decide when to roll

This chooses a subset of instruments that are expiring soon. You will be prompted for the number of days ahead you want to look for expiries. This then behaves exactly like the manual option above, except it automatically cycles through the relevant subset of instruments.

#### Cycle through instrument codes automatically, auto decide when to roll, manually confirm rolls

Again this will first choose a subset of instruments that are expiring soon. What happens next will depend on the parameters you have decided upon:

- If the volume in the forward contract is less than the required relative volume, we do nothing
- If the relative volume is fine and you have no position in the priced contract, then we automatically decide to roll adjusted prices
- If the relative volume is fine and you have a position in the priced contract, and you have asked to manually input the required state on a case by case basis: that's what will happen
- If the relative volume is fine and you have a position in the priced contract, and you have NOT asked to manually input the required state, then the state will automatically be changed to one of passive, force, or force leg (as selected). I strongly recommend using Passive rolling here, and then manually changing individual instruments if required.

If a decision is made to roll adjusted prices, you will be asked to confirm you are happy with the prices changes before they are written to the database.

I recommend that you run a roll report after doing this to see what state things are in.

#### Cycle through instrument codes automatically, auto decide when to roll, automatically roll

This is exactly like the previous option, except that if a decision is made to roll adjusted prices, this will happen automatically without user confirmation.

## Menu driven interactive scripts

The remaining interactive scripts allow you to view and control a large array of things, and hence are menu driven. There are three such scripts:

- interactive_controls: Trade limits, position limits, process control and monitoring
- interactive_diagnostics: View backtest objects, generate ad hoc reports, view logs/emails and errors; view prices, capital, positions & orders, and configuration.
- interactive_order_stack: View order stacks and positions, create orders, net/cancel orders, lock/unlock instruments, delete and clean up the order stack.

Menus are nested, and a common pattern is that *return* will go back a step, or exit.

### Interactive controls

Tools to control the system's behaviour, including operational risk controls.

Python:
```python
from sysproduction.interactive_controls import interactive_controls 
interactive_controls()
```

Linux script:
```
. $SCRIPT_PATH/interactive_controls
```

#### Trade limits

We can set limits for the maximum number of trades we will do over a given period, and for a specific instrument, or a specific instrument within a given strategy. Limits are applied within run_stack_handler whenever a broker order is about to be generated from a contract order. Options are as follows:

- View limits
- Change limits (instrument, instrument & strategy)
- Reset limits (instrument, instrument & strategy): helpful if you have reached your limit but want to keep trading, without increasing the limits upwards
- Autopopulate limits

Autopopulate uses current levels of risk to estimate the appropriate trade limit. So it will make limits smaller when risk is higher, and vice versa. It makes a lot of assumptions when setting limits: that all your strategies have the same risk limit (which you can set), and the same IDM (also can be modified), and that all instruments have the same instrument weight (which you can set), and trade at the same speed (again you can set the maximum proportion of typical position traded daily). It does not use actual instrument weights, and it only sets limits that are global for a particular instrument. It also assumes that trade sizes scale with the square root of time for periods greater than one day.

#### Position limits

We can set the maximum allowable position that can be held in a given instrument, or by a specific strategy for an instrument. An instrument trade that will result in a position which exceeds this limit will be rejected (this occurs when run_strategy_order_generator is run). We can:

- View limits
- Change limits (instrument, instrument & strategy)
- Autopopulate limits

Autopopulate uses current levels of risk to estimate the appropriate position limit. So it will make position limits smaller when risk is higher, and vice versa. It makes a lot of assumptions when setting limits: that all your strategies have the same risk limit (which you can set), and the same IDM (also can be modified), and that all instruments have the same instrument weight (which you can set). It does not use actual instrument weights, and it only sets limits that are global for a particular instrument.

The dynamic optimisation strategy will also use position limits in it's optimisation in production (not in backtests, since fixed position limits make no sense for a historical backtest).

#### Trade control / override

Overrides allow us to reduce positions for a given strategy, for a given instrument (across all strategies), or for a given instrument & strategy combination. They are either:

- a multiplier, between 0 and 1, by which we multiply the desired . A multiplier of 1 is equal to 'normal', and 0 means 'close everything'
- a flag, allowing us only to submit trades which reduce our positions
- a flag, allowing no trading to occur in the given instrument.

Overrides are also set as a result of configured information about different instruments; see the [instruments documentation](/docs/instruments.md) for more detail.

Instrument trades will be modified to achieve any required override effect (this occurs when run_strategy_order_generator is run). We can:

- View overrides
- Update / add / remove override (for strategy, instrument, or instrument & strategy)

See the [instruments documentation](instruments.md) for more discussion on overrides.

#### Broker client IDs

Allows us to release any unused client IDs. Don't do this if any IB connections are active! Automatically called by the startup script.

#### Process control & monitoring

Allows us to control how processes behave.

See [scheduling](#pysystemtrade-scheduling).


##### View processes

Here's an example of the relevant output, start/end times, currently running, status, and PID (process ID).

```
run_capital_update            : Last started 2020-12-08 15:56:32.323000 Last ended status 2020-12-08 14:31:46.601000 GO      PID 63652.0    is running
run_daily_prices_updates      : Last started 2020-12-08 12:36:04.850000 Last ended status 2020-12-08 13:54:47.885000 GO      PID None       is not running
run_systems                   : Last started 2020-12-08 14:10:30.485000 Last ended status 2020-12-08 14:42:40.945000 GO      PID None       is not running
run_strategy_order_generator  : Last started 2020-12-08 14:46:28.013000 Last ended status 2020-12-08 14:49:44.081000 GO      PID None       is not running
run_stack_handler             : Last started 2020-12-08 13:43:53.628000 Last ended status 2020-12-08 14:22:55.388000 GO      PID None       is not running
run_reports                   : Last started 2020-12-08 15:48:55.595000 Last ended status 2020-12-07 23:43:13.476000 GO      PID 62388.0    is running
run_cleaners                  : Last started 2020-12-08 14:51:19.380000 Last ended status 2020-12-08 14:51:40.609000 GO      PID None       is not running
run_backups                   : Last started 2020-12-08 14:54:04.856000 Last ended status 2020-12-08 00:05:48.444000 GO      PID 61604.0    is running
```

You can use the PID to check using the Linux command line eg `ps aux | grep 86140` if a process really is running (in this case I'm checking if run_capital_update really is still going), or if it's abnormally aborted (in which case you will need to change it to 'not running' before relaunching - see below). This is also done automatically by the [system monitor and/or dashboard](/docs/dashboard_and_monitor.md), if running.

Note that processes that have launched but waiting to properly start (perhaps because it is not their scheduled start time, or because another process has not yet started) will be shown as not running and will have no PID registered. You can safely kill them.

#####  Change status of process

You can change the status of any process to STOP, GO or NO RUN. A process which is NO RUN will continue running, but won't start again. This is the correct way to stop processes that you want to kill, as it will properly update their process state and (importantly in the case of run stack handler) do a graceful exit. Stop processes will only stop once they have close running their current method, which means for run_systems and run_strategy_order_generator they will stop when the current strategy has close processing (which can take a while!).

If a process refuses to STOP, then as a last resort you can use `kill NNNN` at the command line where NNNN is the PID, but there may be data corruption, or weird behaviour (particularly if you do this with the stack handler), and you will definitely need to mark it as close (see below).

Marking a process as START won't actually launch it, you will have to do this manually or wait for the crontab to run it. Nor will the process run if it's preconditions aren't met (start and end time window, previous process).

##### Global status change

Sometimes you might want to mark all processes as STOP (emergency shut down?) or GO (post emergency restart).


#####  Mark as close

This will manually mark a process as close. This is done automatically when a process finishes normally, or is told to stop, but if it terminates unexpectedly then the status may well be set as 'running', which means a new version of the process can't be launched until this flag is cleared. Marking a process as close won't stop it if it is still running! Use 'change status' instead. Check the process PID isn't running using `ps aux | grep NNNNN` where NNNN is the PID, before marking it as close.

Note that the startup script will also mark all processes as close (as there should be no processes running on startup). Also if you run the next option ('mark all dead processes as close') this will be automatic.

#####  Mark all dead processes as close

This will check to see if a process PID is active, and if not it will mark a process as close, assumed crashed. This is also done periodically by the [system monitor and/or dashboard](/docs/dashboard_and_monitor.md), if running.

#####  View process configuration

This allows you to see the configuration for each process, either from `control_config.yaml` or the `private_control_config.yaml` file. See [scheduling](#pysystemtrade-scheduling).

#### Update configuration

These options allow you to update, or suggest how to update, the instrument and roll configuration.

- Auto update spread cost configuration based on sampling and trades
- Safely modify roll parameters
- Check price multipliers are consistent with IB and configuration file


### Interactive diagnostics

Tools to view internal diagnostic information.

Python:
```python
from sysproduction.interactive_diagnostics import interactive_diagnostics
interactive_diagnostics()
```

Linux script:
```
. $SCRIPT_PATH/interactive_diagnostics
```

#### Backtest objects

It's often helpful to examine the backtest output of run_systems to understand where particular trades came from (above and beyond what the [strategy report](#strategy-report) gives you. These are saved as a combination of pickled cache and configuration .yaml file, allowing you to see the calculations done when the system ran.

##### Output choice

First of all you can choose your output:

- Interactive python. This loads the backtest, and effectively opens a small python interpreter (actually it just runs eval on the input).
- Plot. This loads a menu allowing you to choose a data element in the backtest, which is then plotted (will obviously fail on headless servers)
- Print. This loads a menu allowing you to choose a data element in the backtest, which is then printed to screen.
- HTML. This loads a menu allowing you to choose a data element in the backtest, which is then output to an HTML file (outputs to ~/temp.html), which can easily be web browsed

##### Choice of strategy and backtest

Next you can choose your strategy, and the backtest you want to see- all backtests are saved with a timestamp (normally these are kept for a few days). The most recent backtest file is the default.

##### Choose stage / method / arguments

Unless you're working in 'interactive python' mode, you can then choose the stage and method for which you want to see output. Depending on exactly what you've asked for, you'll be asked for other parameters like the instrument code and possibly trading rule name. The time series of calling the relevant method will then be shown to you using your chosen output method.

##### Alternative python code

If you prefer to do this exercise in your python environment, then this will interactively allow you to choose a system and dated backtest, and returns the system object for you to do what you wish.

```python

from sysproduction.data.backtest import user_choose_backtest
backtest = user_choose_backtest()
system = backtest.system
```


#### Reports

Allows you to run any of the [reports](#reports-1) on an ad-hoc basis.

#### Logs, errors, emails

Allows you to look at various system diagnostics.

##### View stored emails

The system sends emails quite a bit: when critical errors occur, when reports are sent, and when price spikes occur. To avoid spamming a user, it won't send an email with the same subject line as a previous email sent in the last 24 hours. Instead these emails are stored, and if you view them here they will be printed and then deleted from the store. The most common case is if you get a large price move which affects many different contracts for the same instrument; the first spike email will be sent, and the rest stored.


#### View prices

View dataframes for historical prices. Options are:

- Individual futures contract prices
- Multiple prices
- Adjusted prices
- FX prices

#### View capital

View historical series of capital. See [here](#capital) for more details on how capital works. You can see the:

- Capital for a strategy
- Total capital (across all strategies): current capital
- Total capital: broker valuation
- Total capital: maximum capital
- Total capital: accumulated returns

#### Positions and orders

View historic series of positions and orders. Options are:

- Optimal position history (instruments for strategy)
- Actual position history (instruments for strategy)
- Actual position history (contracts for instrument)
- List of historic instrument level orders (for strategy)
- List of historic contract level orders (for strategy and instrument)
- List of historic broker level orders (for strategy and instrument)
- View full details of any individual order (of any type)


#### Instrument configuration

##### View instrument configuration data

View the configuration data for a particular instrument, eg for EDOLLAR:

```{'Description': 'US STIR Eurodollar', 'Exchange': 'GLOBEX', 'Pointsize': 2500.0, 'Currency': 'USD', 'AssetClass': 'STIR', 'Slippage': 0.0025, 'PerBlock': 2.11, 'Percentage': 0.0, 'PerTrade': 0.0}```

Note there may be further configuration stored in other places, eg broker specific.

#####  View contract configuration data

View the configuration for a particular contract, eg:

```
{'contract_date_dict': {'expiry_date': (2023, 6, 19), 'contract_date': '202306', 'approx_expiry_offset': 0}, 'instrument_dict': {'instrument_code': 'EDOLLAR'}, 'contract_params': {'currently_sampling': True}}
Rollcycle parameters hold_rollcycle:HMUZ, priced_rollcycle:HMUZ, roll_offset_day:-1100.0, carry_offset:-1.0, approx_expiry_offset:18.0
```

See [here](#interactively-roll-adjusted-prices) to understand roll parameters.

### Interactive order stack

Allows us to examine and control the [various order stacks](#positions-and-order-levels).

Python:
```python
from sysproduction.interactive_order_stack import interactive_order_stack
interactive_order_stack()
```

Linux script:
```
. $SCRIPT_PATH/interactive_order_stack
```

#### View

Options are:

- View specific order (on any stack)
- View instrument order stack
- View contract order stack
- View broker order stack (as stored in the local database)
- View broker order stack (will get all the active orders and completed trades from the broker API)
- View positions (optimal, instrument level, and contract level from the database; plus contract level from the broker API)

#### Create orders

Orders will normally be created by run_strategy_order_generator or by run_stack_handler, but sometimes its useful to do these manually.

##### Spawn contract orders from instrument orders

If the stack handler is running it will periodically check for new instrument orders, and then create child contract orders. However you can do this manually. Use case for this might be debugging, or if you don't trust the stack handler and want to do everything step by step, or if you're trading manually (in which case the stack handler won't be running).

##### Create force roll contract orders

If an instrument is in a FORCE or FORCELEG roll status (see [rolling](#interactively-roll-adjusted-prices)), then the stack handler will periodically create new roll orders (consisting of a parent instrument order that is an intramarket spread, allocated to the phantom 'rolling' strategy, and a child contract order). However you can do this manually. Use case for this might be debugging, or if you don't trust the stack handler and want to do everything step by step, or if you're trading manually (in which case the stack handler won't be running).


##### Create (and try to execute...) IB broker orders

If the stack handler is running it will periodically check for contract orders that aren't completely filled, and generate broker orders that it will submit to the broker and then pass to an algo to manage the execution. However you can do this manually. Use case for this might be debugging, or if you don't trust the stack handler and want to do everything step by step.


##### Balance trade: Create a series of trades and immediately fill them (not actually executed)

Ordinarily the stack handler will pick up on any fills, and act accordingly. However there are times when this might not happen. If a position is closed by IB because it is close to expiry, or you submit a manual trade on another platform, or if the stack handler crashes after submitting the order but executing the fill... the possibilities are endless. Anyway, this is a serious problem because the positions you actually have (in the brokers records) won't be reflected in the position database which will be reported in the [reconcile report](#reconcile-report)) - a condition which, when detected, will lock the instrument so it can't be traded until the problem is solved. Less seriously, you'll be missing the trade from your historic trade database.

To get round this you should submit a balance trade, which will ripple through the databases like a normal trade, but won't actually be sent to the broker for execution; thus replacing the missing trade.

##### Balance instrument trade: Create a trade just at the strategy level and fill (not actually executed)

Ordinarily the strategy level positions (for instruments, per strategy, summed across contracts) should match the contract level positions (for instruments and contracts, summed across strategies). However if for some reason an order goes astray you will end up with a mismatch (which will be reported in the [reconcile report](#reconcile-report)). To solve this you can submit an order just at the strategy level (not allocated to a specific contract) which will solve the problem but isn't actually executed.


##### Manual trade: Create a series of trades to be executed

Normally run_strategy_order_generator creates all the trades you should need, but sometimes you might want to generate a manual trade. This could be for testing, because you urgently want to close a position (which you ought to do with an [override](#trade-control--override)), or because something has gone wrong with the roll process and you're stuck in a contract that the system won't automatically close.

Manual trades are not the same as balance trades: they will actually go to the broker for execution!

Note, you can create a manual spread trade: enter the instrument position as zero, ask to create contract orders, then enter the number of legs you want.

##### Cash FX trade

Cash FX isn't the primary asset class traded in pysystemtrade, but we trade FX anyway without realising it (unless you only trade in your account currency). When you buy or sell a futures contract in another country, it will require margin. If you don't have margin in that currency, then IB will lend it to you. Borrowing money in foreign currencies incurs a spread, so it's better to do a spot FX trade, converting your domestic currency (which will probably be earning 0% interest anyway) into the margin currency. As a rule, I periodically optimise my currency holdings so I have a diversified portfolio of currency. Others may prefer to 'sweep' all excess foreign currencies back to their home currency to reduce unwanted currency Beta. Or you could live dangerously, and try and maintain a larger balance in currencies with higher positive deposit rates (basically the carry trade).

First of all you see the balances in each currency. Note, these aren't excess or uncleared balances, so you will need to run an IB report to see what you really have spare or are short of. You can then create an FX trade. In specifying the pairing, don't forget there is a market convention so if you get the pairing the wrong way round your order will be rejected.

#### Netting, cancellation and locks


##### Cancel broker order

If you have an order that has been submitted, and you want to cancel it, here is where you come.

##### Net instrument orders

The complexity of the order stacks is there for a reason; it allows different kinds of strategies to submit trades at the same time. One advantage of this is that orders can be netted. Ordinarily the stack handler will do this netting, but you might want to trigger it manually.

##### Lock/unlock order

There is a 'lock' in the order database, basically an explicit flag preventing the order from being modified. Certain operations which span multiple data tables will impose locks first so the commit does not partially fail (I'm using noSQL so there is no explicit cross table commit available). If operations fail mid lock they will usually try and fall back and remove locks, but this doesn't always work out. So it sometimes necessary to manually unlock orders, and for symmetry manually lock them.


##### Lock/unlock instrument code

If there is a mismatch between the brokers record of positions, and ours, then a lock will be placed on the instrument and no broker trades can be issued for it. This is done automatically by the stack handler. Once a mismatch clears, the stack handler will remove the lock. But you can also do these operations manually.

Note: if you want to avoid trading in an instrument for some other reason, use an [override](#trade-control--override) not a lock: a lock will be automatically cleared by the system, an override won't be.


##### Unlock all instruments

If the broker API has gone crazy or died for some reason then all instruments with active positions will be locked. This is a quick way of unlocking them.

##### Remove Algo lock on contract order

When an algo begins executing a contract order (in part or in full), it locks it. That lock is released when the order has close executing. If the stack handler crashes before that can happen, then no other algo can execute it. Although the order will be deleted in the normal end of day stack clean up, if you can't wait that long you can manually clear the problem.


#### Delete and clean

##### Delete entire stack (CAREFUL!)

You can delete all orders on any of the three stacks. I can't even begin to describe how bad an idea this is. If you want to stop trading urgently then I strongly advise using a [STOP command](#change-status-of-process) on run_stack_handler, or calling the end of day process manually (described below) - which will leave the stack handler running. Only use when debugging or testing, if you really know what you're doing.

##### Delete specific order ID (CAREFUL!)

You can delete a specific live order from the database. Again, this will most likely lead to all kinds of weird side effects. This won't cancel the order either; the broker will continue to try and execute it. Only use when debugging or testing, if you really know what you're doing.

##### End of day process (cancel orders, mark all orders as complete, delete orders)

When run_stack_handler has done it's work (either because it's time is up, or it has received a STOP command) it will run a clean up process. First it will cancel any active orders. Then it will mark all orders as complete, which will update position databases, and move orders to historical data tables. Finally it deletes every order from every stack; ensuring no state continues to the next day (which could lead to weird behaviour).

I strongly advise running this rather than deleting the stack, unless you know exactly what you're doing and have a very valid reason for doing it!


## Reporting, housekeeping and backup scripts

### Run all reports

Python:
```python
from sysproduction.run_reports import run_reports
run_reports()
```

Linux script:
```
. $SCRIPT_PATH/run_reports
```

See [reporting](#reports-1) for details on individual reports.


### Delete old pickled backtest state objects

Python:
```python
from sysproduction.clean_truncate_backtest_states
clean_truncate_backtest_states()
```

Linux script:
```
. $SCRIPT_PATH/clean_truncate_backtest_states
```

Called by: `run_cleaners`

Every time run_systems runs it creates a pickled backtest and saves a copy of it's configuration file. This makes it easier to use them for [diagnostic purposes](#backtest-objects).

However these file are large! So we delete anything more than 5 days old.



### Clean up old logs


Python:
```python
from sysproduction.clean_truncate_log_files import clean_truncate_log_files
clean_truncate_log_files()
```

Linux command line:
```
cd $SCRIPT_PATH
. clean_truncate_log_files
```

Called by: `run_cleaners`

I love logging! Which does mean there are a lot of log entries. This deletes any that are more than a month old.

### Truncate echo files

Python:
```python
from sysproduction.clean_truncate_echo_files import clean_truncate_echo_files
clean_truncate_echo_files()
```

Linux command line:
```
cd $SCRIPT_PATH
. clean_truncate_echo_files
```

Called by: `run_cleaners`

Every day we generate echo files with extension .txt; this process renames ones from yesterday and before with a date suffix, and then deletes anything more than 30 days old.


### Backup Arctic data to .csv files

Python:

```python
from sysproduction.backup_db_to_csv import backup_arctic_to_csv

backup_arctic_to_csv()
```

Linux script:
```
. $SCRIPT_PATH/backup_arctic_to_csv
```

Called by: `run_backups`

See [backups](#mongo--csv-data).

- It copies data out of mongo and Arctic into a temporary .csv directory
- It then copies the .csv files  to the backup directory,  "offsystem_backup_directory", subdirectory /csv



### Backup state files


Python:
```python
from sysproduction.backup_state_files import backup_state_files
backup_state_files()
```

Linux script:
```
. $SCRIPT_PATH/backup_files
```

Called by: `run_backups`

It copies backtest pickle and config files to the backup directory,  "offsystem_backup_directory", subdirectory /statefile

**Important**: the backed up files will contain any data you have added to your private config, some of which may be 
sensitive (e.g., IB account number, email address, email password). If you 
choose to store these files with a cloud storage provider or backup service, you should consider encrypting them first
(some services may do this for you, but many do not).

### Backup mongo dump


Python:
```python
from sysproduction.backup_mongo_data_as_dump import *
backup_mongo_data_as_dump()
```

Linux script:
```
. $SCRIPT_PATH/backup_mongo_data_as_dump
```

Called by: `run_backups`

- Firstly it dumps the mongo databases to the local directory specified in the config parameter (defaults.yaml or private config yaml file) "mongo_dump_directory".
- Then it copies those dumps to the backup directory specified in the config parameter "offsystem_backup_directory", subdirectory /mongo


### Start up script


Python:
```python
from sysproduction.startup import startup
startup()
```

Linux script:
```
. $SCRIPT_PATH/startup
```

There is some housekeeping to do when a machine starts up, primarily in case it crashed and did not close everything gracefully:

- Clear IB client IDs: Do this when the machine restarts and IB is definitely not running (or we'll eventually run out of IDs)
- Mark all running processes as close

## Scripts under other (non-linux) operating systems

There is a built-in Python mechanism for creating command line executables; it may make sense for those who want to
have a production instance of pysystemtrade on MacOS or Windows. Or for Linux users who would prefer to use the standard 
method than the supplied scripts. The mechanism is provided by the packaging tools, and configured in setup.py. See the 
[docs here](https://python-packaging.readthedocs.io/en/latest/command-line-scripts.html#the-console-scripts-entry-point). 

You add a new *entry_points* section to `setup.py` file like:

```
...
test_suite="nose.collector",
include_package_data=True,
entry_points={
    "console_scripts": [
        "interactive_controls = sysproduction.interactive_controls:interactive_controls",
        "interactive_diagnostics = sysproduction.interactive_diagnostics:interactive_diagnostics",
        "interactive_manual_check_fx_prices = sysproduction.interactive_manual_check_fx_prices:interactive_manual_check_fx_prices",
        "interactive_manual_check_historical_prices = sysproduction.interactive_manual_check_historical_prices:interactive_manual_check_historical_prices",
        "interactive_order_stack = sysproduction.interactive_order_stack:interactive_order_stack",
        "interactive_update_capital_manual = sysproduction.interactive_update_capital_manual:interactive_update_capital_manual",
        "interactive_update_roll_status = sysproduction.interactive_update_roll_status:interactive_update_roll_status",
    ],
},
...
```

When `setup.py install` is executed, the above config would generate executable *shims* for all the interactive scripts
into the current python path, which when run, would execute the configured function. There are several advantages to 
this method:
- no need for additional code or scripts
- cross-platform compatibility
- standard Python
- no need to manipulate PATH or other env variables
- name completion
- works with any virtualenv flavour