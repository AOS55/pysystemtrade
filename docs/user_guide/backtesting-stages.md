# Stages

A *stage* within a system does part of the multiple steps of calculation that
are needed to ultimately come up with the optimal positions, and hence the
account curve, for the system. So the backtesting or live trading process
effectively happens within the stage objects.

We define the stages in a system when we create it, by passing a list of stage
objects as the first argument:

```python
from systems.forecasting import Rules
from systems.basesystem import System
data=None ## this won't do anything useful

my_system=System([Rules()], data)
```

(This step is often hidden when we use 'pre-baked' systems)

We can see what stages are in a system just by printing it:

```python
from systems.provided.futures_chapter15.basesystem import futures_system
system=futures_system()
system
```

```
System with stages: accounts, portfolio, positionSize, rawdata, combForecast, forecastScaleCap, rules
```

Stages are attributes of the main system:

```python
from systems.provided.futures_chapter15.basesystem import futures_system
system=futures_system()
system.rawdata
```

```
SystemStage 'rawdata'
```

So we can access the data methods of each stage:

```python
system.rawdata.get_raw_price("EDOLLAR").tail(5)
```

```
              price
2015-04-16  97.9350
2015-04-17  97.9400
2015-04-20  97.9250
2015-04-21  97.9050
2015-04-22  97.8325
```

`system.rawdata.log` provides access to the log for the stage rawdata, and so
on. See [logging](#logging) for more details.



<a name="stage_wiring"> </a>

### Stage 'wiring'

It's worth having a basic understanding of how the stages within a system are
'wired' together. Furthermore if you're going to modify or create new code, or
use [advanced system caching](#caching), you're going to need to understand
this properly.

What actually happens when we call
`system.combForecast.get_combined_forecast("EDOLLAR")` in the pre-baked futures
system? Well this in turn will call other methods in this stage, and they will
call methods in previous stages,.. and so on until we get back to the
underlying data. We can represent this with a diagram:

- `system.combForecast.get_combined_forecast("EDOLLAR")`
  - `system.combForecast.get_forecast_diversification_multiplier("EDOLLAR")`
  - `system.combForecast.get_forecast_weights("EDOLLAR")`
  - `system.combForecast.get_capped_forecast("EDOLLAR", "ewmac2_8"))` etc
    - `system.forecastScaleCap.get_capped_forecast("EDOLLAR", "ewmac2_8"))` etc
      - `system.forecastScaleCap.get_forecast_cap("EDOLLAR", "ewmac2_8")` etc
      - `system.forecastScaleCap.get_scaled_forecast("EDOLLAR", "ewmac2_8")`
        etc
        - `system.forecastScaleCap.get_forecast_scalar("EDOLLAR", "ewmac2_8")`
          etc
        - `system.forecastScaleCap.get_raw_forecast("EDOLLAR", "ewmac2_8")` etc
          - `system.rules.get_raw_forecast("EDOLLAR", "ewmac2_8")` etc
            - `system.data.get_raw_price("EDOLLAR")`
            - `system.rawdata.get_daily_returns_volatility("EDOLLAR")`
              - (further stages to calculate volatility omitted)

A system effectively consists of a 'tree' of which the above shows only a small
part. When we ask for a particular 'leaf' of the tree, the data travels up the
'branches' of the tree, being cached as it goes.

The stage 'wiring' is how the various stages communicate with each other.
Generally a stage will consist of:

1. *Input* methods that get data from another stage without doing any further
   calculation
2. Internal *diagnostic* methods that do intermediate calculations within a
   stage (these may be private, but are usually left exposed so they can be
   used for diagnostic purposes)
3. *Output* methods that other stages will use for their inputs.

For example consider the first few items in the list above. Let's label them
appropriately:

- **Output (combForecast)**:
  `system.combForecast.get_combined_forecast("EDOLLAR")`
  - **Internal (combForecast)**:
    `system.combForecast.get_forecast_diversification_multiplier("EDOLLAR")`
  - **Internal (combForecast)**:
    `system.combForecast.get_forecast_weights("EDOLLAR")`
  - **Input (combForecast)**:
    `system.combForecast.get_capped_forecast("EDOLLAR", "ewmac2_8"))` etc
    - **Output (forecastScaleCap)**:
      `system.forecastScaleCap.get_capped_forecast("EDOLLAR", "ewmac2_8"))` etc

This approach (which you can also think of as the stage "API") is used to make
it easier to modify the code - we can change the way a stage works internally,
or replace it with a new stage with the same name, but as long as we keep the
output method intact we don't need to mess around with any other stage.



### Writing new stages

If you're going to write a new stage (completely new, or to replace an existing
stage) you need to keep the following in mind:

1. New stages should inherit from [`SystemStage`](/systems/stage.py)
2. Modified stages should inherit from the existing stage you're modifying. For
   example if you create a new way of calculating forecast weights then you
   should inherit from [class `ForecastCombine`](/systems/forecast_combine.py),
   and then override the `get_forecast_weights` method; whilst keeping the
   other methods unchanged.
3. Completely new stages will need a unique name; this is specified in the
   object method `_name()`. They can then be accessed with `system.stage_name`
4. Modified stages should use the same name as their parent, or the wiring will
   go haywire.
5. Think about whether you need to protect part of the system cache for this
   stage output [system caching](#caching) from casual deletion.
6. Similarly if you're going to cache complex objects that won't pickle easily
   (like accountCurve objects) you need to put a `not_pickable=True` in the
   decorator call.
7. Use non-cached input methods to get data from other stages. Be wary of
   accessing internal methods in other stages; try to stick to output methods
   only.
8. Use cached input methods to get data from the system data object (since this
   is the first time it will be cached). Again only access public methods of
   the system data object.
9. Use cached methods for internal diagnostic and output methods(see [system
   caching](#caching) ).
10. If you want to store attributes within a stage, then prefix them with _ and
    include a method to access or change them. Otherwise the methods() method
    will return attributes as well as methods.
11. Internal methods should be public if they could be used for diagnostics,
    otherwise prefix them with _ to make them private.
12. The doc string for input and output methods should clearly identify them as
    such. This is to make viewing the wiring easier.
13. The doc string at the head of the stage should specify the input methods
    (and where they take their input from), and the output methods
14. The doc string should also explain what the stage does, and the name of the
    stage
15. Really big stages should be separated across multiple classes (and possibly
    files), using multiple inheritance to glue them together. See the [accounts
    stage](/systems/account.py) for an example.

New stage code should be included in a subdirectory of the systems package (as
for [futures raw data](/systems/futures/) ) or in your [private
directory](/private/).

### Specific stages

The standard list of stages is as follows. The default class is given below, as
well as the system attribute used to access the stage.

1. [Raw data:](#stage_rawdata) [class RawData](/systems/rawdata.py)
   `system.rawdata`
2. [Forecasting:](#rules) [class Rules](/systems/forecasting.py) `system.rules`
   (chapter 7 of my book)
3. [Scale and cap forecasts:](#stage_scale) [class
   ForecastScaleCap](/systems/forecast_scale_cap.py)
   `system.forecastScaleCap`(chapter 7)
4. [Combine forecasts:](#stage_combine) [class
   ForecastCombine](/systems/forecast_combine.py) `system.combForecast`
   (chapter 8)
5. [Calculate subsystem positions:](#position_scale) [class
   PositionSizing](/systems/positionsizing.py) `system.positionSize` (chapters
   9 and 10)
6. [Create a portfolio across multiple instruments:](#stage_portfolio) [class
   Portfolios](/systems/portfolio.py) `system.portfolio` (chapter 11)
7. [Calculate performance:](#accounts_stage) [class
   Account](/systems/account.py) `system.accounts`

Each of these stages is described in more detail below.

<a name="stage_rawdata"> </a>

### Stage: Raw data

The raw data stage is used to pre-process data for calculating trading rules,
scaling positions, or anything else we might need. Good reasons to include something
in raw data are:

1. If it is used multiple times, eg price volatility
2. To provide better diagnostics and visibility in the system, eg the
   intermediate steps required to calculate the carry rule for futures


#### Using the standard [RawData class](/systems/rawdata.py)

The base RawData class includes methods to get instrument prices, daily
returns, volatility, and normalised returns (return over volatility).

As we are trading futures the raw data class has some extra methods needed to calculate the carry
rule for futures, and to expose the intermediate calculations.

(Versions prior to 1.06 had a separate FuturesRawData class)

<a name="vol_calc"> </a>

##### Volatility calculation

There are two types of volatility in my trading systems:

1. Price difference volatility eg sigma (Pt - Pt-1)
2. Percentage return volatility eg sigma (Pt - Pt -1 / P*t-1)

The first kind is used in trading rules to normalise the forecast into
something proportional to Sharpe Ratio. The second kind is used to scale
positions. In both cases we use a 'stitched' price to work out price
differences. So in futures we splice together futures contracts as we roll,
shifting them according to the Panama method. Similarly if the system dealt
with cash equities, it would handle ex-dividend dates in the same way. If we
didn't do this, but just used the 'natural' price (the raw price of the
contract we're trading) to calculate returns, we'd get sharp returns on rolls.

In fact stitched prices are used by default in the system; since they make more
sense for trading rules that usually prefer smoother prices without weird
jumps. Nearly all the methods in raw data that mention price are referring to
the stitched price.

However when working out percentage returns we absolutely don't want to use the
'stitched' price as the denominator. For positive carry assets stitched prices
will increase over time; this means they will be small or even negative in the
past and the percentage returns will be large or have the wrong sign.

For this reason there is a special method in the data class called
`daily_denominator_price`. This tells the code what price to use for the P* in
the calculation above. As we are trading futures this uses the raw price of the
current contract.

The other point to note is that the price difference volatility calculation is
configurable through `config.volatility_calculation`.

The default function used is a robust EWMA volatility calculator with the
following configurable attributes:

- 35 day span
- Needs 10 periods to generate a value
- Will floor any values less than 0.0000000001
- Applies a further vol floor which:
  - Calculates the 5% percentile of vol using a 500 day moving average (needing
    100 periods to generate a value)
  - Floors any vol below that level

YAML:
```yaml
volatility_calculation:
  func: "sysquant.estimators.vol.robust_vol_calc"
  days: 35
  min_periods: 10
  vol_abs_min: 0.0000000001
  vol_floor: True
  floor_min_quant: 0.05
  floor_min_periods: 100
  floor_days: 500

```

If you're considering using your own function please see [configuring defaults
for your own functions](#config_function_defaults)




#### New or modified raw data classes

It would make sense to create new raw data classes for new types of assets, or
to get more visibility inside trading rule calculations.

For example:

1. To work out the quality factor for an equity value system, based on raw
   accounting ratios
2. To work out the moving averages to be used in an EWMAC trading rule, so they
   can be viewed for diagnostic purposes.

For new asset classes in particular you should think hard about what you should
override the `daily_denominator_price` (see discussion on volatility
calculation above).

<a name="rules"> </a>

### Stage: Rules

Trading rules are at the heart of a fully systematic trading system. This stage
description is different from the others; and will be in the form of a tutorial
around creating trading rules.

The base class, Rules() [is here](/systems/forecasting.py); and it shouldn't be
necessary to modify this class.

<a name="TradingRules"> </a>

### Trading rules

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

At a minimum we need to know the function, since other arguments are optional,
and if no data is specified the instrument price is used. A rule specified with
only the function is a 'bare' rule. It should take only one data argument which
is price, and have no other arguments that need new parameter values.

In this project there is a specific [TradingRule
class](/systems/forecasting.py). A `TradingRule` instance contains 3 elements -
a function, a list of any data the function needs, and a dict of any other
arguments that can be passed to the function.

The function can either be the actual function, or a relative reference to it
eg `systems.provided.futures_chapter15.rules.ewmac` (this is useful when a
configuration is created from a file). Data must always be in the form of
references to attributes and methods of the system object, eg
`data.daily_prices` or `rawdata.get_daily_prices`. Either a single data item,
or a list must be passed. Other arguments are in the form a dictionary.

We can create trading rules in a number of different ways. I've noticed that
different people find different ways of defining rules more natural than
others, hence the deliberate flexibility here.

Bare rules which only consist of a function can be defined as follows:

```python

from systems.trading_rules import TradingRule

TradingRule(ewmac)  ## with the actual function
TradingRule("systems.provided.futures_chapter15.rules.ewmac")  ## string reference to the function
```

We can also add data and other arguments. Data is always a list of str, or a
single str. Other arguments are always a dict.

```python
TradingRule(ewmac, data='rawdata.get_daily_prices', other_args=dict(Lfast=2, Lslow=8))
```

Multiple data is fine, and it's okay to omit data or other_args:

```python
TradingRule(some_rule, data=['rawdata.get_daily_prices','data.get_raw_price'])
```

Sometimes it's easier to specify the rule 'en bloc'. You can do this with a 3
tuple. Notice here we're specifying the function with a string, and listing
multiple data items:

```python
TradingRule(("systems.provided.futures_chapter15.rules.ewmac", ['rawdata.get_daily_prices','data.get_raw_price'], dict(Lfast=3, Lslow=12)))
```

You can also specify rules with a dict. If using a dict keywords can be omitted
(but not `function`).

```python
TradingRule(dict(function="systems.provided.futures_chapter15.rules.ewmac", data=['rawdata.get_daily_prices','data.get_raw_price']))
```

Note if you use an 'en bloc' method, and also include the `data` or
`other_args` arguments in your call to `TradingRule`, you'll get a warning.

The dictionary method is used when configuration objects are read from YAML
files; these contain the trading rules in a nested dict.

YAML: (example)
```yaml
trading_rules:
  ewmac2_8:
     function: systems.futures.rules.ewmac
     data:
         - "data.daily_prices"
         - "rawdata.daily_returns_volatility"
     other_args:
         Lfast: 2
         Lslow: 8
     forecast_scalar: 10.6
```

Note that *`forecast_scalar`* isn't strictly part of the trading rule
definition, but if included here will be used instead of the separate
`config.forecast_scalar` parameter (see the [next stage](#stage_scale) ).

#### Data and data arguments

All the items in the `data` list passed to a trading rule are string references to methods in the system object that (usually) take a single argument, the instrument code. In theory these could be anywhere in the system object, but by convention they should only be in `system.rawdata` or `system.data` (if they are in stages that call the rules stage, you will get infinite recursion and things will break), with perhaps occasional reference to `system.get_instrument_list()`. The advantage of using methods in `rawdata` is that these are cached, and can be re-used. It's strongly recommended that you use methods in `rawdata` for trading rules.

What if you want to pass arguments to the data method? For example, you might want to pre-calculate the moving averages of different lengths in `rawdata` and then reuse them to save time. Or you might want to calculate skew over a given time period for all markets and then take a cross sectional average to use in a relative value rule.

We can do this by passing special kinds of `other_args` which are pre-fixed with underscores, eg "_". If an element in the other_ags dictionary has no underscores, then it is passed to the trading rule function as a keyword argument. If it has one leading underscore eg "_argname", then it is passed to the first method in the data list as a keyword argument. If it has two underscores eg "__argname", then it is passed to the second method in the data list, and so on.

Let's see how we could implement the moving average example:

```python
from systems.provided.futures_chapter15.basesystem import *
from systems.trading_rules import TradingRule

data = csvFuturesSimData()
config = Config(
   "systems.provided.futures_chapter15.futuresconfig.yaml")


# First let's add a new method to our rawdata
# As well as the usual instrument_code this has a keyword argument, span, which we are going to access in our trading rule definitions
class newRawData(RawData):
   def moving_average(self, instrument_code, span=8):
      price = self.get_daily_prices(instrument_code)
      return price.ewm(span=span).mean()


# Now for our new trading rule. Multiplier is a trivial variable, included to show you can mix other arguments and data arguments
def new_ewma(fast_ewma, slow_ewma, vol, multiplier=1):
   raw_ewmac = fast_ewma - slow_ewma

   raw_ewmac = raw_ewmac * multiplier

   return raw_ewmac / vol.ffill()


# Now we define our first trading rule. Notice that data gets two kinds of moving average, but the first one will have span 2 and the second span 8
trading_rule1 = TradingRule(dict(function=new_ewma, data=['rawdata.moving_average', 'rawdata.moving_average',
                                                          'rawdata.daily_returns_volatility'],
                                 other_args=dict(_span=2, __span=8, multiplier=1)))

# The second trading rule reuses one of the ewma, but uses a default value for the multiplier and the first ewma span (not great practice, but illustrates what is possible)
trading_rule2 = TradingRule(dict(function=new_ewma, data=['rawdata.moving_average', 'rawdata.moving_average',
                                                          'rawdata.daily_returns_volatility'],
                                 other_args=dict(__span=32)))

rules = Rules(dict(ewmac2_8=trading_rule1, ewmac8_32=trading_rule2))

system = System([
   Account(), Portfolios(), PositionSizing(), newRawData(),
   ForecastCombine(), ForecastScaleCap(), rules
], data, config)

# This will now work in the usual way
system.rules.get_raw_forecast("EDOLLAR", "ewmac8_32")
system.rules.get_raw_forecast("EDOLLAR", "ewmac2_8")
```

Notes: methods passed as data pointers must only have a single argument plus optionally keyword arguments. Multiple non keyword arguments will break. Also in specifying the other arguments you don't have to provide keyword arguments for all data elements, or for the trading rule: all are optional.


### The Rules class, and specifying lists of trading rules

We can pass a trading rule, or a group of rules, into the class Rules() in a
number of ways.

#### Creating lists of rules from a configuration object

Normally we'd pass in the list of rules form a configuration object. Let's have
a look at an incomplete version of the pre-baked chapter 15 futures system.

```python
## We probably need these to get our data

from sysdata.sim.csv_futures_sim_data import csvFuturesSimData
from sysdata.config.configdata import Config
from systems.basesystem import System

## We now import all the stages we need
from systems.forecasting import Rules
from systems.rawdata import RawData

data=csvFuturesSimData()
config=Config("systems.provided.futures_chapter15.futuresconfig.yaml")

rules=Rules()

## build the system
system=System([rules, RawData()], data, config)

rules
```

```
<snip>
Exception: A Rules stage needs to be part of a System to identify trading rules, unless rules are passed when object created
```

```python
## Once part of a system we can see the rules
forecast=system.rules.get_raw_forecast('EDOLLAR','ewmac2_8')
rules
```

```
Rules object with rules ewmac32_128, ewmac64_256, ewmac16_64, ewmac8_32, ewmac4_16, ewmac2_8, carry
```

```python
##
rules.trading_rules()
```

```
{'carry': TradingRule; function: <function carry at 0xb2e0f26c>, data: rawdata.daily_annualised_roll, rawdata.daily_returns_volatility and other_args: smooth_days,
 'ewmac16_64': TradingRule; function: <function ewmac at 0xb2e0f224>, data: rawdata.daily_prices, rawdata.daily_returns_volatility and other_args: Lfast, Lslow,
 'ewmac2_8': TradingRule; function: <function ewmac at 0xb2e0f224>, data: rawdata.daily_prices, rawdata.daily_returns_volatility and other_args: Lfast, Lslow,
 'ewmac32_128': TradingRule; function: <function ewmac at 0xb2e0f224>, data: rawdata.daily_prices, rawdata.daily_returns_volatility and other_args: Lfast, Lslow,
 'ewmac4_16': TradingRule; function: <function ewmac at 0xb2e0f224>, data: rawdata.daily_prices, rawdata.daily_returns_volatility and other_args: Lfast, Lslow,
 'ewmac64_256': TradingRule; function: <function ewmac at 0xb2e0f224>, data: rawdata.daily_prices, rawdata.daily_returns_volatility and other_args: Lfast, Lslow,
 'ewmac8_32': TradingRule; function: <function ewmac at 0xb2e0f224>, data: rawdata.daily_prices, rawdata.daily_returns_volatility and other_args: Lfast, Lslow}
```


What actually happens when we run this? (this is a little complex but worth
understanding).

1. The `Rules` class is created with no arguments.
2. When the `Rules` object is first created it is 'empty' - it doesn't have a list of valid *processed* trading rules.
3. We create the `system` object. This means that all the stages can see the system, in particular they can see the configuration
4. `get_raw_forecast` is called, and looks for the trading rule "ewmac2_8". It gets this by calling the method `get_trading_rules`
5. When the method `get_trading_rules` is called it looks to see if there is a *processed* dict of trading rules
6. The first time the method `get_trading_rules` is called there won't be a processed list. So it looks for something to process
7. First it will look to see if anything was passed when the instance rules of the `Rules()` class was created
8. Since we didn't pass anything instead it processes what it finds in `system.config.trading_rules` - a nested dict, keynames rule variation names.
9. The `Rules` instance now has processed rule names in the form of a dict, keynames rule variation names, each element containing a valid `TradingRule` object



#### Interactively passing a list of trading rules

Often when we're working in development mode we won't have worked up a proper
config. To get round this we can pass a single trading rule or a set of trading
rules to the `Rules()` instance when we create it. If we pass a dict, then the
rules will be given appropriate names, otherwise if a single rule or a list is
passed they will be given arbitrary names "rule0", "rule1", ...

Also note that we don't have pass a single rule, list or dict of rules; we can
also pass anything that can be processed into a trading rule.

```python
## We now import all the stages we need
from systems.forecasting import Rules

## Pass a single rule. Any of the following are fine. See 'Trading rules' for more.
trading_rule=TradingRule(ewmac)
trading_rule=(ewmac, 'rawdata.get_daily_prices', dict(Lfast=2, Lslow=8))
trading_rule=dict(function=ewmac, data='rawdata.get_daily_prices', other_args=dict(Lfast=2, Lslow=8))

rules=Rules(trading_rule)
## The rule will be given an arbitrary name

## Pass a list of rules. Each rule can be defined how you like
trading_rule1=(ewmac, 'rawdata.get_daily_prices', dict(Lfast=2, Lslow=8))
trading_rule2=dict(function=ewmac, other_args=dict(Lfast=4, Lslow=16))

rules=Rules([trading_rule1, tradingrule2])
## The rules will be given arbitrary names

## Pass a dict of rules. Each rule can be defined how you like
trading_rule1=(ewmac, 'rawdata.get_daily_prices', dict(Lfast=2, Lslow=8))
trading_rule2=dict(function=ewmac, other_args=dict(Lfast=4, Lslow=16))

rules=Rules(dict(ewmac2_8=trading_rule1, ewmac4_16=tradingrule2))


```

#### Creating variations on a single trading rule

A very common development pattern is to create a trading rule with some
parameters that can be changed, and then to produce a number of variations. Two
functions are provided to make this easier.

```python

from systems.trading_rules import TradingRule, create_variations_oneparameter, create_variations

## Let's create 3 variations of ewmac
## The default ewmac has Lslow=128
## Let's keep that fixed and vary Lfast
rule = TradingRule("systems.provided.rules.ewmac.ewmac_forecast_with_defaults")
variations = create_variations_oneparameter(rule, [4, 10, 100], "ewmac_Lfast")

variations.keys()
```

```
dict_keys(['ewmac_Lfast_4', 'ewmac_Lfast_10', 'ewmac_Lfast_100'])
```


```python
## Now let's vary both Lslow and Lfast
rule=TradingRule("systems.provided.rules.ewmac.ewmac_forecast_with_defaults")

## each separate rule is specified by a dict. We could use a lambda to produce these automatically
variations=create_variations(rule, [dict(Lfast=2, Lslow=8), dict(Lfast=4, Lslow=16)], key_argname="Lfast")
variations.keys()
   dict_keys(['ewmac_Lfast_4', 'ewmac_Lfast_2'])

variations['Lfast_2'].other_args
   {'Lfast': 4, 'Lslow': 16}

```

We'd now create an instance of `Rules()`, passing variations in as an argument.

#### Using a newly created Rules() instance

Once we have our new rules object we can create a new system with it:

```python
## build the system
system=System([rules, RawData()], data, config)

```

It's generally a good idea to put new fixed forecast scalars (see [forecasting
scaling and capping](#stage_scale) ) and forecast weights into the config (see
[the combining stage](#stage_combine) ); although if you're estimating these
parameters automatically this won't be a problem. Or if you're just playing
with ideas you can live with the default forecast scale of 1.0, and you can
delete the forecast weights so that the system will default to using equal
weights:

```python
del(config.forecast_weights)
```



#### Passing trading rules to a pre-baked system function

If we've got a pre-baked system and a new set of trading rules we want to try
that aren't in a config, we can pass them into the system when it's created:

```python
from systems.provided.futures_chapter15.basesystem import futures_system

## we now create my_rules as we did above, for example
trading_rule1=(ewmac, 'rawdata.get_daily_prices', dict(Lfast=2, Lslow=8))
trading_rule2=dict(function=ewmac, other_args=dict(Lfast=4, Lslow=16))

system=futures_system(trading_rules=dict(ewmac2_8=trading_rule1, ewmac4_16=tradingrule2)) ## we may need to change the configuration
```


#### Changing the trading rules in a system on the fly (advanced)

The workflow above has been to create a `Rules` instance (either empty, or
passing in a set of trading rules), then create a system that uses it. However
sometimes we might want to modify the list of trading rules in the system
object. For example you may have loaded a pre-baked system in (which will have
an empty `Rules()` instance and so be using the rules from the config). Rather
than replace that wholesale, you might want to drop one of the rules, add an
additional one, or change a rule that already exists.

To do this we need to directly access the private `_trading_rules` attribute
that stores **processed** trading rules in a dict. This means we can't pass in
any old rubbish that can be parsed into a trading rule as we did above; we need
to pass in actual `TradingRule` objects.

```python
from systems.provided.futures_chapter15.basesystem import futures_system
from systems.trading_rules import TradingRule

system = futures_system()

## Parse the existing rules in the config (not required if you're going to backtest first as this will call this method doing its normal business)
system.rules.trading_rules()

#############
## add a rule by accessing private attribute
new_rule = TradingRule(
   "systems.provided.futures_chapter15.rules.ewmac")  ## any form of [TradingRule](#TradingRule) is fine here
system.rules._trading_rules['new_rule'] = new_rule
#############


#############
## modify a rule with existing key 'ewmac2_8'
modified_rule = system.rules._trading_rules['ewmac2_8']
modified_rule.other_args['Lfast'] = 10

## We can also do:
## modified_rule.function=new_function
## modified_rule.data='data.get_raw_price'
##

system.rules._trading_rules['ewmac2_8'] = modified_rule
#############


#############
## delete a rule (not recommended)
## Removing the rule from the set of fixed forecast_weights or rule_variations (used for estimating forecast_weights) would have the same effect - and you need to do this anyway
## Rules which aren't in the list of variations or weights are not calculated, so there is no benefit in deleting a rule in terms of speed / space
##
system.rules._trading_rules.pop("ewmac2_8")
#############

```



<a name="stage_scale"> </a>

### Stage: Forecast scale and cap [ForecastScaleCap class](/systems/forecast_scale_cap.py)

This is a simple stage that performs two steps:

1. Scale forecasts so they have the right average absolute value, by multiplying
   raw forecasts by a forecast scalar
2. Cap forecasts at a maximum value


#### Using fixed weights (/systems/forecast_scale_cap.py)

The standard configuration uses fixed scaling and caps. It is included in
[standard futures system](#futures_system).

Forecast scalars are specific to each rule. Scalars can either be included in
the `trading_rules` or `forecast_scalars` part of the config. The former takes
precedence if both are included:

YAML: (example)
```yaml
trading_rules:
  some_rule_name:
     function: systems.futures.rules.arbitrary_function
     forecast_scalar: 10.6

```

YAML: (example)
```yaml
forecast_scalars:
   some_rule_name: 10.6
```


The forecast cap is also configurable, but must be the same for all rules:

YAML:
```yaml
forecast_cap: 20.0
```

If entirely missing default values of 1.0 and 20.0 are used for the scale and
cap respectively.

<a name="scalar_estimate"> </a>

#### Calculating estimated forecasting scaling on the fly(/systems/forecast_scale_cap.py)

See [this blog
post](https://qoppac.blogspot.com/2016/01/pysystemtrader-estimated-forecast.html).

You may prefer to estimate your forecast scales from the available data. This
is often necessary if you have a new trading rule and have no idea at all what
the scaling should be. To do this you need to turn on estimation
`config.use_forecast_scale_estimates=True`. It is included in the pre-baked
[estimated futures system](#futures_system).

All the config parameters needed are stored in
`config.forecast_scalar_estimate`.

You can either estimate scalars for individual instruments, or using data
pooled across instruments. The config parameter `pool_instruments` determines
which option is used.

##### Pooled forecast scale estimate (default)

We do this if `pool_instruments=True`. This defaults to using the function
"sysquant.estimators.forecast_scalar.forecast_scalar", but this is configurable using the parameter
`func`. If you're changing this please see [configuring defaults for your own
functions](#config_function_defaults).

I strongly recommend using pooling, since it's always good to get more data.
The only reason not to use it is if you've been an idiot and designed a
forecast for which the scale is naturally different across instruments (see
chapter 7 of my book).

This function calculates a cross sectional median of absolute values, then
works out the scalar to get it 10, using a rolling moving average (so always
out of sample).

I also recommend using the defaults, a `window` size of 250000 (essentially
long enough so you're estimating with an expanding window) and `min_periods` of
500 (a couple of years of daily data; less than this and we'll get something
too unstable especially for a slower trading rule, more and we'll have to wait
too long to get a value). The other parameter of interest is `backfill` which
is boolean and defaults to True. This backfills the first scalar value found
back into the past so we don't lose any data; strictly speaking this is
cheating but we're not selecting the parameter for performance reasons so I for
one can sleep at night.

Note: The pooled estimate is [cached](#caching) as an 'across system',
non instrument specific, item.

##### Individual instrument forecast scale estimate

We do this if `pool_instruments=False`. Other parameters work in the same way.

Note: The estimate is [cached](#caching) separately for each instrument.


<a name="stage_combine"> </a>

### Stage: Forecast combine [ForecastCombine class](/systems/forecast_combine.py)

We now take a weighted average of forecasts using instrument weights, and
multiply by the forecast diversification multiplier.


#### Using fixed weights and multipliers(/systems/forecast_combine.py)

The default fixed weights and a fixed multiplier. It is included in the
pre-baked [standard futures system](#futures_system).

Both weights and multiplier are configurable.

Forecast weights can be (a) common across instruments, or (b) specified
differently for each instrument. If not included equal weights will be used.

YAML: (a)
```yaml
forecast_weights:
     ewmac: 0.50
     carry: 0.50

```

YAML: (b)
```yaml
forecast_weights:
     SP500:
      ewmac: 0.50
      carry: 0.50
     US10:
      ewmac: 0.10
      carry: 0.90

```

The diversification multiplier can also be (a) common across instruments, or
(b) we use a different one for each instrument (would be normal if instrument
weights were also different).


YAML: (a)
```yaml
forecast_div_multiplier: 1.0

```

YAML: (b)
```yaml
forecast_div_multiplier:
     SP500: 1.4
     US10:  1.1
```

Note that the `get_combined_forecast` method in the standard fixed base class
automatically adjusts forecast weights if different trading rules have
different start dates for their forecasts. It does not adjust the multiplier.
This means that in the past the multiplier will probably be too high.

#### Using estimated weights and diversification multiplier(/systems/forecast_combine.py)


This behaviour is included in the pre-baked [estimated futures
system](#futures_system). We switch to it by setting
`config.use_forecast_weight_estimates=True` and/or
`config.use_forecast_div_mult_estimates=True`.

##### Estimating the forecast weights

See [optimisation](#optimisation) for more information.


##### Removing expensive trading rules

See [optimisation](#optimisation) for more information.

##### Estimating the forecast diversification multiplier

See [estimating diversification multipliers](#divmult).

#### Forecast mapping

A new optional feature introduced in version 0.18.2 is *forecast mapping*. This is the non linear mapping discussed in [this blog post](https://qoppac.blogspot.com/2016/03/diversification-and-small-account-size.html) whereby we do not take a forecast until it has reached some threshold. Because this will reduce the standard deviation of our forecasts we compensate by ramping up the forecast more quickly until the raw forecast reaches the existing cap (which defaults to 20). This is probably illustrated better if we look at the non-linear mapping function:

```python
#This is syscore.algos.map_forecast_value
def map_forecast_value_scalar(x, threshold, capped_value, a_param, b_param):
    """
    Non linear mapping of x value; replaces forecast capping; with defaults will map 1 for 1

    We want to end up with a function like this, for raw forecast x and mapped forecast m,
        capped_value c and threshold_value t:

    if -t < x < +t: m=0
    if abs(x)>c: m=sign(x)*c*a
    if c < x < -t:   (x+t)*b
    if t < x < +c:   (x-t)*b

    :param x: value to map
    :param threshold: value below which map to zero
    :param capped_value: maximum value we want x to take (without non linear mapping)
    :param a_param: multiple at capped value
    :param b_param: slope
    :return: mapped x
    """
    x = float(x)
    if np.isnan(x):
        return x
    if abs(x)<threshold:
        return 0.0
    if x >= -capped_value and x <= -threshold:
        return b_param*(x+threshold)
    if x >= threshold and x <= capped_value:
        return b_param*(x-threshold)
    if abs(x)>capped_value:
        return sign(x)*capped_value*a_param

    raise Exception("This should cover all conditions!")

```

What values should we use for a,b and threshold (t)? We want to satisfy the following rules:

- a_param should be set so that we typically hold four contracts when the raw_forecast is at capped_value (see chapter 12 of my book, "Systematic Trading")
- We require b = (c*a)/(c-t)
- Given our parameters, and the distribution of the raw forecast (assumed to be Gaussian), the average absolute value of the final distribution should be unchanged.

These values aren't estimated by default, so you can use this external function:

```python
# Assuming futures_system already contains a system which has positions
from systems.diagoutput import systemDiag

sysdiag = systemDiag(futures_system)
sysdiag.forecast_mapping()
```

Parameters are specified by market as follows:

YAML:
```yaml
forecast_mapping:
  AEX:
    a_param: 2.0
    b_param: 8.0
    threshold: 15.0
```

If the forecast_mapping key is missing from the configuration object, or the instrument is missing from the dict, then no mapping will be done (the raw forecast will remain unchanged). Also note that if a_param = b_param = 1, and threshold=0, then this is equivalent to no mapping.


### Stage: Position scaling

<a name="notional"> We now scale our positions according to our percentage
volatility target (chapters 9 and 10 of my book). At this stage we treat our
target, and therefore our account size, as fixed. So we ignore any compounding
of losses and profits. It's for this reason the I refer to the 'notional'
position. Later in the documentation I'll relax that assumption. </a>

#### Using the standard [PositionSizing class](/systems/positionsizing.py)

The annualised percentage volatility target, notional trading capital and
currency of trading capital are all configurable.

YAML:
```yaml
percentage_vol_target: 16.0
notional_trading_capital: 1000000
base_currency: "USD"
```

Note that the stage code tries to get the percentage volatility of an
instrument from the rawdata stage. Since a rawdata stage might be omitted, it
can also fall back to calculating this from scratch using the data object and
the default volatility calculation method.



<a name="stage_portfolio"> </a>

### Stage: Creating portfolios [Portfolios class](/systems/portfolio.py)

The instrument weights and instrument diversification multiplier are used to
combine different instruments together into the final portfolio (chapter eleven
of my book).


#### Using fixed weights and instrument diversification multiplier(/systems/portfolio.py)

The default uses fixed weights and multiplier.

Both are configurable. If omitted equal weights will be used, and a multiplier
of 1.0

YAML:
```yaml
instrument_weights:
    EDOLLAR: 0.5
    US10: 0.5
instrument_div_multiplier: 1.2

```

Note that the `get_instrument_weights` method in the standard fixed base class
automatically adjusts raw forecast weights if different instruments have
different start dates for their price history and forecasts. It does not adjust
the multiplier. This means that in the past the multiplier will probably be too
high.



#### Using estimated weights and instrument diversification multiplier(/systems/portfolio.py)

You can estimate the correct instrument diversification multiplier 'on the
fly', and also estimate instrument weights. This functionality is included in
the pre-baked [estimated futures system](#futures_system). It is accessed by
setting `config.use_instrument_weight_estimates=True` and/or
`config.use_instrument_div_mult_estimates=True`.

##### Estimating the instrument weights

See [optimisation](#optimisation) for more information.

##### Estimating the forecast diversification multiplier

See [estimating diversification multipliers](#divmult).

<a name="buffer"> </a>

#### Buffering and position inertia

Position inertia, or buffering, is a way of reducing trading costs. The idea is
that we avoid trading if our optimal position changes only slightly by applying
a 'no trade' buffer around the current position. There is more on this subject
in chapter 11 of my book.

There are two methods that I use. *Position* buffering is the same as the
position inertia method used in my book. We compare the current position to the
optimal position. If it's not within 10% (the 'buffer') then we trade to the
optimal position, otherwise we don't bother.

This configuration will implement position inertia as in my book.

YAML:
```yaml
buffer_trade_to_edge: False
buffer_method: position
buffer_size: 0.10
```

The second method is *forecast* buffering. Here we take a proportion of the
average absolute position (what we get with a forecast of 10), and use that to
size the buffer width. This is more theoretically correct; since the buffer
doesn't shrink as we get to zero. Secondly if outside the buffer we trade to
the nearest edge of the buffer, rather than going to the optimal position. This
further reduces transaction costs. Here are my recommended settings for
forecast buffering:

YAML:
```yaml
buffer_trade_to_edge: True
buffer_method: forecast
buffer_size: 0.10
```

Note that buffering can work on both rounded and unrounded positions. In the
case of rounded positions we round the lower limit of the buffer, and the upper
limit.

These python methods allow you to see buffering in action.

```python
system.portfolio.get_notional_position("US10") ## get the position before buffering
system.portfolio.get_buffers_for_position("US10") ## get the upper and lower edges of the buffer
system.accounts.get_buffered_position("US10", roundpositions=True) ## get the buffered position.
```

Note that in a live trading system buffering is done downstream of the
system module, in a process which can also see the actual current positions we
hold [the strategy order generation)](/docs/production.md).

Finally, if you set buffer_method to none there will be no buffering.

#### Capital correction

If you want to see positions that reflect varying capital, then read the
section on [capital correction](#capcorrection).


<a name="accounts_stage"> </a>

### Stage: Accounting

The final stage is the all important accounting stage, which calculates p&l.

<a name="standard_accounts_stage"> </a>

#### Using the standard [Account class](/systems/accounts/accounts_stage.py)

The standard accounting class includes several useful methods:

- `portfolio`: works out the p&l for the whole system (returns accountCurveGroup)
- `pandl_for_instrument`: the contribution of a particular instrument to the
  p&l (returns accountCurve)
- `pandl_for_subsystem`: work out how an instrument has done in isolation
  (returns accountCurve)
- `pandl_across_subsystems`: group together all subsystem p&l (not the same as
  portfolio! Instrument weights aren't used) (returns accountCurveGroup)
- `pandl_for_trading_rule`: how a trading rule has done aggregated over all
  instruments (returns accountCurveGroup)
- `pandl_for_trading_rule_weighted`: how a trading rule has done over all
  instruments as a proportion of total capital (returns accountCurveGroup)
- `pandl_for_trading_rule_unweighted`: how a trading rule has done over all
  instruments, unweighted (returns accountCurveGroup)
- `pandl_for_all_trading_rules`: how all trading rules have done over all
  instruments (returns nested accountCurveGroup)
- `pandl_for_all_trading_rules_unweighted`: how all trading rules have done
  over all instruments, unweighted (returns nested accountCurveGroup)
- `pandl_for_instrument_rules`: how all trading rules have done for a
  particular instrument (returns accountCurveGroup)
- `pandl_for_instrument_rules_unweighted`: how all trading rules have done for
  one instrument, unweighted (returns accountCurveGroup)
- `pandl_for_instrument_forecast`: work out how well a particular trading rule
  variation has done with a particular instrument (returns accountCurve)
- `pandl_for_instrument_forecast_weighted`: work out how well a particular
  trading rule variation has done with a particular instrument as a proportion
  of total capital (returns accountCurve)


(Note that [buffered](#buffer) positions are only used at the final portfolio
stage; the positions for forecasts and subsystems are not buffered. So their
trading costs may be a little overstated).

(Warning: see [weighted and unweighted account curve groups](#weighted_acg) )

Most of these classes share some useful arguments (all boolean):

- `delayfill`: Assume we trade at the next days closing price. Always defaults
  to True (more conservative)
- `roundpositions`: Round positions to nearest instrument block. Defaults to
  True for portfolios and instruments, defaults to False for subsystems. Not
  used in `pandl_for_instrument_forecast` or `pandl_for_trading_rule` (always
  False)

All p&l methods return an object of type `accountCurve` (for instruments,
subsystems and instrument forecasts) or `accountCurveGroup` (for portfolio and
trading rule), or even nested `accountCurveGroup`
(`pandl_for_all_trading_rules`, `pandl_for_all_trading_rules_unweighted`). This
inherits from a pandas data frame, so it can be plotted, averaged and so on. It
also has some special methods. To see what they are use the `stats` method:

```python
from systems.provided.futures_chapter15.basesystem import futures_system
system=futures_system()
system.accounts.portfolio().stats()
```

```
[[('min', '-1.997e+05'),
  ('max', '4.083e+04'),
  ('median', '-1.631'),
  ('mean', '156.9'),
  ('std', '5226'),
  ('skew', '-7.054'),
  ('ann_mean', '4.016e+04'),
  ('ann_std', '8.361e+04'),
  ('sharpe', '0.4803'),
  ('sortino', '0.5193'),
  ('avg_drawdown', '-1.017e+05'),
  ('time_in_drawdown', '0.9621'),
  ('calmar', '0.1199'),
  ('avg_return_to_drawdown', '0.395'),
  ('avg_loss', '-3016'),
  ('avg_gain', '3371'),
  ('gaintolossratio', '1.118'),
  ('profitfactor', '1.103'),
  ('hitrate', '0.4968'),
  ('t_stat', '2.852'),
  ('p_value', '0.004349')],
 ('You can also plot / print:',
  ['rolling_ann_std', 'drawdown', 'curve', 'percent'])]
```

The `stats` method lists three kinds of output:

1. Statistics which can also be extracted with their own methods eg to extract
   sortino use `system.accounts.portfolio().sortino()`
2. Methods which can be used to do interesting plots eg
   `system.accounts.portfolio().drawdown()`
3. Attributes which can be used to get returns over different periods, eg
   `systems.accounts.portfolio().annual`


#### `accountCurve`

There is a lot more to `accountCurve` and group objects than meets the eye.

Let's start with `accountCurve`, which is the output you get from
`systems.account.pandl_for_subsystem` amongst other things

```python
acc_curve=system.accounts.pandl_for_subsystem("EDOLLAR")
```

This *looks* like a pandas data frame, where each day is a different return.
However it's actually a bit more interesting than that. There's actually three
different account curves buried inside this; *gross* p&l without costs,
*costs*, and *net* including costs. We can access any of them like so:

```python
acc_curve.gross
acc_curve.net
acc_curve.costs
acc_curve.to_ncg_frame() ## this method returns a data frame with all 3 elements as columns
```

The *net* version is identical to acc_curve; this is deliberate to encourage
you to look at net returns. Each curve defaults to displaying daily returns,
however we can also access different frequencies (daily, weekly, monthly,
annual):

```python
acc_curve.gross.daily ## equivalent to acc_curve.gross
acc_curve.net.daily ## equivalent to acc_curve and acc_curve.net
acc_curve.net.weekly ## or also acc_curve.weekly
acc_curve.costs.monthly
```

Once you have the frequency you require you can then use any of the statistical
methods:

```python
acc_curve.gross.daily.stats() ## Get a list of methods. equivalent to acc_curve.gross.stats()
acc_curve.annual.sharpe() ## Sharpe ratio based on annual
acc_curve.gross.weekly.std() ## standard deviation of weekly returns
acc_curve.daily.ann_std() ## annualised std. deviation of daily (net) returns
acc_curve.costs.annual.median() ## median of annual costs

```

... or other interesting methods:

```python
import syscore.pandas.strategy_functions

acc_curve.rolling_ann_std()  ## rolling annual standard deviation of daily (net) returns
acc_curve.gross.curve()  ## cumulated returns = account curve of gross daily returns
syscore.pandas.strategy_functions.drawdown()  ## drawdown of monthly net returns
acc_curve.costs.weekly.curve()  ## cumulated weekly costs
```

Personally I prefer looking at statistics in percentage terms. This is easy.
Just use the .percent property before you use any statistical method:

```python
import syscore.pandas.strategy_functions

acc_curve.capital  ## tells me the capital I will use to calculate %
acc_curve.percent
acc_curve.gross.daily.percent
acc_curve.net.daily.percent
acc_curve.costs.monthly.percent
acc_curve.gross.daily.percent.stats()
acc_curve.monthly.percent.sharpe()
acc_curve.gross.weekly.percent.std()
acc_curve.daily.percent.ann_std()
acc_curve.costs.annual.percent.median()
acc_curve.percent.rolling_ann_std()
acc_curve.gross.percent.curve()
syscore.pandas.strategy_functions.drawdown()
acc_curve.costs.weekly.percent.curve()
```

Incidentally you can 'daisy-chain' the percentage, frequency, and gross/net/costs operators in any order; the underlying object isn't actually changed, it's just the representation of it that is modified. If you want to reverse a percentage operator you can use .value_terms().


#### `accountCurveGroup` in more detail

`accountCurveGroup`, is the output you get from `systems.account.portfolio`,
`systems.account.pandl_across_subsystems`,
`pandl_for_instrument_rules_unweighted`, `pandl_for_trading_rule` and
`pandl_for_trading_rule_unweighted`. For example:

```python
acc_curve_group=system.accounts.portfolio()
```

Again this *looks* like a pandas data frame, or indeed like an ordinary account
curve object. So for example these all work:

```python
acc_curve_group.gross.daily.stats() ## Get a list of methods. equivalent to acc_curve.gross.stats()
acc_curve_group.annual.sharpe() ## Sharpe ratio based on annual
acc_curve_group.gross.weekly.std() ## standard deviation of weekly returns
acc_curve_group.daily.ann_std() ## annualised std. deviation of daily (net) returns
acc_curve_group.costs.annual.median() ## median of annual costs

```

These are in fact all giving the p&l for the entire portfolio (the sum of
individual account curves across all assets); defaulting to giving the net,
daily curve. To find out which assets we use acc_curve_group.asset_columns; to
access a particular asset we use `acc_curve_group['assetName']`.

```python
acc_curve_group.asset_columns
acc_curve_group['US10']
```

*Warning see [weighted and unweighted account curve groups](#weighted_acg)*

The second command returns the account curves *just* for US 10 year bonds. So
we can do things like:

```python
acc_curve_group['US10'].gross.daily.stats() ## Get a list of methods. equivalent to acc_curve.gross.stats()
acc_curve_group['US10'].annual.sharpe() ## Sharpe ratio based on annual
acc_curve_group['US10'].gross.weekly.std() ## standard deviation of weekly returns
acc_curve_group['US10'].daily.ann_std() ## annualised std. deviation of daily (net) returns
acc_curve_group['US10'].costs.annual.median() ## median of annual costs

acc_curve_group.gross['US10'].weekly.std() ## notice equivalent way of getting account curves
```

Sometimes it's nicer to plot all the individual account curves, so we can get a
data frame.


```python
acc_curve_group.to_frame() ## returns net account curves all assets in a frame
acc_curve_group.net.to_frame() ## returns net account curves all assets in a frame
acc_curve_group.gross.to_frame() ## returns net account curves all assets in a frame
acc_curve_group.costs.to_frame() ## returns net account curves all assets in a frame
```
*Warning see [weighted and unweighted account curve groups](#weighted_acg)*


The other thing you can do is get a dictionary of any statistical method,
measured across all assets:

```python
acc_curve_group.get_stats("sharpe", "net", "daily") ## get all annualised sharpe ratios using daily data
acc_curve_group.get_stats("sharpe", freq="daily") ## equivalent
acc_curve_group.get_stats("sharpe", curve_type="net") ## equivalent
acc_curve_group.net.get_stats("sharpe", freq="daily") ## equivalent
acc_curve_group.net.get_stats("sharpe", percent=False) ## defaults to giving stats in % terms, this turns it off

```

*Warning see [weighted and unweighted account curve groups](#weighted_acg)*

You can get summary statistics for these. These can either be simple averages
across all assets, or time weighted by the amount of data each asset has.

```python
acc_curve_group.get_stats("sharpe").mean() ## get simple average of annualised sharpe ratios for net returns using daily data
acc_curve_group.get_stats("sharpe").std(timeweighted=True) ## get time weighted standard deviation of sharpes across assets,
acc_curve_group.get_stats("sharpe").tstat(timeweighted=False) ## t tstatistic for average sharpe ratio
acc_curve_group.get_stats("sharpe").pvalue(timeweighted=True) ## p value of t statistic of time weighted average sharpe ratio.

```



#### A nested `accountCurveGroup`

A nested `accountCurveGroup`, is the output you get from
`pandl_for_all_trading_rules` and `pandl_for_all_trading_rules_unweighted`. For
example:

```python
nested_acc_curve_group=system.accounts.pandl_for_all_trading_rules()
```

This is an account curve group, whose elements are the performance of each
trading rule eg this kind of thing works:

```python
ewmac64_acc=system.accounts.pandl_for_all_trading_rules()['ewmac64_256']
```

However this is also an accountCurveGroup! So you can, for example display how
each instrument within this trading rule contributed to performance as a data
frame:

```python
ewmac64_acc.to_frame()
```


<a name="weighted_acg"> </a>

##### Weighted and unweighted account curve groups

There are two types of account curve; weighted and unweighted. Weighted curves
include returns for each instrument (or trading rule) as a proportion of the
total capital at risk. Unweighted curves show each instrument or trading rule
in isolation.

Weighted:
- `portfolio`: works out the p&l for the whole system (weighted group -
  elements are `pandl_for_instrument` - effective weights are instrument
  weights * IDM)
- `pandl_for_instrument`: the contribution of a particular instrument to the
p&l (weighted individual curve for one instrument - effective weight is
instrument weight * IDM) -`pandl_for_instrument_rules`: how all trading rules
have done for a particular instrument (weighted group - elements are
`pandl_for_instrument_forecast` across trading rules; effective weights are
forecast weights * FDM)
- `pandl_for_instrument_forecast_weighted`: work out how well a particular
  trading rule variation has done with a particular instrument as a proportion
  of total capital (weighted individual curve - weights are forecast weight *
  FDM * instrument weight * IDM)
- `pandl_for_trading_rule_weighted`: how a trading rule has done over all
  instruments as a proportion of total capital (weighted group -elements are
  `pandl_for_instrument_forecast_weighted` across instruments - effective
  weights are risk contribution of instrument to trading rule)
- `pandl_for_all_trading_rules`: how all trading rules have done over all
  instruments (weighted group -elements are `pandl_for_trading_rule_weighted`
  across variations - effective weight is risk contribution of each trading
  rule)

Partially weighted (see below):
- `pandl_for_trading_rule`: how a trading rule has done over all instruments
  (weighted group -elements are `pandl_for_instrument_forecast_weighted` across
  instruments, weights are risk contribution of each instrument to trading
  rule)

Unweighted:
- `pandl_across_subsystems`: works out the p&l for all subsystems (unweighted
  group - elements are `pandl_for_subsystem`)
- `pandl_for_subsystem`: work out how an instrument has done in isolation
  (unweighted individual curve for one instrument)
- `pandl_for_instrument_forecast`: work out how well a particular trading rule
  variation has done with a particular instrument (unweighted individual curve)
- `pandl_for_instrument_rules_unweighted`: how all trading rules have done for
  a particular instrument (unweighted group - elements are
  `pandl_for_instrument_forecast` across trading rules)
- `pandl_for_trading_rule_unweighted`: how a trading rule has done over all
instruments (unweighted group -elements are `pandl_for_instrument_forecast`
across instruments)
- `pandl_for_all_trading_rules_unweighted`: how all trading rules have done
  over all instruments (unweighted group -elements are `pandl_for_trading_rule`
  across instruments - effective weight is risk contribution of each trading
  rule)


Note that `pandl_across_subsystems` / `pandl_for_subsystem` are effectively the
unweighted versions of `portfolio` / `pandl_for_instrument`.

The difference is important for a few reasons.

- Firstly the return and risk of individual weighted curves will be lower than
  the target
- The returns of individual weighted curves will also be highly non stationary,
  at least for instruments. This is because the weightings of instruments
  within a portfolio, or a trading rule, will change over time. Usually there
  are fewer instruments. This means that the risk profile will show much higher
  returns earlier in the series. Statistics such as sharpe ratio may be highly
  misleading.
- The portfolio level aggregate returns of unweighted group curves will make no
  sense. They will be equally weighted, whereas we'd normally have different
  weights.
- Also for portfolios of unweighted groups risk will usually fall over time as
  markets are added and diversification effects appear. Again this is more
  problematic for groups of instruments (within a portfolio, or within a
  trading rule)

Weighting for trading rules p&l is a *little* complicated.

*`pandl_for_instrument_forecast`:* If I want the p&l of a single trading rule
for one instrument in isolation, then I use `pandl_for_instrument_forecast`.

*`pandl_for_trading_rule_unweighted`*: If I aggregate these across instruments
then I get `pandl_for_trading_rule_unweighted`. The individual unweighted
curves are instrument p&l for each instrument and forecast.

*`pandl_for_instrument_forecast_weighted`:* The weighted p&l of a single
trading rule for one instrument, as a proportion of the *entire system's
capital*, will be it's individual p&l in isolation
(`pandl_for_instrument_forecast`) multiplied by the product of the instrument
and forecast weights, and the IDM and FDM (this ignores the effect of total
forecast capping and position buffering or inertia).

*`pandl_for_trading_rule_weighted`:* The weighted p&l of a single trading rule
across individual instruments, as a proportion of the *entire system's
capital*, will be the group of `pandl_for_instrument_forecast_weighted` of
these for a given rule. You can get this with
`pandl_for_trading_rule_weighted`. The individual curves within this will be
instrument p&l for the relevant trading rule, effectively weighted by the
product of instrument, forecast weights, FDM and IDM. The risk of the total
curve will be equal to the risk of the rule as part of the total capital, so
will be lower than you'd expect.

*`pandl_for_all_trading_rules`:* If I group the resulting curves across trading
rules, then I get `pandl_for_all_trading_rules`. The individual curves will be
individual trading rules, weighted by their contribution to total risk. The
total curve is the entire system; it will look close to but not exactly like a
`portfolio` account curve because of the non linear effects of combined
forecast capping, and position buffering or inertia, and rounding if that's
used for the portfolio curve.

*`pandl_for_trading_rule`:* If I want the performance of a given trading rule
across individual instruments in isolation, then I need to take
`pandl_for_trading_rule_weighted` and normalise it so that the returns are as a
proportion of the sum of all the relevant forecast weight * FDM * instrument
weight * IDM; this is equivalent to the rules risk contribution within the
system. . This is an unweighted curve in one sense (it's not a proportion of
total capital), but it's weighted in another (the individual curves when added
up give the group curve). The total account curve will have the same target
risk as the entire system. The individual curves within it are for each
instrument, weighted by their contribution to risk.

*`pandl_for_all_trading_rules_unweighted`:* If I group *these* curves together,
then I get `pandl_for_all_trading_rules_unweighted`. The individual curves will
be individual trading rules but not weighted; so each will have its own risk
target. This is an unweighted group in the truest sense; the total curve won't
make sense.


To summarise:

- Individual account curves either in, or outside, a weighted group should be
  treated with caution. But the entire portfolio curve is fine.
- The portfolio level account curve for an unweighted group should be treated
  with caution. But the individual curves are fine.
- With the exception of `pandl_for_trading_rule` the portfolio level curve for
  a weighted group is a proportion of the entire system capital.

The attribute `weighted` is set to either True (for weighted curves
including `pandl_for_trading_rule`) or False (otherwise). All curve __repr__
methods also show either weighted or unweighted status.

#### Testing account curves

If you want to know how significant the returns for an account curve are (no
matter where you got it from), then use the method `accurve.t_test()`. This
returns the two sided t-test statistic and p-value for a null hypothesis of a
zero mean.

Sometimes you might want to compare the performance of two systems, instruments
or trading rules. The function `from syscore.accounting import account_t_test`
can be used for this purpose. The two parameters can be anything that looks
like an account curve, no matter where you got it from.

When run it returns a two sided t-test statistic and p-value for the null
hypothesis of identical means. This is done on the period of time that both
objects are trading.

Warning: The assumptions underlying a t-test may be violated for financial
data. Use with care.

<a name="costs"> </a>

#### Costs

I work out costs in two different ways:

- by applying a constant drag calculated according to the standardised cost in
  Sharpe ratio terms and the estimated turnover (see chapter 12 of my book)
- using the actual costs for each trade. 

The former method is always used for costs derived from forecasts
(`pandl_for_instrument_forecast`, `pandl_for_instrument_forecast_weighted`,
`pandl_for_trading_rule`, `pandl_for_all_trading_rules`,
`pandl_for_all_trading_rules_unweighted`, `pandl_for_trading_rule_unweighted`,
`pandl_for_trading_rule_weighted`, `pandl_for_instrument_rules_unweighted`, and
`pandl_for_instrument_rules`).

For costs derived from actual positions (everything else) we can use either method. Actual cash costs are more accurate especially if your system has sparse positions (eg the dynamic optimised system I describe elsewhere). However it's quicker to use SR costs, so if you set `use_SR_costs=True` you will speed things up with some loss of accuracy.

Both cost methods now account for holding - rollover costs.

Note that 'actual costs' are normally standardised for historic volatility (although you can optionally turn this off in config `vol_normalise_currency_costs=False` which is useful for comparing with live trading purposes, but I do not recommend it for historical purposes as I don't think it is accurate in the past)

Costs that can be included are:

- Slippage, in price points. Half the bid-ask spread, unless trading in large
  size or with a long history of trading at a better cost.
- Cost per instrument block, in local currency. This is used for most futures.
- Percentage of value costs (0.01 is 1%). Used for US equities.
- Per trade costs, in local currency. Common for UK brokers. This won't be
  applied correctly unless `roundpositions=True` in the accounts call.

To see the turnover that has been estimated use:

```
system.accounts.turnover_at_portfolio_level() ## Total portfolio turnover
system.accounts.subsystem_turnover(instrument_code) ### Annualised turnover of subsystem
system.accounts.instrument_turnover(instrument_code) ### Annualised turnover of portfolio level position
system.accounts.forecast_turnover(instrument_code, rule_variation_name) ## Annualised turnover of forecast
```

Instrument level turnovers are accurate for the vanilla system but may be misleading for systems with sparse positions (eg the dynamic optimised system I describe elsewhere) because the notion of 'average position' is difficult to quantify. 

To see holding costs in SR units:


```
system.accounts.get_rolls_per_year("EDOLLAR") ## four
system.accounts.get_SR_cost_per_trade_for_instrument("EDOLLAR") ## about 1 SR unit
system.accounts.get_SR_holding_cost_only("EDOLLAR") ## cost of 4 rolls per year: which is two 'turnovers'
system.accounts.get_SR_trading_cost_only_given_turnover("EDOLLAR", 5.0) ## trading five times a year, no holding cost
system.accounts.get_SR_cost_given_turnover("EDOLLAR", 5) ## includes both holding and trading costs for a turnover of 5
system.accounts.get_SR_cost_for_instrument_forecast("EDOLLAR", "carry") ## includes both
system.accounts.pandl_for_subsystem("EDOLLAR") ## includes both, assuming you're using SR costs
```



For calculating forecast costs (`pandl_for_instrument_forecast`... and so on.
Note these are used for estimating forecast weights) I offer the option to pool
costs across instruments. You can either pool the estimate of turnovers (which
I recommend), or pool the average of cost * turnover (which I don't recommend).
Averaging in the pooling process is always done with more weight given to
instruments that have more history.

```
forecast_cost_estimate:
   use_pooled_costs: False
   use_pooled_turnover: True
```


