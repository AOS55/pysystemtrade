# How do I?

The following are common back-testing questions

## Experiment with a single trading rule and instrument

Although the project is intended mainly for working with trading systems, it's
possible to do some limited experimentation without building a system. See [the
introduction](introduction.md) for an example.

## Create a standard futures backtest

This creates the staunch systems trader example defined in chapter 15 of my
book, using the csv data that is provided, and gives you the position in the
Eurodollar market:

```python
from systems.provided.futures_chapter15.basesystem import futures_system
system=futures_system()
system.portfolio.get_notional_position("EDOLLAR")
```
See [standard futures system](#futures_system) for more.


## Create a futures backtest which estimates parameters

This creates the staunch systems trader example defined in chapter 15 of my
book, using the csv data that is provided, and estimates forecast scalars,
instrument and forecast weights, and instrument and forecast diversification
multipliers:

```python
from systems.provided.futures_chapter15.estimatedsystem import futures_system
system=futures_system()
system.portfolio.get_notional_position("EDOLLAR")
```

See [estimated futures system](#futures_system).



## See intermediate results from a backtest

This will give you the raw forecast (before scaling and capping) of one of the
EWMAC rules for Eurodollar futures in the standard futures backtest:

```python
from systems.provided.futures_chapter15.basesystem import futures_system
system=futures_system()
system.rules.get_raw_forecast("EDOLLAR", "ewmac64_256")
```

For a complete list of possible intermediate results, use `print(system)` to
see the names of each stage, and then `stage_name.methods()`. Or see [this
table](#table_system_stage_methods) and look for rows marked with **D** for
diagnostic. Alternatively type `system` to get a list of stages, and
`system.stagename.methods()` to get a list of methods for a stage (insert the
name of the stage, not stagename).


## See how profitable a backtest was

```python
from systems.provided.futures_chapter15.basesystem import futures_system
system=futures_system()
system.accounts.portfolio().stats() ## see some statistics
system.accounts.portfolio().curve().plot() ## plot an account curve
system.accounts.portfolio().percent.curve().plot() ## plot an account curve in percentage terms
system.accounts.pandl_for_instrument("US10").percent.stats() ## produce % statistics for a 10 year bond
system.accounts.pandl_for_instrument_forecast("EDOLLAR", "carry").sharpe() ## Sharpe for a specific trading rule variation
```

For more information on what statistics are available, see the [relevant guide
section](#standard_accounts_stage).



<a name="change_backtest_parameters"> </a>

## Change backtest parameters

The backtest looks for its configuration information in the following places:

1. Elements in the configuration object
2. If not found, in: the private yaml config if it exists here `/private/private_config.yaml`
3. If not found, in: Project defaults

Configuration objects can be loaded from [yaml](https://pyyaml.org/) files, or
created with a dictionary. This suggests that you can modify the systems
behaviour in any of the following ways:

1. Change or create a configuration yaml file, read it in, and create a new
   system
2. Change a configuration object in memory, and create a new system with it.
3. Change a configuration object within an existing system (advanced)
4. Create a private config yaml `/private/private_config.yaml` (this useful if you want to make a global change that affects all your backtest)
5. Change the project defaults (definitely not recommended)

For a list of all possible configuration options, see [this
table](#Configuration_options).

If you use options 2 or 3, you can [save the config](#save_config) to a yaml
file.

### Option 1: Change the configuration file

Configurations in this project are stored in [yaml](https://pyyaml.org) files.
Don't worry if you're not familiar with yaml; it's just a nice way of creating
nested dicts, lists and other python objects in plain text. Just be aware that
indentations are important, just in like python, to create nesting.

You can make a new config file by copying this
[one](/systems/provided/futures_chapter15/futuresconfig.yaml), and modifying
it. Best practice is to save this as
`pysystemtrade/private/this_system_name/config.yaml` (you'll need to create a
couple of directories first).

You should then create a new system which points to the new config file:

```python
from sysdata.config.configdata import Config
from systems.provided.futures_chapter15.basesystem import futures_system

my_config=Config("private.this_system_name.config.yaml")
system=futures_system(config=my_config)
```

See [here](#filenames) for how to specify filenames in pysystemtrade.

### Option 2: Change the configuration object; create a new system

We can also modify a configuration object from a loaded system directly, and
then create a new system with it:

```python
from systems.provided.futures_chapter15.basesystem import futures_system
system=futures_system()
new_config=system.config

new_idm=1.1 ## new IDM

new_config.instrument_div_multiplier=new_idm

## Heres an example of how you'd change a nested parameter
## If the element doesn't yet exist in your config:

system.config.volatility_calculation=dict(days=20)

## If it does exist:
system.config.volatility_calculation['days']=20


system=futures_system(config=new_config)
```

This is useful if you're experimenting interactively 'on the fly'.


### Option 3: Change the configuration object within an existing system (not recommended - advanced)

If you opt for (3) you will need to understand about [system caching](#caching)
and [how defaults are handled](#defaults_how). To modify the configuration
object in the system directly:

```python
from systems.provided.futures_chapter15.basesystem import futures_system
system=futures_system()

## Anything we do with the system may well be cached and will need to be cleared before it sees the new value...


new_idm=1.1 ## new IDM
system.config.instrument_div_multiplier=new_idm

## If we change anything that is nested, we need to change just one element to avoid clearing the defaults:
# So, do this:
system.config.volatility_calculation['days']=20

# Do NOT do this - it will wipe out all the other elements in the volatility_calculation dictionary:
# system.config.volatility_calculation=dict(days=20)


## The config is updated, but to reiterate anything that uses it will need to be cleared from the cache
```

Because we don't create a new system and have to recalculate everything from
scratch, this can be useful for testing isolated changes to the system **if**
you know what you're doing.

### Option 4: Create a `/private/private_config.yaml` file

This makes sense if you want to make a global change to a particular parameter rather than constantly including certain things in your configuration files. Anything in this file will overwrite the system defaults, but will in turn be overwritten by the backtest configuration .yaml file. This file will also come in very handy when it comes to [using pysystemtrade as a production trading environment](/docs/production.md)


### Option 5: Change the project defaults (definitely not recommended)

I don't recommend changing the defaults - a lot of tests will fail for a
start - but should you want to more information is given [here](#defaults).


## Run a backtest on a different set of instruments

Fixed instrument weights: You need to change the instrument weights in the
configuration. Only instruments with weights have positions produced for them.
Estimated instrument weights: You need to change the instruments section of the
configuration.

There are two easy ways to do this - change the config file, or the config
object already in the system (for more on changing config parameters see
['change backtest parameters'](#change_backtest_parameters) ). You also need to
ensure that you have the data you need for any new instruments. See ['use my
own data'](#create_my_own_data) below.


### Change instruments: Change the configuration file

You should make a new config file by copying this
[one](/systems/provided/futures_chapter15/futuresconfig.yaml). Best practice is
to save this as `pysystemtrade/private/this_system_name/config.yaml` (you'll
need to create this directory).

For fixed weights, you can then change this section of the config:

```yaml
instrument_weights:
   - EDOLLAR: 0.117
   - US10: 0.117
   - EUROSTX: 0.20
   - V2X: 0.098
   - MXP: 0.233
   - CORN: 0.233
instrument_div_multiplier: 1.89
```

You may also have to change the forecast_weights, if they're instrument
specific:

```yaml
forecast_weights:
   EDOLLAR:
   - ewmac16_64: 0.21
   - ewmac32_128: 0.08
   - ewmac64_256: 0.21
   - carry: 0.50
```


*At this stage you'd also need to recalculate the diversification multiplier
(see chapter 11 of my book). See [estimating the forecast diversification
multiplier](#divmult).

For estimated instrument weights you'd change this section:

```
instruments: ["EDOLLAR", "US10", "EUROSTX", "V2X", "MXP", "CORN"]
```

Note that if moving from fixed to estimated instrument weights (by changing `system.config.use_instrument_weight_estimates` to `True`), the set of instruments selected in your `system.config.instrument_weights` will be ignored; if you want to continue using this same set of instruments, you need to say so:

```python
system.config.instruments = list(system.config.instrument_weights.keys())
```

(The IDM will be re-estimated automatically)

You may also need to change this section, if you have different rules for each
instrument:

```yaml
rule_variations:
     EDOLLAR: ['ewmac16_64','ewmac32_128', 'ewmac64_256', 'carry']
```

You should then create a new system which points to the new config file:

```python
from sysdata.config.configdata import Config

my_config=Config("private.this_system_name.config.yaml")

from systems.provided.futures_chapter15.basesystem import futures_system
system=futures_system(config=my_config)
```

See [here](#filenames) for how to specify filenames in pysystemtrade.



### Change instruments: Change the configuration object

We can also modify the configuration object in the system directly:

For fixed weights:

```python
from systems.provided.futures_chapter15.basesystem import futures_system
system=futures_system()
new_config=system.config

new_weights=dict(SP500=0.5, KR10=0.5) ## create new weights
new_idm=1.1 ## new IDM

new_config.instrument_weights=new_weights
new_config.instrument_div_multiplier=new_idm

system=futures_system(config=new_config)

```

For estimated weights:

```python
from systems.provided.futures_chapter15.estimatedsystem import futures_system
system=futures_system()
new_config=system.config

new_config.instruments=["SP500", "KR10"]

del(new_config.rule_variations) ## means all instruments will use all trading rules

# this stage is optional if we want to give different instruments different sets of rules
new_config.rule_variations=dict(SP500=['ewmac16_64','carry'], KR10=['ewmac32_128', 'ewmac64_256', 'carry'])

system=futures_system(config=new_config)

```

## Run the backtest only on more recent data

You need to set the start_date in the .yaml backtest configuration file:

```bash
## Note you must use this format
start_date: '2000-01-19'
```


## Run a backtest on all available instruments

If there are is no `instrument_weights` or `instruments` elements in the config, then the backtest will be run over all available instruments in the data. 

## Exclude some instruments from the backtest

Refer to the [instruments document](/docs/instruments.md).

## Exclude some instruments from having positive instrument weights

Refer to the [instruments document](/docs/instruments.md).

<a name="how_do_i_write_rules"> </a>

## Create my own trading rule

At some point you should read the relevant guide section
['rules'](#TradingRules) as there is much more to this subject than I will
explain briefly here.


### Writing the function


A trading rule consists of:

- a function
- some data (specified as positional arguments)
- some optional control arguments (specified as key word arguments)


So the function must be something like these:

```python
def trading_rule_function(data1):
   ## do something with data1

def trading_rule_function(data1, arg1=default_value):
   ## do something with data1
   ## controlled by value of arg1

def trading_rule_function(data1, data2):
   ## do something with data1 and data2

def trading_rule_function(data1, data2, arg1=default_value, arg2=default_value):
   ## do something with data1
   ## controlled by value of arg1 and arg2

```
... and so on.

Functions must return a Tx1 pandas dataframe.

### Adding the trading rule to a configuration

We can either modify the YAML file or the configuration object we've already
loaded into memory. See ['changing backtest
parameters'](#change_backtest_parameters) for more details. If you want to use a
YAML file you need to first save the function into a .py module, so it can be
referenced by a string (we can also use this method for a config object in
memory).

For example the rule imported like this:

```python
from systems.futures.rules import ewmac
```

Can also be referenced like so: `systems.futures.rules.ewmac`

Also note that the list of data for the rule will also be in the form of string
references to methods in the system object. So for example to get the daily
price we'd use the method `system.rawdata.daily_prices(instrument_code)` (for a
list of all the data methods in a system see [stage
methods](#table_system_stage_methods) or type `system.rawdata.methods()` and
`system.rawdata.methods()`). In the trading rule specification this would be
shown as "rawdata.daily_prices".

If no data is included, then the system will default to passing a single data
item - the price of the instrument. Finally if any or all the `other_arg`
keyword arguments are missing then the function will use its own defaults.

At this stage we can also remove any trading rules that we don't want. We also
ought to modify the forecast scalars (See [forecast scale
estimation](#scalar_estimate]), forecast weights and probably the forecast
diversification multiplier (see [estimating the forecast diversification
multiplier](#divmult)). If you're estimating weights and scalars (i.e. in the
pre-baked estimated futures system provided) this will be automatic.

*If you're using fixed values (the default) then if you don't include a
forecast scalar for the rule, it will use a value of 1.0. If you don't include
forecast weights in your config then the system will default to equally
weighting. But if you include forecast weights, but miss out the new rule, then
it won't be used to calculate the combined forecast.*

Here's an example for a new variation of the EWMAC rule. This rule uses two
types of data - the price (stitched for futures), and a precalculated estimate
of volatility.

YAML: (example)
```yaml
trading_rules:
  .... existing rules ...
  new_rule:
     function: systems.futures.rules.ewmac
     data:
         - "rawdata.daily_prices"
         - "rawdata.daily_returns_volatility"
     other_args:
         Lfast: 10
         Lslow: 40
#
#
## Following section is for fixed scalars, weights and div. multiplier:
#
forecast_scalars:
  ..... existing rules ....
  new_rule=10.6
#
forecast_weights:
  .... existing rules ...
  new_rule=0.10
#
forecast_div_multiplier=1.5
#
#
## Alternatively if you're estimating these quantities use this section:
#
use_forecast_weight_estimates: True
use_forecast_scale_estimates: True
use_forecast_div_mult_estimates: True

rule_variations:
     EDOLLAR: ['ewmac16_64','ewmac32_128', 'ewmac64_256', 'new_rule']
#
# OR if all variations are the same for all instruments
#
rule_variations: ['ewmac16_64','ewmac32_128', 'ewmac64_256', 'new_rule']
#
```



Python (example - assuming we already have a config object loaded to modify)

```python

from systems.trading_rules import TradingRule

# method 1
new_rule = TradingRule(
   dict(function="systems.futures.rules.ewmac", data=["rawdata.daily_prices", "rawdata.daily_returns_volatility"],
        other_args=dict(Lfast=10, Lslow=40)))

# method 2 - good for functions created on the fly
from systems.futures.rules import ewmac

new_rule = TradingRule(dict(function=ewmac, data=["rawdata.daily_prices", "rawdata.daily_returns_volatility"],
                            other_args=dict(Lfast=10, Lslow=40)))

## both methods - modify the configuration
config.trading_rules['new_rule'] = new_rule

## If you're using fixed weights and scalars

config.forecast_scalars['new_rule'] = 7.0
config.forecast_weights = dict(...., new_rule=0.10)  ## all existing forecast weights will need to be updated
config.forecast_div_multiplier = 1.5

## If you're using estimates

config.use_forecast_scale_estimates = True
config.use_forecast_weight_estimates = True
use_forecast_div_mult_estimates: True

config.rule_variations = ['ewmac16_64', 'ewmac32_128', 'ewmac64_256', 'new_rule']
# or to specify different variations for different instruments
config.rule_variations = dict(SP500=['ewmac16_64', 'ewmac32_128', 'ewmac64_256', 'new_rule'], US10=['new_rule', ....)
```

Once we've got the new config, by which ever method, we just use it in our
system, eg:

```python
## put into a new system

from systems.provided.futures_chapter15.basesystem import futures_system
system=futures_system(config=config)
```


<a name="create_my_own_data"> </a>

## Use different data or instruments

The default data used for the simulation is .csv files for futures stitched
prices, fx and contract related data. It's my intention to update
this and try to keep it reasonably current with each release. The data is stored in the [data/futures directory](/data/futures/)

You can update that data, if you wish. Be careful to save it as a .csv with the
right formatting, or pandas will complain. Check that a file is correctly
formatted like so:

```python
import pandas as pd
test=pd.read_csv("filename.csv")
test
```
You can also add new files for new instruments. Be sure to keep the file format and header names consistent.

You can create your own directory for .csv files. For example supposed you wanted to get your adjusted prices from
`pysystemtrade/private/system_name/adjusted_price_data`. Here is how you'd use it:

```python
from sysdata.sim.csv_futures_sim_data import csvFuturesSimData
from systems.provided.futures_chapter15.basesystem import futures_system

data=csvFuturesSimData(csv_data_paths=dict(csvFuturesAdjustedPricesData = "private.system_name.adjusted_price_data"))
system=futures_system(data=data)
```
Notice that we use python style "." internal references within a project, we don't give actual path names. See [here](#filenames) for how to specify filenames in pysystemtrade.

The full list of keys that you can use in the `csv_data_paths` are:
* `csvFuturesInstrumentData` (configuration and costs)
* `csvFuturesMultiplePricesData` (prices for current, next and carry contracts)
* `csvFuturesAdjustedPricesData` (stitched back-adjusted prices)
* `csvFxPricesData` (for FX prices)
* `csvRollParametersData` (for roll configuration)
  
Note that you can't put adjusted prices and carry data in the same directory since they use the same file format.

There is more detail about using .csv files [here](#csv).

If you want to store your data in Mongo DB databases instead you need to [use a different data object](#arctic_data).

If you want to get your data from Quandl.com, then see the document [working with futures data](/docs/data.md)

If you want to get data from a different place (eg a database, yahoo finance,
broker, quandl...) you'll need to [create your own Data object](#create_data).

If you want to use a different set of data values (eg equity EP ratios,
interest rates...) you'll need to [create your own Data object](#create_data).

If you want to delve deeper into data storage see the document [working with futures data](/docs/data.md)

## Save my work

To remain organised it's good practice to save any work into a directory like
`pysystemtrade/private/this_system_name/` (you'll need to create the
directory first). If you plan to contribute to github, just be careful to
avoid adding 'private' to your commit ( [you may want to read
this](https://24ways.org/2013/keeping-parts-of-your-codebase-private-on-github/)
).

You can save the contents of a system cache to avoid having to redo
calculations when you come to work on the system again (but you might want to
read about [system caching and pickling](#caching) before you reload them).

```python
from systems.provided.futures_chapter15.basesystem import futures_system

system = futures_system()
system.accounts.portfolio().sharpe() ## does a whole bunch of calculations that will be saved in the cache

system.cache.pickle("private.this_system_name.system.pck") ## use any file extension you like

## In a new session
from systems.provided.futures_chapter15.basesystem import futures_system

system = futures_system()
system.cache.unpickle("private.this_system_name.system.pck")

## this will run much faster and reuse previous calculations
# only complex accounting p&l objects aren't saved in the cache
system.accounts.portfolio().sharpe()

```

You can also save a config object into a yaml file - see [saving
configuration](#save_config).