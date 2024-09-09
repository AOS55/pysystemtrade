# Processes

This section gives much more detail on certain important processes that span
multiple stages: logging, estimating correlations and diversification
multipliers, optimisation, and capital correction.

<a name="filenames"> </a>

## File names

There are a number of different ways one might want to specify path and file names. Firstly, we could use a *relative* pathname. Secondly, we might want to use an *absolute* path, which is the actual full pathname. This is useful if we want to access something outside the pysystemtrade directory structure. Finally we have the issue of OS differences; are you a '\\' or a '/' person?

For convenience I have written some functions that translate between these different formats, and the underlying OS representation.

```python
from syscore.fileutils import get_resolved_pathname, resolve_path_and_filename_for_package

# Resolve both filename and pathname jointly. Useful when writing the name of eg a configuration file
## Absolute format
### Windows (note use of double backslash in str) Make sure you include the initial backslash, or will be treated as relative format
resolve_path_and_filename_for_package("\\home\\rob\\file.csv")

### Unix. Make sure you include the initial forward slash,
resolve_path_and_filename_for_package("/home/rob/file.csv")

## Relative format to find a file in the installed pysystemtrade
### Dot format. Notice there is no initial 'dot' and we don't need to include 'pysystemtrade'
resolve_path_and_filename_for_package("syscore.tests.pricedata.csv")

# Specify the path and filename separately
resolve_path_and_filename_for_package("\\home\\rob", "file.csv")
resolve_path_and_filename_for_package("/home/rob", "file.csv")
resolve_path_and_filename_for_package("syscore.tests", "pricedata.csv")

# Resolve just the pathname
get_resolved_pathname("/home/rob")
get_resolved_pathname("\\home\\rob")
get_resolved_pathname("syscore.tests")

## DON'T USE THESE:-
### It's possible to use Unix or Windows for relative filenames, but I prefer not to, so there is a clearer disctinction between absolute and relative.
### However this works:
resolve_path_and_filename_for_package("syscore/tests/pricedata.csv")

### Similarly, I prefer not to use dot format for absolute filenames but it will work
resolve_path_and_filename_for_package(".home.rob.file.csv")

### Finally, You can mix and match the above formats in a single string, but it won't make the code very readable!
resolve_path_and_filename_for_package("\\home/rob.file.csv")

```


These functions are used internally whenever a file name is passed in, so feel free to use any of these file formats when specifying eg a configuration filename.
```
### Absolute: Windows (note use of double backslash in str)
"\\home\\rob\\file.csv"

### Absolute: Unix.
"/home/rob/file.csv"

## Relative: Dot format to find a file in the installed pysystemtrade
"syscore.tests.pricedata.csv"
```

<a name="logging"> </a>

## Logging

### Basic logging

pysystemtrade uses the [Python logging module](https://docs.python.org/3.10/library/logging.html). The system, data, config and each stage object all have a .log attribute, to allow the system to report to the user; as do the functions provided to estimate correlations and do optimisations.

By default, log messages will print out to the console (`std.out`) at level DEBUG. This what you get in sim. This is configured by function `_configure_sim()` in `syslogging.logger.py`.

If you want to change the level, or the format of the messages, then create an environment variable that points to an alternative YAML logging configuration. Something like this for Bash

```
PYSYS_LOGGING_CONFIG=/home/path/to/your/logging_config.yaml
```

It could be a file within the project, so will accept the relative dotted path format. There's an example YAML file that replicates the default sim configuration

```
PYSYS_LOGGING_CONFIG=syslogging.logging_sim.yaml
```

If you're writing your own code, and want to inform the user that something is happening you should do one of the following:

```python
## self could be a system, stage, config or data object
#
self.log.debug("this is a message at level logging.DEBUG")
self.log.info("this is a message at level logging.INFO")
self.log.warning("level logging.WARNING")
self.log.error("level logging.ERROR")
self.log.critical("level logging.CRITICAL")

# parameterise the message
log.info("Hello %s", "world")
log.info("Goodbye %s %s", "cruel", "world")
```

I strongly encourage the use of logging, rather than printing, since printing on a 'headless' automated trading server will not be visible


### Advanced logging

In my experience wading through long log files is a rather time-consuming experience. On the other hand it's often more useful to use a logging approach to monitor system behaviour than to try and create quantitative diagnostics. For this reason I'm a big fan of logging with *attributes*. This project uses a custom version of [logging.LoggerAdapter](https://docs.python.org/3.10/library/logging.html#loggeradapter-objects) for that purpose:

```python
from syslogging.logger import *

# setting attributes on logger initialisation
log = get_logger("logger name", {"stage": "first"})

# setting attributes on message creation
log.info("logger name", instrument_code="GOLD")
```

A logger is initialised with a name; should be the name of the top level calling function. Production types include price collection, execution and so on. Every time a log method is called, it will typically know one or more of the following:

- stage: Used by stages in System objects, such as 'rawdata'
- component: other parts of the top level function that have their own loggers
- currency_code: Currency code (used for fx), format 'GBPUSD'
- instrument_code: Self explanatory
- contract_date: Self explanatory, format 'yyyymm'
- broker: broker name
- clientid: IB unique identification
- strategy_name: self explanatory
- order_id: Self explanatory, used for live trading
- instrument_order_id: Self explanatory, used for live trading
- contract_order_id: Self explanatory, used for live trading
- broker_order_id: Self explanatory, used for live trading

You do need to keep track of what attributes your logger has. Generally speaking you should use this kind of pattern to write a log item

```python
# this is from the ForecastScaleCap code
#
# This log will already have type=base_system, and stage=forecastScaleCap
#
self.log.debug("Calculating scaled forecast for %s %s" % (instrument_code, rule_variation_name),
    instrument_code=instrument_code, rule_variation_name=rule_variation_name
)
```
This has the advantage of keeping the original log attributes intact. If you want to do something more complex it's worth looking at the docstring for [`syslogging.get_logger()`](/syslogging/logger.py) which shows usage patterns, including how to merge attributes.

## Optimisation

See my blog posts on optimisation:
[without](https://qoppac.blogspot.com/2016/01/correlations-weights-multipliers.html)
and [with
costs](https://qoppac.blogspot.com/2016/05/optimising-weights-with-costs.html).

I use an optimiser to calculate both forecast and instrument weights. The
process is almost identical for both.

### The optimisation function, and data

From the config
```
forecast_weight_estimate: ## can also be applied to instrument weights
   func: sysquant.optimisation.generic_optimiser.genericOptimiser ## this is the only function provided
   pool_instruments: True ## not used for instrument weights
   frequency: "W" ## other options: D, M, Y

```

I recommend using weekly data, since it speeds things up and doesn't affect out
of sample performance.

### Removing expensive assets (forecast weights only)

Again I recommend you check out this [blog
post](https://qoppac.blogspot.com/2016/05/optimising-weights-with-costs.html).

```
forecast_weight_estimate:
   ceiling_cost_SR: 0.13 ## Max cost to allow for assets, annual SR units.
    
```

See ['costs'](#costs) to see how to configure pooling when estimating the costs
of forecasts.

By default this is set to 9999 which effectively means that all trading rules are included at the optimisation stage. However the use of `post_ceiling_cost_SR` can be used to remove rules that are too expensive. This is recommended if you are pooling gross returns.


### Pooling gross returns (forecast weights only)

Pooling across instruments is only available when calculating forecast weights.
Again I recommend you check out this [blog
post](https://qoppac.blogspot.com/2016/05/optimising-weights-with-costs.html).
Only instruments whose rules have survived the application of a ceiling cost (`ceiling_cost_SR`)
will be included in the pooling process. If you want to pool all instruments, regardless of costs, then you should set `ceiling_cost_SR` to be some high number, and use `post_ceiling_cost_SR` instead to eliminate expensive rules after the optimisation is complete (this is the default).


```
forecast_weight_estimate:
   pool_gross_returns: True ## pool gross returns for estimation
forecast_cost_estimate:
   use_pooled_costs: False  ### use weighted average of [SR cost * turnover] across instruments with the same set of trading rules
   use_pooled_turnover: True ### Use weighted average of turnover across instruments with the same set of trading rules
```

See ['costs'](#costs) to see how to configure pooling when estimating the costs
of forecasts. 



### Working out net costs (both instrument and forecast weights)

Again I recommend you check out this [blog
post](https://qoppac.blogspot.com/2016/05/optimising-weights-with-costs.html).

```
forecast_weight_estimate:  ## can also be applied to instrument weights
   equalise_gross: False ## equalise gross returns so that only costs are used for optimisation
   cost_multiplier: 0.0 ## multiply costs by this number. Zero means grosss returns used. Higher than 1 means costs will be inflated. Use zero if apply_cost_weight=True (see later)
```


### Time periods


There are three options available for the fitting period - `expanding`
(recommended), `in sample` (never!) and `rolling`. See Chapter 3 of my book.

From the config
```
   date_method: expanding ## other options: in_sample, rolling
   rollyears: 20 ## only used when rolling
```

### Moment estimation

To do an optimisation we need estimates of correlations, means, and standard
deviations.

From the config

```
forecast_weight_estimate:  ## can also be applied to instrument weights
   correlation_estimate:
     func: sysquant.estimators.correlation_estimator.correlationEstimator
     using_exponent: False
     ew_lookback: 500
     min_periods: 20
     floor_at_zero: True

   mean_estimate:
     func: sysquant.estimators.mean_estimator.meanEstimator
     using_exponent: False
     ew_lookback: 500
     min_periods: 20

   vol_estimate:
     func: sysquant.estimators.stdev_estimator.stdevEstimator
     using_exponent: False
     ew_lookback: 500
     min_periods: 20
```

If you're using shrinkage or single period optimisation I'd suggest using an
exponential weight for correlations, means, and volatility.

### Methods

There are five methods provided to optimise with in the function I've included.
Personally I'd use handcrafting, which is the default.

#### Equal weights

This will give everything in the optimisation equal weights.

```
   method: equal_weights
```



Tip: Set `date_method: in_sample` to speed things up.


#### One period (not recommend)

This is the classic Markowitz optimisation with the option to equalise Sharpe
Ratios (makes things more stable) and volatilities. Since we're dealing with
things that should have the same volatility anyway the latter is something I
recommend doing.

```
   method: one_period
   equalise_SR: True
   ann_target_SR: 0.5  ## Sharpe we head to if we're equalising
   equalise_vols: True
```

Notice that if you equalise Sharpe then this will override the effect of any
pooling or changes to cost calculation.

#### Bootstrapping (recommended, but slow)

Bootstrapping is no longer implemented; after a code refactoring I couldn't think of an elegant way of doing it.

#### Shrinkage (okay, but tricky to calibrate)

This is a basic shrinkage towards a prior of equal sharpe ratios, and equal
correlations; with priors equal to the average of estimates from the data.
Shrinkage of 1.0 means we use the priors, 0.0 means we use the empirical
estimates.

```yaml
   method: shrinkage
   shrinkage_SR: 0.90
   ann_target_SR: 0.5  ## Sharpe we head to if we're shrinking
   shrinkage_corr: 0.50
   equalise_vols: True

```

Notice that if you equalise Sharpe by shrinking with a factor of 1.0, then this
will override the effect of any pooling or changes to cost calculation.


#### Handcrafting (recommended)

See [my series of blog posts](https://qoppac.blogspot.com/2018/12/portfolio-construction-through.html)

```yaml
   method: handcraft
   equalise_SR: False # optional
   equalise_vols: True ## This *must* be true for the code to work
```


### Post processing

If we haven't accounted for costs earlier (eg by setting `cost_multiplier=0`)
then we can adjust our portfolio weights according to costs after they've been
calculated. See this blog post [blog
post](https://qoppac.blogspot.com/2016/05/optimising-weights-with-costs.html).

If weights are *cleaned*, then in a fitting period when we need a weight, but
none has been calculated (due to insufficient data for example), an instrument
is given a share of the weight.


```yaml
   apply_cost_weight: False
   cleaning: True

```

At this stage the other cost ceiling will be applied (`config.post_ceiling_cost_SR`). 

<a name="divmult"> </a>

## Estimating correlations and diversification multipliers

See [my blog
post](https://qoppac.blogspot.com/2016/01/correlations-weights-multipliers.html)


You can estimate diversification multipliers for both instruments (IDM - see
chapter 11) and forecasts (FDM - see chapter 8).

The first step is to estimate *correlations*. The process is the same, except
that for forecasts you have the option to pool instruments together. As the
following YAML extract shows I recommend estimating these with an exponential
moving average on weekly data:

```yaml
forecast_correlation_estimate:
   pool_instruments: True ## not available for IDM estimation
   func:sysquant.estimators.pooled_correlation.pooled_correlation_estimator ## function to use for estimation. This handles both pooled and non pooled data
   frequency: "W"   # frequency to downsample to before estimating correlations
   date_method: "expanding" # what kind of window to use in backtest
   using_exponent: True  # use an exponentially weighted correlation, or all the values equally
   ew_lookback: 250 ## lookback when using exponential weighting
   min_periods: 20  # min_periods, used for both exponential, and non exponential weighting
   cleaning: True  # Replace missing values with an average so we don't lose data early on
   floor_at_zero: True
   forward_fill_data: True

instrument_correlation_estimate:
   func: sysquant.estimators.correlation_over_time.correlation_over_time_for_returns # these aren't pooled'
   frequency: "W"
   date_method: "expanding"
   using_exponent: True
   ew_lookback: 250
   min_periods: 20
   cleaning: True
   rollyears: 20
   floor_at_zero: True
   forward_fill_price_index: True # we ffill prices not returns or goes wrong

```

Once we have correlations, and the forecast or instrument weights, it's a
trivial calculation.

```yaml
instrument_div_mult_estimate:
   func: sysquant.estimators.diversification_multipliers.diversification_multiplier_from_list
   ewma_span: 125   ## smooth to apply, business day space
   div_mult: 2.5 ## maximum allowable multiplier
```

I've included a smoothing function, otherwise jumps in the multiplier will
cause trading in the backtest. Note that the FDM is calculated on an instrument
by instrument basis, but if instruments have had their forecast weights and
correlations estimated on a pooled basis they'll have the same FDM. It's also a
good idea to floor negative correlations at zero to avoid inflating the DM to
very high values.

# Capital correction: Varying capital

Capital correction is the process by which we change the capital we have at
risk, and thus our positions, according to any profits or losses made. Most of
pysystemtrade assumes that capital is *fixed*. This has the advantage that risk
is stable over time, and account curves can more easily be interpreted. However
a more common method is to use *compounded* capital, where profits are added to
capital and losses deducted. If we make money then our capital, and the risk
we're taking, and the size of our positions, will all increase over time.

There is much more in this [blog
post](https://qoppac.blogspot.com/2016/06/capital-correction-pysystemtrade.html).
Capital correction is controlled by the following config parameter which
selects the function used for correction using the normal dot argument (the
default here being the function `fixed_capital` in the module
`syscore.capital`)

YAML:
```yaml
capital_multiplier:
   func: syscore.capital.fixed_capital
```

Other functions I've written are `full_compounding` and `half_compounding`.
Again see the blog post [blog
post](https://qoppac.blogspot.com/2016/06/capital-correction-pysystemtrade.html)
for more detail.

To get the varying capital multiplier which the chosen method calculates use
`system.accounts.capital_multiplier()`. The multiplier will be 1.0 at a given
time if the variable capital is identical to the fixed capital.

Here's a list of methods with their counterparts for both fixed and variable
capital:

|                             | Fixed capital | Variable capital |
|:-------------------------:|:---------:|:---------------:|
| Get capital at risk | `positionSize.get_daily_cash_vol_target()['notional_trading_capital']` | `accounts.get_actual_capital()` |
| Get position in a system portfolio | `portfolio.get_notional_position` | `portfolio.get_actual_position` |
| Get buffers for a position | `portfolio.get_buffers_for_position` | `portfolio.get_actual_buffers_for_position` |
| Get buffered position | `accounts.get_buffered_position`| `accounts.get_buffered_position_with_multiplier`|
| Get p&l for instrument at system level | `accounts.pandl_for_instrument`| `accounts.pandl_for_instrument_with_multiplier`|
| P&L for whole system | `accounts.portfolio`| `accounts.portfolio_with_multiplier`|

All other methods in pysystemtrade use fixed capital.