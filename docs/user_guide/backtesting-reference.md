# Reference


<a name="table_system_stage_methods"> </a>

## Table of standard system.data and system.stage methods

The tables in this section list all the public methods that can be used to get data
out of a system and its 'child' stages. You can also use the methods() method:

```python
system.rawdata.methods() ## works for any stage or data
```

### Explanation of columns

For brevity the name of the system instance is omitted from the 'call' column
(except where it's the actual system object we're calling directly). So for
example to get the instrument price for Eurodollar from the data object, which
is marked as *`data.get_raw_price`* we would do something like this:

```python
from systems.provided.futures_chapter15.basesystem import futures_system
name_of_system=futures_system()
name_of_system.data.get_raw_price("EDOLLAR")
```

Standard methods are in all systems. Non standard methods are for stage classes
inherited from the standard class, eg the raw data method specific to
*futures*; or the *estimate* classes which estimate parameters rather than use
fixed versions.

Common arguments are:

- `instrument_code`: A string indicating the name of the instrument
- `rule_variation_name`: A string indicating the name of the trading rule
  variation

Types are one or more of D, I, O:

- **D**iagnostic: Exposed method useful for seeing intermediate calculations
- Key **I**nput: A method which gets information from another stage. See [stage
  wiring](#stage_wiring). The description will list the source of the data.
- Key **O**utput: A method whose output is used by other stages. See [stage
  wiring](#stage_wiring). Note this excludes items only used by specific
  trading rules (notably rawdata.daily_annualised_roll)

Private methods are excluded from this table.


### System object

| Call | Standard?| Arguments | Type | Description |
|:-------------------------:|:---------:|:---------------:|:----:|:--------------------------------------------------------------:|
| `system.get_instrument_list` | Standard | | D,O | List of instruments available; either from config.instrument weights, config.instruments, or from data set|

Other methods exist to access logging and caching.

### Data object


| Call | Standard?| Arguments | Type | Description |
|:-------------------------:|:---------:|:---------------:|:----:|:--------------------------------------------------------------:|
| `data.get_raw_price` | Standard | `instrument_code` | D,O | Intraday prices if available (backadjusted if relevant)|
| `data.daily_prices` | Standard | `instrument_code` | D,O | Default price used for trading rule analysis (backadjusted if relevant)|
| `data.get_instrument_list` | Standard | | D,O | List of instruments available in data set (not all will be used for backtest)|
| `data.get_value_of_block_price_move`| Standard | `instrument_code` | D,O | How much does a $1 (or whatever) move in the price of an instrument block affect it's value? |
| `data.get_instrument_currency`|Standard | `instrument_code` | D,O | What currency does this instrument trade in? |
| `data.get_fx_for_instrument` |Standard | `instrument_code, base_currency` | D, O | What is the exchange rate between the currency of this instrument, and some base currency? |
| `data.get_instrument_raw_carry_data` | Futures | `instrument_code` | D, O | Returns a dataframe with the 4 columns PRICE, CARRY, PRICE_CONTRACT, CARRY_CONTRACT |
| `data.get_raw_cost_data`| Standard | `instrument_code` | D,O | Cost data (slippage and different types of commission) |



### [Raw data stage](#stage_rawdata)


| Call | Standard?| Arguments | Type | Description |
|:-------------------------:|:---------:|:---------------:|:----:|:--------------------------------------------------------------:|
| `rawdata.get_daily_prices` | Standard | `instrument_code` | I | `data.daily_prices`|
| `rawdata.daily_denominator_price` | Standard | `instrument_code` | O | Price used to calculate % volatility (for futures the current contract price) |
| `rawdata.daily_returns` | Standard | `instrument_code` | D, O | Daily returns in price units|
| `rawdata.get_daily_percentage_returns` | Standard | `instrument_code` | D | Daily returns as a percentage. |
| `rawdata.daily_returns_volatility` | Standard | `instrument_code` | D,O | Daily standard deviation of returns in price units |
| `rawdata.get_daily_percentage_volatility` | Standard | `instrument_code` | D,O | Daily standard deviation of returns in % (10.0 = 10%) |
| `rawdata.get_daily_vol_normalised_returns` | Standard | `instrument_code` | D | Daily returns normalised by vol (1.0 = 1 sigma) |
| `rawdata.get_instrument_raw_carry_data` | Futures | `instrument_code` | I | data.get_instrument_raw_carry_data |
| `rawdata.raw_futures_roll`| Futures | `instrument_code` | D | The raw difference between price and carry |
| `rawdata.roll_differentials` | Futures | `instrument_code` | D | The annualisation factor |
| `rawdata.annualised_roll` | Futures | `instrument_code` | D | Annualised roll |
| `rawdata.daily_annualised_roll` | Futures | `instrument_code` | D | Annualised roll. Used for carry rule. |



### [Trading rules stage (chapter 7 of book)](#rules)


| Call | Standard?| Arguments | Type | Description |
|:-------------------------:|:---------:|:---------------:|:----:|:--------------------------------------------------------------:|
| `rules.trading_rules` | Standard | | D,O | List of trading rule variations |
| `rules.get_raw_forecast` | Standard | `instrument_code`, `rule_variation_name` | D,O| Get forecast (unscaled, uncapped) |


### [Forecast scaling and capping stage (chapter 7 of book)](#stage_scale)


| Call | Standard?| Arguments | Type | Description |
|:-------------------------:|:---------:|:---------------:|:----:|:--------------------------------------------------------------:|
| `forecastScaleCap.get_raw_forecast` | Standard | `instrument_code`, `rule_variation_name` | I | `rules.get_raw_forecast` |
| `forecastScaleCap.get_forecast_scalar` | Standard / Estimate | `instrument_code`, `rule_variation_name` | D | Get the scalar to use for a forecast |
| `forecastScaleCap.get_forecast_cap` | Standard |  | D,O | Get the maximum allowable forecast |
| `forecastScaleCap.get_forecast_floor` | Standard |  | D,O | Get the minimum allowable forecast |
| `forecastScaleCap.get_scaled_forecast` | Standard | `instrument_code`, `rule_variation_name` | D | Get the forecast after scaling (after capping) |
| `forecastScaleCap.get_capped_forecast` | Standard | `instrument_code`, `rule_variation_name` | D, O | Get the forecast after scaling (after capping) |


### [Combine forecasts stage (chapter 8 of book)](#stage_combine)


| Call | Standard?| Arguments | Type | Description |
|:-------------------------:|:---------:|:---------------:|:----:|:--------------------------------------------------------------:|
| `combForecast.get_trading_rule_list` | Standard | `instrument_code` | I | List of trading rules from config or prior stage |
| `combForecast.get_all_forecasts` | Standard | `instrument_code`, (`rule_variation_list`) | D | pd.DataFrame of forecast values |
| `combForecast.get_forecast_cap` | Standard |  | I | `forecastScaleCap.get_forecast_cap` |
| `combForecast.calculation_of_raw_estimated_monthly_forecast_weights` | Estimate | `instrument_code` | D | Forecast weight calculation objects |
| `combForecast.get_forecast_weights` | Standard / Estimate| `instrument_code` | D | Forecast weights, adjusted for missing forecasts|
| `combForecast.get_forecast_correlation_matrices` | Estimate | `instrument_code` | D | Correlations of forecasts |
| `combForecast.get_forecast_diversification_multiplier` | Standard / Estimate | `instrument_code` | D | Get diversification multiplier |
| `combForecast.get_combined_forecast` | Standard | `instrument_code` | D,O | Get weighted average of forecasts for instrument |



### [Position sizing stage (chapters 9 and 10 of book)](#position_scale)


| Call | Standard?| Arguments | Type | Description |
|:-------------------------:|:---------:|:---------------:|:----:|:--------------------------------------------------------------:|
| `positionSize.get_combined_forecast` | Standard | `instrument_code` | I | `combForecast.get_combined_forecast` |
| `positionSize.get_price_volatility` | Standard | `instrument_code` | I | `rawdata.get_daily_percentage_volatility` (or `data.daily_prices`) |
| `positionSize.get_underlying_price` | Standard | `instrument_code` | I | `rawdata.daily_denominator_price` (or `data.daily_prices`); `data.get_value_of_block_price_move` |
| `positionSize.get_fx_rate` | Standard | `instrument_code` | I | `data.get_fx_for_instrument` |
| `positionSize.get_daily_cash_vol_target` | Standard | | D | Dictionary of base_currency, percentage_vol_target, notional_trading_capital, annual_cash_vol_target, daily_cash_vol_target |
| `positionSize.get_block_value` | Standard | `instrument_code` | D | Get value of a 1% move in the price |
| `positionSize.get_instrument_currency_vol` | Standard | `instrument_code` |D | Get daily volatility in the currency of the instrument |
| `positionSize.get_instrument_value_vol` | Standard | `instrument_code` |D | Get daily volatility in the currency of the trading account |
| `positionSize.get_average_position_at_subsystem_level` | Standard | `instrument_code` | D |Get ratio of target volatility vs volatility of instrument in instrument's own currency |
| `positionSize.get_subsystem_position`| Standard | `instrument_code` | D, O |Get position if we put our entire trading capital into one instrument |



### [Portfolio stage (chapter 11 of book)](#stage_portfolio)


| Call | Standard?| Arguments | Type | Description |
|:-------------------------:|:---------:|:---------------:|:----:|:--------------------------------------------------------------:|
| `portfolio.get_subsystem_position`| Standard | `instrument_code` | I |`positionSize.get_subsystem_position` |
| `portfolio.pandl_across_subsystems`| Estimate |  | I | `accounts.pandl_across_subsystems`|
| `portfolio.calculation_of_raw_instrument_weights`| Estimate | | D | Instrument weight calculation objects |
| `portfolio.get_unsmoothed_instrument_weights_fitted_to_position_lengths`| Standard / Estimate| | D |Get raw instrument weights |
| `portfolio.get_instrument_weights`| Standard / Estimate| | D |Get instrument weights, adjusted for missing instruments |
| `portfolio.get_instrument_diversification_multiplier`| Standard / Estimate | | D |Get instrument div. multiplier |
| `portfolio.get_notional_position`| Standard | `instrument_code` | D,O |Get the *notional* position (with constant risk capital; doesn't allow for adjustments when profits or losses are made) |
| `portfolio.get_buffers_for_position`| Standard | `instrument_code` | D,O |Get the buffers around the position |
| `portfolio.get_actual_position`| Standard | `instrument_code` | D,O | Get position accounting for capital multiplier|
| `portfolio.get_actual_buffers_for_position`| Standard | `instrument_code` | D,O |Get the buffers around the position, accounting for capital multiplier |



### [Accounting stage](#accounts_stage)

Inputs:

| Call | Standard?| Arguments | Type | Description |
|:-------------------------:|:---------:|:---------------:|:----:|:--------------------------------------------------------------:|
| `accounts.get_notional_position`| Standard | `instrument_code` | I | `portfolio.get_notional_position`|
| `accounts.get_actual_position`| Standard | `instrument_code` | I | `portfolio.get_actual_position`|
| `accounts.get_capped_forecast`| Standard | `instrument_code`, `rule_variation_name` | I | `forecastScaleCap.get_capped_forecast`|
| `accounts.get_instrument_list`| Standard | | I | `system.get_instrument_list` |
| `accounts.get_notional_capital`| Standard | | I | `positionSize.get_daily_cash_vol_target`|
| `accounts.get_fx_rate`| Standard | `instrument_code` | I | `positionSize.get_fx_rate`|
| `accounts.get_value_of_block_price_move`| Standard | `instrument_code` | I | `data.get_value_of_block_price_move`|
| `accounts.get_daily_returns_volatility`| Standard | `instrument_code` | I | `rawdata.daily_returns_volatility` or `data.daily_prices`|
| `accounts.get_raw_cost_data`| Standard | `instrument_code` | I | `data.get_raw_cost_data` |
| `accounts.get_buffers_for_position`| Standard | `instrument_code` | I | `portfolio.get_buffers_for_position`|
| `accounts.get_actual_buffers_for_position`| Standard | `instrument_code` | I | `portfolio.get_actual_buffers_for_position`|
| `accounts.get_instrument_diversification_multiplier`| Standard | | I | `portfolio.get_instrument_diversification_multiplier`|
| `accounts.get_instrument_weights`| Standard | | I | `portfolio.get_instrument_weights`|
| `accounts.list_of_rules_for_code`| Standard | `instrument_code` | I | `combForecast.get_trading_rule_list`|
| `accounts.has_same_rules_as_code`| Standard | `instrument_code` | I | `combForecast.has_same_rules_as_code`|


Diagnostics:

| Call | Standard?| Arguments | Type | Description |
|:-------------------------:|:---------:|:---------------:|:----:|:--------------------------------------------------------------:|
| `accounts.list_of_trading_rules`| Standard | | D | All trading rules across instruments|
| `accounts.get_instrument_scaling_factor`| Standard | `instrument_code` | D | IDM * instrument weight|
| `accounts.get_buffered_position`| Standard | `instrument_code` | D | Buffered position at portfolio level|
| `accounts.get_buffered_position_with_multiplier`| Standard | `instrument_code` | D | Buffered position at portfolio level, including capital multiplier|
| `accounts.subsystem_turnover`| Standard | `instrument_code` | D | Annualised turnover of subsystem|
| `accounts.instrument_turnover`| Standard | `instrument_code` | D | Annualised turnover of instrument position at portfolio level|
| `accounts.forecast_turnover`| Standard | `instrument_code`, `rule_variation_name` | D | Annualised turnover of forecast|
| `accounts.get_SR_cost_for_instrument_forecast`| Standard | `instrument_code`, `rule_variation_name` | D | SR cost * turnover for forecast|
| `accounts.capital_multiplier`| Standard | | D, O | Capital multiplier, ratio of actual to fixed notional capital|
| `accounts.get_actual_capital`| Standard | | D | Actual capital (fixed notional capital times multiplier)|


Accounting outputs:

| Call | Standard?| Arguments | Type | Description |
|:-------------------------:|:---------:|:---------------:|:----:|:--------------------------------------------------------------:|
| `accounts.pandl_for_instrument`| Standard | `instrument_code` | D | P&l for an instrument within a system|
| `accounts.pandl_for_instrument_with_multiplier`| Standard | `instrument_code` | D | P&l for an instrument within a system, using multiplied capital|
| `accounts.pandl_for_instrument_forecast`| Standard | `instrument_code`, `rule_variation_name` | D | P&l for a trading rule and instrument |
| `accounts.pandl_for_instrument_forecast_weighted`| Standard | `instrument_code`, `rule_variation_name` | D | P&l for a trading rule and instrument as a % of total capital |
| `accounts.pandl_for_instrument_rules`| Standard | `instrument_code` | D,O | P&l for all trading rules in an instrument, weighted |
| `accounts.pandl_for_instrument_rules_unweighted`| Standard | `instrument_code` | D,O | P&l for all trading rules in an instrument, unweighted |
| `accounts.pandl_for_trading_rule`| Standard | `rule_variation_name` | D | P&l for a trading rule over all instruments |
| `accounts.pandl_for_trading_rule_weighted`| Standard | `rule_variation_name` | D | P&l for a trading rule over all instruments as % of total capital |
| `accounts.pandl_for_trading_rule_unweighted`| Standard | `rule_variation_name` | D | P&l for a trading rule over all instruments, unweighted |
| `accounts.pandl_for_subsystem`| Standard | `instrument_code` | D | P&l for an instrument outright|
| `accounts.pandl_across_subsystems`| Standard | `instrument_code` | O,D | P&l across instruments, outright|
| `accounts.pandl_for_all_trading_rules`| Standard | | D | P&l for trading rules across whole system |
| `accounts.pandl_for_all_trading_rules_unweighted`| Standard | | D | P&l for trading rules across whole system |
| `accounts.portfolio`| Standard | | O,D | P&l for whole system |
| `accounts.portfolio_with_multiplier`| Standard | | D | P&l for whole system using multiplied capital|



<a name="Configuration_options"> </a>

## Configuration options

Below is a list of all configuration options for the system. The 'Yaml' section
shows how they appear in a yaml file. The 'python' section shows an example of
how you'd modify a config object in memory having first created it, like this:


```python
## Method one: from an existing system
from systems.provided.futures_chapter15.basesystem import futures_system
system=futures_system()
new_config=system.config

## Method two: from a config file
from syscore.fileutils import get_pathname_for_package
from sysdata.config.configdata import Config

my_config=Config(get_pathname_for_package("private", "this_system_name", "config.yaml"))

## Method three: with a blank config
from sysdata.config.configdata import Config
my_config=Config()
```

Each section also shows the project default options, which you could change
[here](#defaults).

When modifying a nested part of a config object, you can of course replace it
wholesale:

```python
new_config.instrument_weights=dict(SP500=0.5, US10=0.5))
new_config
```

Or just in part:

```python
new_config.instrument_weights['SP500']=0.2
new_config
```
If you do this make sure the rest of the config is consistent with what you've done. In either case, it's a good idea to examine the modified config once it's part of the system (since that will include any defaults) and make sure you're happy with it.



### Raw data stage

#### Volatility calculation
Represented as: dict of str, int, or float. Keywords: Parameter names Defaults:
As below

The function used to calculate volatility, and any keyword arguments passed to
it. Note if any keyword is missing then the project defaults will be used. See
['volatility calculation'](#vol_calc) for more information.

The following shows how to modify the configuration, and also the default
values:

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

Python
```python
config.volatility_calculation=dict(func="syscore.algos.robust.vol.calc", days=35, min_periods=10, vol_abs_min= 0.0000000001, vol_floor=True, floor_min_quant=0.05, floor_min_periods=100, floor_days=500)
```

If you're considering using your own function please see [configuring defaults
for your own functions](#config_function_defaults)


### Rules stage

#### Trading rules
Represented as: dict of dicts, each representing a trading rule. Keywords:
trading rule variation names. Defaults: n/a

The set of trading rules. A trading rule definition consists of a dict
containing: a *function* identifying string, an optional list of *data*
identifying strings, and *other_args* an optional dictionary containing named
parameters to be passed to the function. This is the only method that can be
used for YAML.

There are numerous other ways to define trading rules using python code. See
['Rules'](#rules) for more detail.

Note that *forecast_scalar* isn't strictly part of the trading rule definition,
but if included here will be used instead of the separate
'config.forecast_scalar' parameter (see the next section).

YAML: (example)
```yaml
trading_rules:
  ewmac2_8:
     function: systems.futures.rules.ewmac
     data:
         - "rawdata.daily_prices"
         - "rawdata.daily_returns_volatility"
     other_args:
         Lfast: 2
         Lslow: 8
     forecast_scalar: 10.6
```

Python (example)
```python
config.trading_rules=dict(ewmac2_8=dict(function="systems.futures.rules.ewmac", data=["rawdata.daily_prices", "rawdata.daily_returns_volatility"], other_args=dict(Lfast=2, Lslow=8), forecast_scalar=10.6))
```

### Forecast scaling and capping stage

Switch between fixed (default) and estimated versions as follows:

YAML: (example)
```yaml
use_forecast_scale_estimates: True
```

Python (example)
```python
config.use_forecast_scale_estimates=True
```



#### Forecast scalar (fixed)
Represented as: dict of floats. Keywords: trading rule variation names.
Default: 1.0

The forecast scalar to apply to a trading rule, if fixed scaling is being used.
If undefined the default value of 1.0 will be used.

Scalars can also be put inside trading rule definitions (this is the first
place we look):

YAML: (example)
```yaml
trading_rules:
  rule_name:
     function: systems.futures.rules.arbitrary_function
     forecast_scalar: 10.6

```

Python (example)
```python
config.trading_rules=dict(rule_name=dict(function="systems.futures.rules.arbitrary_function", forecast_scalar=10.6))
```

If scalars are not found there they can be put in separately (if you do both
then the scalar in the actual rule specification will take precedence):

YAML: (example)
```yaml
forecast_scalars:
   rule_name: 10.6
```

Python (example)
```python
config.forecast_scalars=dict(rule_name=10.6)
```

#### Forecast scalar (estimated)
Represented as: dict of str, float and int. Keywords: parameter names Default:
see below

The method used to estimate forecast scalars on a rolling out of sample basis.
Any missing config elements are pulled from the project defaults. Compulsory
arguments are pool_instruments (determines if we pool estimate over multiple
instruments) and func (str function pointer to use for estimation). The
remaining arguments are passed to the estimation function.

See [forecast scale estimation](#scalar_estimate) for more detail.

If you're considering using your own function please see [configuring defaults
for your own functions](#config_function_defaults)


YAML:
```yaml
# Here is how we do the estimation. These are also the *defaults*.
use_forecast_scale_estimates: True
forecast_scalar_estimate:
   pool_instruments: True
   func: "sysquant.estimators.forecast_scalar.forecast_scalar"
   window: 250000
   min_periods: 500
   backfill: True
```

Python (example)
```python
## pooled example
config.trading_rules=dict(pool_instruments=True, func="sysquant.estimators.forecast_scalar.forecast_scalar", window=250000, min_periods=500, backfill=True)
```


#### Forecast cap (fixed - all classes)
Represented as: float

The forecast cap to apply to a trading rule. If undefined the project default
value of 20.0 will be used.


YAML:
```yaml
forecast_cap: 20.0
```

Python
```python
config.forecast_cap=20.0
```

### Forecast combination stage

Switch between fixed (default) and estimated versions as follows:

YAML: (example)
```yaml
use_forecast_weight_estimates: True
```

Python (example)
```python
config.use_forecast_weight_estimates=True
```

Change smoothing used for both fixed and variable weights:

YAML: (example)
```yaml
forecast_weight_ewma_span: 6
```

Remove trading rules which are too expensive for a given instrument:

YAML: (example)
```yaml
post_ceiling_cost_SR: 0.13
```




#### Forecast weights (fixed)
Represented as: (a) dict of floats. Keywords: trading rule variation names. (b)
                dict of dicts, each representing the weights for an instrument.
                Keywords: instrument names Default: Equal weights, across all
                trading rules in the system

The forecast weights to be used to combine forecasts from different trading
rule variations. These can be (a) common across instruments, or (b) specified
differently for each instrument.

Notice that the default is equal weights, but these are calculated on the fly
and don't appear in the defaults file.

YAML: (a)
```yaml
forecast_weights:
     ewmac: 0.50
     carry: 0.50

```

Python (a)
```python
config.forecast_weights=dict(ewmac=0.5, carry=0.5)
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

Python (b)
```python
config.forecast_weights=dict(SP500=dict(ewmac=0.5, carry=0.5), US10=dict(ewmac=0.10, carry=0.90))
```
#### Forecast weights (estimated)

To estimate forecast weights we need to define which trading rule variations
we're using.

##### List of trading rules to get forecasts for

Represented as: (a) list of str, each a rule variation name (b) dict of list of
                str, each representing the rules for an instrument. Keywords:
                instrument names Default: Using all trading rules in the system

The rules for which forecast weights are to be calculated. These can be (a)
common across instruments, or (b) specified differently for each instrument. If
not specified will use all the rules defined in the system.

YAML: (a)
```yaml
rule_variations:
     - "ewmac"
     - "carry"

```

Python (a)
```python
config.rule_variations=["ewmac", "carry"]
```

YAML: (b)
```yaml
rule_variations:
     SP500:
      - "ewmac"
      - "carry"
     US10:
      - "ewmac"
```

Python (b)
```python
config.forecast_weights=dict(SP500=["ewmac","carry"], US10=["ewmac"])
```
##### Parameters for estimating forecast weights

See the section on [Optimisation](#optimisation)


#### Forecast diversification multiplier  (fixed)
Represented as: (a) float or (b) dict of floats with keywords: instrument_codes
Default: 1.0

This can be (a) common across instruments, or (b) we use a different one for
each instrument (would be normal if instrument weights were also different).


YAML: (a)
```yaml
forecast_div_multiplier: 1.0

```

Python (a)
```python
config.forecast_div_multiplier=1.0
```

YAML: (b)
```
forecast_div_multiplier:
     SP500: 1.4
     US10:  1.1
```

Python (b)
```python
config.forecast_div_multiplier=dict(SP500=1.4, US10=1.0)
```


#### Forecast diversification multiplier  (estimated)

See the section on [estimating correlations and diversification multipliers](#estimating-correlations-and-diversification-multipliers)



##### Forecast mapping

Represented as: dict (key names instrument names) of dict (key names: a_param,b_param, threshold). Defaults: dict(a_param = 1.0, b_param = 1.0, threshold = 0.0) equivalent to no mapping



YAML, showing defaults
```yaml
forecast_mapping:
  AUD:
    a_param: 1.0
    b_param: 1.0
    threshold: 0.0
# etc
```

Python, example of how to change certain parameters:

```python
config.forecast_mapping = dict()
config.forecast_maping['AUD'] = dict(a_param=1.0, b_param=1.0, threshold = 0.0)
config.forecast_maping['AUD']['a_param'] = 1.0
```



### Position sizing stage

#### Capital scaling parameters
Represented as: floats, int or str Defaults: See below

The annualised percentage volatility target, notional trading capital and
currency of trading capital. If any of these are undefined in the config the
default values shown below will be used.


YAML:
```yaml
percentage_vol_target: 16.0
notional_trading_capital: 1000000
base_currency: "USD"
```

Python

```python
config.percentage_vol_target=16.0
config.notional_trading_capital=1000000
config.base_currency="USD"
```

### Portfolio combination stage

Switch between fixed (default) and estimated versions as follows:

YAML: (example)
```yaml
use_instrument_weight_estimates: True
```

Python (example)
```python
config.use_instrument_weight_estimates=True
```

Change smoothing used for both fixed and variable weights:

YAML: (example)
```
instrument_weight_ewma_span: 125
```


#### Instrument weights (fixed)
Represented as: dict of floats. Keywords: instrument_codes Default: Equal
weights

The instrument weights used to combine different instruments together into the
final portfolio.

Although the default is equal weights, these are not included in the system
defaults file, but calculated on the fly.

YAML:
```yaml
instrument_weights:
    EDOLLAR: 0.5
    US10: 0.5
```

Python
```python
config.instrument_weights=dict(EDOLLAR=0.5, US10=0.5)
```

#### Instrument weights (estimated)

See the section on [Optimisation](#optimisation)


#### Instrument diversification multiplier (fixed)
Represented as: float Default: 1.0


YAML:
```yaml
instrument_div_multiplier: 1.0
```

Python
```python
config.instrument_div_multiplier=1.0
```

#### Instrument diversification multiplier (estimated)

See the section on [estimating correlations and diversification multipliers](#estimating-correlations-and-diversification-multipliers)


#### Buffering

Represented as: bool Default: see below

Which [buffering or position inertia method](#buffer) should we use?
'position': based on optimal position (position inertia), 'forecast': based on
position with a forecast of +10; or 'none': do not use a buffer. What size
should the buffer be, as a proportion of the position or average forecast
position? 0.1 is 10%.

YAML:
```yaml
buffer_method: position
buffer_size: 0.10
```



### Accounting stage

#### Buffering and position inertia

To work out the portfolio positions should we trade to the edge of the
[buffer](#buffer), or to the optimal position?

Represented as: bool Default: True

YAML:
```yaml
buffer_trade_to_edge: True
```



#### Costs

Should we use normalised Sharpe Ratio [costs](#costs), or the actual costs for
instrument level p&l (we always use SR costs for forecasts)?


YAML:
```yaml
use_SR_costs: True
```

Should we pool SR costs across instruments when working out forecast p&L?

YAML:
```yaml
forecast_cost_estimate:
   use_pooled_costs: False  ### use weighted average of SR cost * turnover across instruments with the same set of trading rules
   use_pooled_turnover: True ### Use weighted average of turnover across instruments with the same set of trading rules
```

#### Capital correction

Which capital correction method should we use?

YAML:
```yaml
capital_multiplier:
   func: syscore.capital.fixed_capital
```

Other valid functions include full_compounding and half_compounding.
