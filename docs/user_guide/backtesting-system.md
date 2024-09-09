# System

An instance of a system object consists of a number of **stages**, some
**data**, and normally a **config** object.


### Pre-baked systems

We can create a system from an existing 'pre-baked system'. These include a
ready made set of data, a list of stages, and a config.

```python
from systems.provided.futures_chapter15.basesystem import futures_system
system=futures_system()
```

We can override what's provided, and include our own data, and / or
configuration, in such a system:

```python
system=futures_system(data=my_data)
system=futures_system(config=my_config)
system=futures_system(data=my_data, config=my_config)
```

Finally we can also create our own [trading rules object](#rules), and pass
that in. This is useful for interactive model development. If for example we've
just written a new rule on the fly:

```python
my_rules=dict(rule=a_new_rule)
system=futures_system(trading_rules=my_rules) ## we probably need a new configuration as well here if we're using fixed forecast weights
```


<a name="futures_system"> </a>

#### [Futures system for chapter 15](/systems/provided/futures_chapter15/basesystem.py)

This system implements the framework in chapter 15 of my book.

```python
from systems.provided.futures_chapter15.basesystem import futures_system
system=futures_system()
```


Effectively it implements the following;

```python
data=csvFuturesSimData() ## or the data object that has been passed
config=Config("systems.provided.futures_chapter15.futuresconfig.yaml") ## or the config object that is passed

## Optionally the user can provide trading_rules (something which can be parsed as a set of trading rules); however this defaults to None in which case
##     the rules in the config will be used.

system=System([Account(), Portfolios(), PositionSizing(), RawData(), ForecastCombine(),
                   ForecastScaleCap(), Rules(trading_rules)], data, config)
```

<a name="estimated_system"> </a>

#### [Estimated system for chapter 15](/systems/provided/futures_chapter15/estimatedsystem.py)


This system implements the framework in chapter 15 of my book, but includes
estimation of forecast scalars, instrument and forecast diversification
multiplier, instrument and forecast weights.


```python
from systems.provided.futures_chapter15.estimatedsystem import futures_system
system=futures_system()
```


Effectively it implements the following;

```python
data=csvFuturesSimData() ## or the data object that has been passed
config=Config("systems.provided.futures_chapter15.futuresestimateconfig.yaml") ## or the config object that is passed

## Optionally the user can provide trading_rules (something which can be parsed as a set of trading rules); however this defaults to None in which case
##     the rules in the config will be used.

system=System([Account(), Portfolios(), PositionSizing(), RawData(), ForecastCombine(),
                   ForecastScaleCap(), Rules(trading_rules)], data, config)
```

The key configuration differences from the standard system are that the
estimation parameters:
 - `use_forecast_scale_estimates`
 - `use_forecast_weight_estimates`
 - `use_instrument_weight_estimates`
-  `use_forecast_div_mult_estimates`
-  `use_instrument_div_mult_estimates`

 ... are all set to `True`.

Warning: Be careful about changing a system from estimated to non estimated 'on
the fly' by varying the estimation parameters (in the form use_*_estimates).
See [persistence of 'switched' stage objects](#switch_persistence) for more
information.


### Using the system object

The system object doesn't do very much in itself, except provide access to its
'child' stages, its cache, and a limited number of methods. The child stages
are all attributes of the parent system.

#### Accessing child stages, data, and config within a system


For example to get the final portfolio level 'notional' position, which is in
the child stage named `portfolio`:

```python
system.portfolio.get_notional_position("EDOLLAR")
```

We can also access the methods in the data object that is part of every system:

```python
system.data.get_raw_price("EDOLLAR")
```

For a list of all the methods in a system and its stages see [stage methods](#table_system_stage_methods). Alternatively:
```python
system ## lists all the stages
system.accounts.methods() ## lists all the methods in a particular stage
system.data.methods() ## also works for data
```



We can also access or change elements of the config object:

```python
system.config.trading_rules
system.config.instrument_div_multiplier=1.2
```

#### System methods

The base system only has a public few methods of it's own (apart from those used for
caching, described below):

`system.get_instrument_list()` This will get the list of instruments in the
system, either from the config object if it contains instrument weights, or
from the data object.


These methods also get lists of instruments, see [instrument documentation](/docs/instruments.md) for more.
```
get_list_of_bad_markets
get_list_of_markets_not_trading_but_with_data
get_list_of_duplicate_instruments_to_remove
get_list_of_ignored_instruments_to_remove
get_list_of_instruments_to_remove
get_list_of_markets_with_trading_restrictions'
```

`system.log` provides access to the system's log. See [logging](#logging) for more 
details.

<a name="caching"> </a>

### System Caching and pickling

Pulling in data and calculating all the various stages in a system can be a
time consuming process. So the code supports caching. When we first ask for
some data by calling a stage method, like
`system.portfolio.get_notional_position("EDOLLAR")`, the system first checks to
see if it has already pre-calculated this figure. If not then it will calculate
the figure from scratch. This in turn may involve calculating preliminary
figures that are needed for this position, unless they've already been
pre-calculated. So for example to get a combined forecast, we'd already need to
have all the individual forecasts from different trading rule variations for a
particular instrument. Once we've calculated a particular data point, which
could take some time, it is stored in the system object cache (along with any
intermediate results we also calculated). The next time we ask for it will be
served up immediately.

Most of the time you shouldn't need to worry about caching. If you're testing
different configurations, or updating or changing your data, you just have to
make sure you recreate the system object from scratch after each change. A new
system object will have an empty cache.

Cache labels

```python
from copy import copy
from systems.provided.futures_chapter15.basesystem import futures_system

system=futures_system()
system.combForecast.get_combined_forecast("EDOLLAR")

## What's in the cache?
system.cache.get_cache_refs_for_instrument("EDOLLAR")

   [_get_forecast_scalar_fixed in forecastScaleCap for instrument EDOLLAR [carry] , get_raw_forecast in rules for instrument EDOLLAR [ewmac32_128]  ...


## Let's make a change to the config:
system.config.forecast_div_multiplier=0.1

## This will produce the same result, as we've cached the result
system.combForecast.get_combined_forecast("EDOLLAR")

## but if we make a new system with the new config...
system=futures_system(config=system.config)

## check the cache is empty:
system.cache.get_cache_refs_for_instrument("EDOLLAR")

## ... we get a different result
system.combForecast.get_combined_forecast("EDOLLAR")

## We can also turn caching off
## First clear the cache
system.cache.clear()

## ... should be nothing here
system.cache.get_cache_refs_for_instrument("EDOLLAR")

## Now turn off caching
system.cache.set_caching_off()

## Now after getting some data:
system.combForecast.get_combined_forecast("EDOLLAR")

##.... the cache is still empty
system.cache.get_cache_refs_for_instrument("EDOLLAR")

## if we change the config
system.config.forecast_div_multiplier=100.0

## ... then the result will be different without needing to create a new system
system.combForecast.get_combined_forecast("EDOLLAR")
```

### Pickling and unpickling saved cache data

It can take a while to backtest a large system. It's quite useful to be able to
save the contents of the cache and reload it later. I use the python pickle
module to do this.

For boring python related reasons not all elements in the cache will be saved.
The accounting information, and the optimisation functions used when estimating
weights, will be excluded and won't be reloaded.


```python
from systems.provided.futures_chapter15.basesystem import futures_system

system = futures_system()
system.accounts.portfolio().sharpe() ## does a whole bunch of calculations that will be saved in the cache. A bit slow...

system.cache.get_itemnames_for_stage("accounts") # includes 'portfolio'

# if I asked for this again, it's superfast
system.accounts.portfolio().sharpe()

## save it down
system.cache.pickle("private.this_system_name.system.pck") ## Using the 'dot' method to identify files in the workspace. use any file extension you like


## Now in a new session
system = futures_system()
system.cache.get_items_with_data() ## check empty cache

system.cache.unpickle("private.this_system_name.system.pck")

system.cache.get_items_with_data() ## Cache is now populated. Any existing data in system instance would have been removed.
system.get_itemnames_for_stage("accounts") ## now doesn't include ('accounts', 'portfolio', 'percentageTdelayfillTroundpositionsT')

system.accounts.portfolio().sharpe() ## Not coming from the cache, but this will run much faster and reuse many previous calculations

```

See [here](#filenames) for how to specify filenames in pysystemtrade.


### Advanced caching

It's also possible to selectively delete certain cached items, whilst keeping
the rest of the system intact. You shouldn't do this without understanding
[stage wiring](#stage_wiring). You need to have a good knowledge of the various
methods in each stage, to understand the downstream implications of either
deleting or keeping a particular data value.

There are four attributes of data stored in the cache:

1. Unprotected data that is deleted from the cache on request
2. Protected data that wouldn't normally be deletable. Outputs of lengthy
   estimations are usually protected
3. Data specific to a particular instrument (can be protected or unprotected)
4. Data which applies to the whole system; or at least to multiple instruments
   (can be protected or unprotected)

Protected items and items common across the system wouldn't normally be deleted
since they are usually the slowest things to calculate.

For example here are is how we'd check the cache after getting a notional
position (which generates a huge number of intermediate results). Notice the
way we can filter and process lists of cache keys.


```python
system.portfolio.get_notional_position("EDOLLAR")

system.cache.get_items_with_data() ## this list everything.
system.cache.get_cacherefs_for_stage("portfolio") ## lists everything in a particular stage
system.cache.get_items_with_data().filter_by_stage_name("portfolio") ## more idiomatic way
system.cache._get_protected_items() ## lists protected items
system.cache.get_items_with_data().filter_by_instrument_code("EDOLLAR") ## list items with data for an instrument
system.cache.get_cache_refs_across_system() ## list items that run across the whole system or multiple instruments

system.cache.get_items_with_data().filter_by_itemname('get_capped_forecast').unique_list_of_instrument_codes() ## lists all instruments with a capped forecast

```

Now if we want to selectively clear parts of the cache we could do one of the
following:

```python
system.cache.delete_items_for_instrument(instrument_code) ## deletes everything related to an instrument: NOT protected, or across system items
system.cache.delete_items_across_system() ## deletes everything that runs across the system; NOT protected, or instrument specific items
system.cache.delete_all_items() ## deletes all items relating to an instrument or across the system; NOT protected
system.cache.delete_items_for_stage(stagename) ## deletes all items in a particular stage, NOT protected

## Be careful with these:
system.cache.delete_items_for_instrument(instrument_code, delete_protected=True) ## deletes everything related to an instrument including protected; NOT across system items
system.cache.delete_items_across_system(delete_protected=True) ## deletes everything including protected items that runs across the system; NOT instrument specific items
## If you run these you will empty the cache completely:
system.cache.delete_item(itemname) ## delete everything in the cache for a paticluar item - including protected and across system items
system.cache.delete_all_items(delete_protected=True) ## deletes all items relating to an instrument or across the system - including protected items
system.cache.delete_items_for_stage(stagename, delete_protected=True) ## deletes all items in a particular stage - including protected items
system.cache.clear()
```




#### Advanced Caching when backtesting.

Creating a new system might be very slow. For example estimating the forecast
scalars, and instrument and forecast weights from scratch will take time,
especially if you're bootstrapping. For this reason they're protected from
cache deletion.

A possible workflow might be:

1. Create a basic version of the system, with all the instruments and trading
   rules that you need.
2. Run a backtest. This will optimise the instrument and forecast weights, and
   estimate forecast scalars (to really speed things up here you could use a
   faster method like shrinkage. See the section on
   [optimisation](#optimisation) for more information.).
3. Change and modify the system as desired. Make sure you change the config
   object that is embedded within the system. Don't create a new system object.
4. After each change, run `system.delete_all_items()` before backtesting the
   system again. Anything that is protected won't be re-estimated, speeding up
   the process.
5. Back to step 3, until you're happy with the results (but beware of implicit
   overfitting!)
6. run `system.delete_all_items(delete_protected=True)` or equivalently create
   a new system object
7. Run a backtest. This will re-estimate everything from scratch for the final
   version of your system.

Another reason to use caching would be if you want to do your initial
exploration with just a subset of the data.

1. Create a basic version of the system, with a subset of the instruments and
   trading rules that you need.
2. .... 6 as before
7. Add the rest of your instruments to your data set.
8. Run a backtest. This will re-estimate everything from scratch for the final
   version of your system, including the expanded instrument weights.

Here's a simple example of using caching in system development:

```python
from systems.provided.futures_chapter15.basesystem import futures_system
system=futures_system()

# step 2
system.accounts.portfolio().curve() ## effectively runs an entire backtest

# step 3
new_idm=1.1 ## new IDM
system.config.instrument_div_multiplier=new_idm

# step 4
system.cache.delete_all_items() ## protected items won't be affected
system.accounts.portfolio().curve() ## re-run the backtest

# Assuming we're happy- move on to step 6
system.cache.delete_all_items(delete_protected=True)

## or alternatively recreate the system using the modified config:
new_config=system.config
system=futures_system(config=new_config)

## Step 7
system.accounts.portfolio().curve() ## re-run the final backtest

```


#### Advanced caching behaviour with a live trading system

Although the project doesn't yet include a live trading system, the caching
behaviour of the system object will make it more suitable for a live system. If
we're trading slowly enough, eg every day, we might be want to to do this
overnight:

1. Get new prices for all instruments
2. Save these in wherever our data object is looking
3. Create a new system object from scratch
4. Run the system by asking for optimal positions for all instruments

Step 4 might be very involved and slow, but markets are closed so that's fine.

Then we do the following throughout the day:

5. Wait for a new price to come in (perhaps through a message bus)
6. So we don't subsequently use stale prices delete everything specific to that
   instrument with `system.delete_items_for_instrument(instrument_code)`
7. Re-calculate the optimal positions for this instrument
8. This is then passed to our trading algo

Because we've deleted everything specific to the instrument we'll recalculate
the positions, and all intermediate stages, using the new price. However we
won't have to repeat lengthy calculations that cut across instruments, such as
correlation estimates, risk overlays, cross sectional data or weight
estimation. That can wait till our next overnight run.


### Very advanced: Caching in new or modified code

If you're going to write new methods for stages (or a complete new stage) you
need to follow some rules to keep caching behaviour consistent.

The golden rule is a particular value should only be cached once, in a single
place.

So the data object methods should never cache; they should just behave like
'pipes' passing data through to system stages on request. This saves the hassle
of having to write methods which delete items in the data object cache as well
as the system cache.

Similarly most stages contain 'input' methods, which do no calculations but get
the 'output' from an earlier stage and then 'serve' it to the rest of the
stage. These exist to simplify changing the internal wiring of a stage and
reduce the coupling between methods from different stages. These should also
never cache; or again we'll be caching the same data multiple times ( see
[stage wiring](#stage_wiring) ).

You should cache as early as possible; so that all the subsequent stages that
need that data item already have it. Avoid looping back, where a stage uses
data from a later stage, as you may end up with infinite recursion.

The cache 'lives' in the parent system object in the attribute `system.cache`,
*not* the stage with the relevant method. There are standard functions which
will check to see if an item is cached in the system, and then call a function
to calculate it if required (see below). To make this easier when a stage
object joins a system it gains an attribute self.parent, which will be the
'parent' system.

Think carefully about whether your method should create data that is protected
from casual cache deletion. As a rule anything that cuts across instruments and
/ or changes slowly should be protected. Here are the current list of protected
items:

- Estimated Forecast scalars
- Estimated Forecast weights
- Estimated Forecast diversification multiplier
- Estimated Forecast correlations
- Estimated Instrument diversification multiplier
- Estimated Instrument weights
- Estimated Instrument correlations

To this list I'd add any cross sectional data, and anything that measures
portfolio risk (not yet implemented in this project).

Also think about whether you're going to cache any complex objects that
`pickle` might have trouble with, like class instances. You need to flag these
up as problematic.

Caching is implemented (as of version 0.14.0 of this project) by python
decorators attached to methods in the stage objects. There are four decorators
used by the code:

- `@input` - no caching done, input method see [stage wiring](#stage_wiring)
- `@dont_cache` - no caching done, used for very trivial calculations that
  aren't worth caching
- `@diagnostic()` - caching done within the body of a stage
- `@output()` - caching done producing an output

Notice the latter two decorators are always used with brackets, the former
without. Labelling your code with the `@input` or `@dont_cache` decorators has
no effect whatsoever, and is purely a labelling convention.

Similarly the `@diagnostic` and `@output` decorators perform exactly the same
way; it's just helpful to label your [stage wiring](#stage_wiring) and make
output functions clear.

The latter two decorators take two optional keyword arguments which default to
False `@diagnostic(protected=False, not_pickable=False)`. Set `protected=True`
if you want to stop casual deletion of the results of a method. Set
`not_pickable=True` if the results of a method will be a complex nested object
that pickle will struggle with.

Here are some fragments of code from the forecast_combine.py file showing cache
decorators in use:

```python

    @dont_cache
    def _use_estimated_weights(self):
        return str2Bool(self.parent.config.use_forecast_weight_estimates)

    @dont_cache
    def _use_estimated_div_mult(self):
        # very simple
        return str2Bool(self.parent.config.use_forecast_div_mult_estimates)

    @input
    def get_forecast_cap(self):
        """
        Get the forecast cap from the previous module

        :returns: float

        KEY INPUT
        """

        return self.parent.forecastScaleCap.get_forecast_cap()

    @diagnostic()
    def get_trading_rule_list_estimated_weights(self, instrument_code):
        # snip

    @diagnostic(protected=True, not_pickable=True)
    def get_forecast_correlation_matrices_from_code_list(self, codes_to_use):
        # snip

```



### Creating a new 'pre-baked' system

It's worth creating a new pre-baked system if you're likely to want to repeat a
backtest, or when you've settled on a system you want to paper or live trade.

The elements of a new pre-baked system will be:

1. New stages, or a different choice of existing stages.
2. A set of data (either new or existing)
3. A configuration file
4. A python function that loads the above elements, and returns a system object

To remain organised it's good practice to save your configuration file and any
python functions you need into a directory like
`pysystemtrade/private/this_system_name/` (you'll need to create a couple of
directories first). If you plan to contribute to github, just be careful to
avoid adding 'private' to your commit ( [you may want to read
this](https://24ways.org/2013/keeping-parts-of-your-codebase-private-on-github/)
). If you have novel data you're using for this system, you may also want to
save it in the same directory.

Then it's a case of creating the python function. Here is an extract from the
[futuressystem for chapter
15](/systems/provided/futures_chapter15/basesystem.py)

```python
## We probably need these to get our data

from sysdata.sim.csv_futures_sim_data import csvFuturesSimData
from sysdata.config.configdata import Config

## We now import all the stages we need
from systems.forecasting import Rules
from systems.basesystem import System
from systems.forecast_combine import ForecastCombine
from systems.forecast_scale_cap import ForecastScaleCap
from systems.rawdata import RawData
from systems.positionsizing import PositionSizing
from systems.portfolio import Portfolios
from systems.accounts.accounts_stage import Account


def futures_system(data=None, config=None, trading_rules=None):
    if data is None:
        data = csvFuturesSimData()

    if config is None:
        config = Config("systems.provided.futures_chapter15.futuresconfig.yaml")

    ## It's nice to keep the option to dynamically load trading rules but if you prefer you can remove this and set rules=Rules() here
    rules = Rules(trading_rules)

    ## build the system
    system = System([Account(), Portfolios(), PositionSizing(), RawData(), ForecastCombine(),
                     ForecastScaleCap(), rules], data, config)

    return system
```


### Changing or making a new System class

It shouldn't be necessary to modify the `System()` class or create new ones.


<a name="stage_general"> </a>