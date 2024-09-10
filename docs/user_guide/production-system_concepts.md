# Production system concepts

## Configuration files

Configuration for the system is spread across a few different places:

- System defaults
- Private config (which overrides the system defaults)
- Backtest config (which overrides the system defaults and the private config)
- Control configs: private and default
- Broker and data source specific config
- Instrument and roll configuration


### System defaults & Private config

Most configuration is stored in [/sysdata/config/defaults.yaml](/sysdata/config/defaults.yaml), with the possibility of overriding in the private configuration file `/private/private_config.yaml`, an example of which is here [/examples/production/private_config_example.yaml](/examples/production/private_config_example.yaml). Anything included in the private config will override the defaults.yaml file.

Exceptionally, the following are configuration options that are not in defaults.yaml and *must* be in private_config.yaml:

- `broker_account`: IB account id, str
- `offsystem_backup_directory`: if you're using off site backup (if not set it to a local drive or modify the backup scripts)

[Strategy configuration](#strategies)
- `strategy_list` (dict, keys are strategy names)
   - `strategy_name`
      - `load_backtests`
         - `object` class to create system instance, eg sysproduction.strategy_code.run_system_classic.runSystemClassic
         - `function` method in class to create system instance eg system_method
      - `reporting_code`
         - `function` to produce strategy reporting code eg sysproduction.strategy_code.report_system_classic.report_system_classic
- `strategy_capital_allocation` see [capital](#capital)
   - `function` to produce allocations eg sysproduction.strategy_code.strategy_allocation.weighted_strategy_allocation
   - `strategy_weights` dict of strategy names
      - `strategy_name` weight as float


The following are configuration options that are not in defaults.yaml and *may* be in private_config.yaml:

- `quandl_key`: if using quandl data
- `barchart_key`: if using barchart data
- `email_address`: if you want to get emailed errors and reports
- `email_pwd`
- `email_server`: this is the outgoing server


The following are configuration options that are in defaults.yaml and can be overridden in private_config.yaml:

[Backup paths](#data-backup)
- `backtest_store_directory` parent directory, backtests are stored under strategy_name subdirectory
- `csv_backup_directory`
- `mongo_dump_directory`
- `echo_directory`

[Broker](#linking-to-a-broker)
- `ib_ipaddress`: 127.0.0.1
- `ib_port`: 4001
- `ib_idoffset`: 100

[Database](#data-storage)
- `mongo_host`: 127.0.0.1
- `mongo_db`: 'production'

[Price collection](#update-futures-contract-historical-price-data-daily)
- `max_price_spike`: 8
- `intraday_frequency`: H

[Capital calculation](#capital)
- `production_capital_method`: 'full'
- `base_currency` (also used by backtesting, but clearly more important here)




### System backtest .yaml config file(s)

See the [user guide for backtesting](/docs/backtesting.md).

The interaction of system, private, and backtest configs can be a bit confusing. Inside a backtest (which can either be in production or sim mode), configuration options will be pulled in the following priority (1) specific backtest .yaml configuration, (2) private_config.yaml, (3) defaults.yaml file. 

Outside of the backtest code, in production configuration options are pulled in the following priority order: (1) private_config.yaml, (2) defaults.yaml file. *The production code can't see inside your backtest configuration files - it is not specific to a strategy so doesn't know which configuration to look for'*. 


### Control config files

As discussed above, these are used purely for control and monitoring purposes in [/syscontrol/control_config.yaml](/syscontrol/control_config.yaml), overridden by /private/private_control_config.yaml).

### Broker and data source specific configuration files

The following are configurations mainly for mapping from our codes to broker codes:

- [/sysbrokers/IB/ib_config_futures.csv](/sysbrokers/IB/config/ib_config_futures.csv)
- [/sysbrokers/IB/ib_config_spot_FX.csv](/sysbrokers/IB/config/ib_config_spot_FX.csv)


### Instrument and roll configuration

The following are .csv configurations used in both production and sim:

- [/data/futures/csvconfig/instrumentconfig.csv](/data/futures/csvconfig/instrumentconfig.csv) 
- [/data/futures/csvconfig/rollconfig.csv](/data/futures/csvconfig/rollconfig.csv) 


### Set up configuration


The following are used when initialising the database with it's initial configuration, but will also be used in the simulation environment:

- [/data/futures/csvconfig/spreadcosts.csv](/data/futures/csvconfig/spreadcosts.csv) 


## Capital

*Capital* is how much we have 'at risk' in our trading account. This total capital is then allocated to trading strategies; see [strategy-capital](#strategy-capital) on a [daily basis](#update-capital-and-p&l-by-polling-brokerage-account).

The simplest possible case is that your capital at risk is equal to what is in your trading account. If you do nothing else, that is how the system will behave. For all other cases, the behaviour of capital will depend on the interaction between stored capital values and the parameter value `production_capital_method` (defaults to *full* unless set in private yaml config). If you want to do things differently, you should consider modifying that parameter and/or using the [interactive tool](#interactively-modify-capital-values) to modify or initialise capital.

On initialising capital you can choose what the following values are:

- Brokerage account value (defaults to value from brokerage API). You might want to change this if you have stitched in some capital from another system, otherwise usually leave as the default
- Current capital allocated (defaults to brokerage account value).
- Maximum capital allocated (defaults to current capital). This is only used if `production_capital_method='half'`. It's effectively the 'high water mark' for your strategy. You might want to set this higher than current capital if for example you have already been running the strategy elsewhere, and it's accumulated losses. Although you can set it lower than current capital, there is no logical reason for doing that.
- Accumulated profits and losses (defaults to zero). Doesn't affect capital calculations, but is nice to know. You may want to set this if you've already been running the strategy elsewhere.

If you don't initialise capital deliberately, then the first time that is run it will populate the fields with the defaults (which will effectively mean your capital will be equal to your current trading account value).

After initialising the capital is updated [daily](#update-capital-and-p&l-by-polling-brokerage-account). First the valuation of the brokerage account is captured, and compared to the previous valuation. The difference between the valuations is your profit (or loss) since the capital was last checked, and this is written to the p&l accumulation account.

What will happen next will depend on `production_capital_method`. Read [this first](https://qoppac.blogspot.com/2016/06/capital-correction-pysystemtrade.html):

- if *full*, then your profit or loss is added to capital employed. For tidiness, maximum capital is set to be equal to current capital employed. This will result in your returns being compounded. This is the default.
- if *half* then your profit or loss is added to capital employed, until your capital is equal to the maximum capital employed. After that no further profits accrue to your capital. This is 'Kelly compatible' because losses reduce capital, but your returns will not be compounded. It's the method I use myself.
- if *fixed* then no change is made to capital. For tidiness, maximum capital is set to be equal to current capital employed. This isn't recommended as it isn't 'Kelly compatible', and if you lose money you will make exponentially increasing losses as a % of your account value. It could plausibly make sense in a small test account where you want to maintain a minimum position size.

Capital is mostly 'fire and forget', with a few exceptions which require the [interactive tool](#interactively-modify-capital-values).

### Large changes in capital

If brokerage account value has changed by more than 10% no further action is taken as it's likely this is an erroneous figure. An email is sent, and you are invited to run the [interactive tool](#interactively-modify-capital-values) choosing option 'Update capital from IB account value'. The system will get the valuation again, and if the change is still larger than 10% you will have the option of accepting this (having checked it yourself of course!).


### Withdrawals and deposits of cash or stock

The method above is neat in that it 'self recovers'; if you don't collect capital for a while it will adjust correctly when restarted. However this does mean that if you withdraw cash or securities from your brokerage account, it will look like you've made a loss and your capital will reduce. The reverse will happen if you make a deposit. This may not bother you (you actually want this to happen and aren't using the account level p&l figures), but if it does you can run the [interactive tool](#interactively-modify-capital-values) and select 'Adjust account value for withdrawal or deposit'. Make sure you are using the base currency of the account.

If you forget to do this, you should select 'Delete values of capital since time T' in the interactive tool. You can then delete the erroneous rows of capital, account for the withdrawal, and finally 'Update capital from IB account value' to make sure it has worked properly.

If you want the p&l to be correct, but do want your capital to reduce (increase), then you should use option 'Modify any/all values' after accounting for the withdrawal. Decrease (increase) the total capital figure accordingly. If you are using half compounding you also need to increase the maximum capital figure if it is lower than the new total capital figure.


### Change in capital methodology or capital base

In the [interactive tool](#interactively-modify-capital-values), option 'Modify any/all values'. Note that it's possible to change the method for capital calculation, the maximum capital or anything you wish even after you have started trading, and there may be good reasons for doing so. It's recommended that you don't delete previous capital values if you want to be able to consistently calculate your 'account level' percentage profit and loss; but the option is there to do so ('Delete everything and start again' in the interactive tool).

- Changing the capital method: this is fine, and indeed I've done it myself. The system doesn't record historic values of this parameter but you can usually infer it from the behaviour of historic capital values.
- Changing the total capital: this is also fine and can often make sense. For example you might want to start a new system off with a limited amount of capital and gradually increase it, even if the full amount is already in the brokerage account. Or temporarily reduce it because you're a scaredy cat. If using half compounding then think about your maximum capital.
- Changing your maximum capital (only affects half compounding): this might make sense but think about behaviour. If you reduce it below current total capital, then total capital will immediately reduce to that level. If you increase it above current total capital, then you will be able to accumulate profits until you reach the new maximum.

You can also change other values in the interactive tool, but be careful and make sure you know what you are doing and why!


## Strategies

Each strategy is defined in the config parameter `strategy_list`, found either in the defaults.yaml file or overridden in private yaml configuration. The following shows the parameters for an example strategy, named (appropriately enough) `example`.

```
strategy_list:
  example:
    load_backtests:
      object: sysproduction.strategy_code.run_system_classic.runSystemClassic
      function: system_method
    reporting_code:
      function: sysproduction.strategy_code.report_system_classic.report_system_classic
```

### Strategy capital

Strategy capital is allocated from [total capital](#capital). This is done by the scripted function, [update strategy capital](#allocate-capital-to-strategies). It is controlled by the configuration element below (in the defaults.yaml file, or overridden in private_config.yaml).

```
strategy_capital_allocation:
  function: sysproduction.strategy_code.strategy_allocation.weighted_strategy_allocation
  strategy_weights:
    example: 100.0
```

The allocation calls the function specified, with any other parameters passed as keywords. This default function is very simple, and just carves out the capital proportionally across all the strategies listed in `strategy_weights`. If you wish to use it you will just need to change the `strategy_weights` dict element. Alternatively, you can write your own capital allocation function.


#### Risk target

The actual risk a strategy will take depends on both it's capital and it's risk target. The risk target is set in the configuration option, `percentage_vol_target`, in the backtest configuration .yaml file for the relevant strategy (if not supplied, the defaults.yaml value is used; this is *not* overridden by private_config.yaml). Risk targets can be different across strategies.

#### Changing risk targets and/or capital

Strategy capital can be changed at any time, and indeed will usually change daily since it depends on the total capital allocated. You can also change the weight a strategy across the total strategy. A history of a strategies capital is stored, so any changes can be seen historically. Weights are not stored, but can be backed out from the total capital and strategy capital.

We do not store a history of the risk target of a strategy, so if you change the risk target this will make it difficult to compare across time. I do not advise doing this.


### System runner

System runners run overnight backtests for each of the strategies you are running (see [here](#run-updated-backtest-systems-for-one-or-more-strateges) for more details.)

The following shows the parameters for an example strategy, named (appropriately enough) `example` stored in [syscontrol/control_config.yaml](/syscontrol/control_config.yaml) (remember you must specify your own personal strategy configuration in private_control_config.yaml).

```
process_configuration_methods:
  run_systems:
    example:
      max_executions: 1
      object: sysproduction.strategy_code.run_system_classic.runSystemClassic
      backtest_config_filename: systems.provided.futures_chapter15.futures_config.yaml
```

Note the generic process parameters max_executions and frequency, both are optional, but it is strongly reccomended that you set max_executions to 1 unless you want the backtest to run multiple times throughout the day (in which case you should also set frequency, which is the gap between runs in minutes).

A system usually does the following:

- get the amount of capital currently in your trading account. See [strategy-capital](#strategy-capital).
- run a backtest using that amount of capital
- get the position buffer limits, and save these down (for the classic system, other systems may save different values down)
- store the backtest state (pickled cache) in the directory specified by the parameter csv_backup_directory (set in your private config file, or the system defaults file), subdirectory strategy name, filename date and time generated. It also copies the config file used to generate this backtest with a similar naming pattern.

As an example [here](/sysproduction/strategy_code/run_system_classic.py) is the provided 'classic' run systems function.

### Strategy order generator

Once a backtest has been run it will generate a list of desired optimal positions (for the classic buffered positions, these will include buffers). From those, and our actual current positions, we need to calculate what trades are required for execution by the run_stack_handler process.

The following shows the parameters for an example strategy, named (appropriately enough) `example` stored in [syscontrol/control_config.yaml](/syscontrol/control_config.yaml) (remember you must specify your own personal strategy configuration in private_control_config.yaml)

```
process_configuration_methods:
  run_strategy_order_generator:
    example:
      object: sysexecution.strategies.classic_buffered_positions.orderGeneratorForBufferedPositions
      max_executions: 1
```

Example of an order generator, [here](/sysexecution/strategies/classic_buffered_positions.py). Different order generators would be required for eg strategies that used limit orders, or conditional orders, or did not use buffering.


### Load backtests

It's often useful for diagnostic purposes to reload the backtest (eg for strategy reporting, discussed below). This configuration specifies the object class that is used, and the relevant method (in this case it's the same as the run_systems class, but with a different method):


```
strategy_list:
  example:
    load_backtests:
      object: sysproduction.strategy_code.run_system_classic.runSystemClassic
      function: system_method

```


### Reporting code

Reports are run that are specific for each strategy, to achieve this we need to configure which function will do the reporting:

```
strategy_list:
  example:
    reporting_code:
      function: sysproduction.strategy_code.report_system_classic.report_system_classic
```

Example [here](/sysproduction/strategy_code/report_system_classic.py).