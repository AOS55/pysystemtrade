# Backtesting

This is the user guide for using pysystemtrade as a backtesting platform. Before reading this you should have gone through the [introduction.](/docs/introduction.md)

Related documents:

- [Storing futures and spot FX data](/docs/data.md)
- [Using pysystemtrade as a production trading environment](/docs/production.md)
- [Connecting pysystemtrade to interactive brokers](/docs/IB.md)
- [Recent undocumented changes](/docs/recent_changes.md)

## Overview

Provide an overview of the backtesting process and how it fits together.


This guide is divided into four parts. The first ['How do I?'](#how_do_i)
explains how to do many common tasks. The second part ['Guide'](#guide) details
the relevant parts of the code, and explains how to modify or create new parts.
The third part ['Processes'](#Processes) discusses certain processes that cut
across multiple parts of the code in more detail. The final part
['Reference'](#reference) includes lists of methods and parameters.

<!-- Table of Contents
=================

* [How do I?](#how-do-i)
   * [How do I.... Experiment with a single trading rule and instrument](#how-do-i-experiment-with-a-single-trading-rule-and-instrument)
   * [How do I....Create a standard futures backtest](#how-do-icreate-a-standard-futures-backtest)
   * [How do I....Create a futures backtest which estimates parameters](#how-do-icreate-a-futures-backtest-which-estimates-parameters)
   * [How do I....See intermediate results from a backtest](#how-do-isee-intermediate-results-from-a-backtest)
   * [How do I....See how profitable a backtest was](#how-do-isee-how-profitable-a-backtest-was)
   * [How do I....Change backtest parameters](#how-do-ichange-backtest-parameters)
      * [Option 1: Change the configuration file](#option-1-change-the-configuration-file)
      * [Option 2: Change the configuration object; create a new system](#option-2-change-the-configuration-object-create-a-new-system)
      * [Option 3: Change the configuration object within an existing system (not recommended - advanced)](#option-3-change-the-configuration-object-within-an-existing-system-not-recommended---advanced)
      * [Option 4: Change the project defaults (definitely not recommended)](#option-4-change-the-project-defaults-definitely-not-recommended)
   * [How do I....Run a backtest on a different set of instruments](#how-do-irun-a-backtest-on-a-different-set-of-instruments)
      * [Change instruments: Change the configuration file](#change-instruments-change-the-configuration-file)
      * [Change instruments: Change the configuration object](#change-instruments-change-the-configuration-object)
   * [How do I.... run the backtest only on more recent data](#how-do-i-run-the-backtest-only-on-more-recent-data)
   * [How do I....Run a backtest on all available instruments](#how-do-irun-a-backtest-on-all-available-instruments)
   * [How do I.... Exclude some instruments from the backtest](#how-do-i-exclude-some-instruments-from-the-backtest)
   * [How do I.... Exclude some instruments from having positive instrument weights](#how-do-i-exclude-some-instruments-from-having-positive-instrument-weights)
   * [How do I....Create my own trading rule](#how-do-icreate-my-own-trading-rule)
      * [Writing the function](#writing-the-function)
      * [Adding the trading rule to a configuration](#adding-the-trading-rule-to-a-configuration)
   * [How do I....Use different data or instruments](#how-do-iuse-different-data-or-instruments)
   * [How do I... Save my work](#how-do-i-save-my-work)
* [Guide](#guide)
   * [Data](#data)
      * [Using the standard data objects](#using-the-standard-data-objects)
         * [Generic data objects](#generic-data-objects)
         * [The csvFuturesSimData object](#the-csvfuturessimdata-object)
         * [The arcticSimData object](#the-arcticsimdata-object)
            * [Setting up your Arctic and Mongo DB databases](#setting-up-your-arctic-and-mongo-db-databases)
            * [Using dbFuturesSimData](#using-dbfuturessimdata)
      * [Creating your own data objects](#creating-your-own-data-objects)
         * [The Data() class](#the-data-class)
   * [Configuration](#configuration)
      * [Creating a configuration object](#creating-a-configuration-object)
         * [1) Creating a configuration object with a dictionary](#1-creating-a-configuration-object-with-a-dictionary)
         * [2) Creating a configuration object from a file](#2-creating-a-configuration-object-from-a-file)
         * [3) Creating a configuration object from a pre-baked system](#3-creating-a-configuration-object-from-a-pre-baked-system)
         * [4) Creating a configuration object from a list](#4-creating-a-configuration-object-from-a-list)
         * [5) Creating configuration files from .csv files](#5-creating-configuration-files-from-csv-files)
      * [Project defaults](#project-defaults)
         * [Handling defaults when you change certain functions](#handling-defaults-when-you-change-certain-functions)
         * [How the defaults work](#how-the-defaults-work)
      * [Viewing configuration parameters](#viewing-configuration-parameters)
      * [Modifying configuration parameters](#modifying-configuration-parameters)
      * [Using configuration in a system](#using-configuration-in-a-system)
      * [Including your own configuration options](#including-your-own-configuration-options)
      * [Saving configurations](#saving-configurations)
      * [Modifying the configuration class](#modifying-the-configuration-class)
   * [System](#system)
      * [Pre-baked systems](#pre-baked-systems)
         * [<a href="/systems/provided/futures_chapter15/basesystem.py">Futures system for chapter 15</a>](#futures-system-for-chapter-15)
         * [<a href="/systems/provided/futures_chapter15/estimatedsystem.py">Estimated system for chapter 15</a>](#estimated-system-for-chapter-15)
      * [Using the system object](#using-the-system-object)
         * [Accessing child stages, data, and config within a system](#accessing-child-stages-data-and-config-within-a-system)
         * [System methods](#system-methods)
      * [System Caching and pickling](#system-caching-and-pickling)
      * [Pickling and unpickling saved cache data](#pickling-and-unpickling-saved-cache-data)
      * [Advanced caching](#advanced-caching)
         * [Advanced Caching when backtesting.](#advanced-caching-when-backtesting)
         * [Advanced caching behaviour with a live trading system](#advanced-caching-behaviour-with-a-live-trading-system)
      * [Very advanced: Caching in new or modified code](#very-advanced-caching-in-new-or-modified-code)
      * [Creating a new 'pre-baked' system](#creating-a-new-pre-baked-system)
      * [Changing or making a new System class](#changing-or-making-a-new-system-class)
   * [Stages](#stages)
      * [Stage 'wiring'](#stage-wiring)
      * [Writing new stages](#writing-new-stages)
      * [Specific stages](#specific-stages)
      * [Stage: Raw data](#stage-raw-data)
         * [Using the standard <a href="/systems/rawdata.py">RawData class</a>](#using-the-standard-rawdata-class)
            * [Volatility calculation](#volatility-calculation)
         * [New or modified raw data classes](#new-or-modified-raw-data-classes)
      * [Stage: Rules](#stage-rules)
      * [Trading rules](#trading-rules)
         * [Data and data arguments](#data-and-data-arguments)
      * [The Rules class, and specifying lists of trading rules](#the-rules-class-and-specifying-lists-of-trading-rules)
         * [Creating lists of rules from a configuration object](#creating-lists-of-rules-from-a-configuration-object)
         * [Interactively passing a list of trading rules](#interactively-passing-a-list-of-trading-rules)
         * [Creating variations on a single trading rule](#creating-variations-on-a-single-trading-rule)
         * [Using a newly created Rules() instance](#using-a-newly-created-rules-instance)
         * [Passing trading rules to a pre-baked system function](#passing-trading-rules-to-a-pre-baked-system-function)
         * [Changing the trading rules in a system on the fly (advanced)](#changing-the-trading-rules-in-a-system-on-the-fly-advanced)
      * [Stage: Forecast scale and cap <a href="/systems/forecast_scale_cap.py">ForecastScaleCap class</a>](#stage-forecast-scale-and-cap-forecastscalecap-class)
         * [Using fixed weights (/systems/forecast_scale_cap.py)](#using-fixed-weights-systemsforecast_scale_cappy)
         * [Calculating estimated forecasting scaling on the fly(/systems/forecast_scale_cap.py)](#calculating-estimated-forecasting-scaling-on-the-flysystemsforecast_scale_cappy)
            * [Pooled forecast scale estimate (default)](#pooled-forecast-scale-estimate-default)
            * [Individual instrument forecast scale estimate](#individual-instrument-forecast-scale-estimate)
      * [Stage: Forecast combine <a href="/systems/forecast_combine.py">ForecastCombine class</a>](#stage-forecast-combine-forecastcombine-class)
         * [Using fixed weights and multipliers(/systems/forecast_combine.py)](#using-fixed-weights-and-multiplierssystemsforecast_combinepy)
         * [Using estimated weights and diversification multiplier(/systems/forecast_combine.py)](#using-estimated-weights-and-diversification-multipliersystemsforecast_combinepy)
            * [Estimating the forecast weights](#estimating-the-forecast-weights)
            * [Removing expensive trading rules](#removing-expensive-trading-rules)
            * [Estimating the forecast diversification multiplier](#estimating-the-forecast-diversification-multiplier)
         * [Forecast mapping](#forecast-mapping)
      * [Stage: Position scaling](#stage-position-scaling)
         * [Using the standard <a href="/systems/positionsizing.py">PositionSizing class</a>](#using-the-standard-positionsizing-class)
      * [Stage: Creating portfolios <a href="/systems/portfolio.py">Portfolios class</a>](#stage-creating-portfolios-portfolios-class)
         * [Using fixed weights and instrument diversification multiplier(/systems/portfolio.py)](#using-fixed-weights-and-instrument-diversification-multipliersystemsportfoliopy)
         * [Using estimated weights and instrument diversification multiplier(/systems/portfolio.py)](#using-estimated-weights-and-instrument-diversification-multipliersystemsportfoliopy)
            * [Estimating the instrument weights](#estimating-the-instrument-weights)
            * [Estimating the forecast diversification multiplier](#estimating-the-forecast-diversification-multiplier-1)
         * [Buffering and position inertia](#buffering-and-position-inertia)
         * [Capital correction](#capital-correction)
      * [Stage: Accounting](#stage-accounting)
         * [Using the standard <a href="/systems/accounts/accounts_stage.py">Account class</a>](#using-the-standard-account-class)
         * [accountCurve](#accountcurve)
         * [accountCurveGroup in more detail](#accountcurvegroup-in-more-detail)
         * [A nested accountCurveGroup](#a-nested-accountcurvegroup)
            * [Weighted and unweighted account curve groups](#weighted-and-unweighted-account-curve-groups)
         * [Testing account curves](#testing-account-curves)
         * [Costs](#costs)
* [Processes](#processes)
   * [File names](#file-names)
   * [Logging](#logging)
      * [Basic logging](#basic-logging)
      * [Advanced logging](#advanced-logging)
   * [Optimisation](#optimisation)
      * [The optimisation function, and data](#the-optimisation-function-and-data)
      * [Removing expensive assets (forecast weights only)](#removing-expensive-assets-forecast-weights-only)
      * [Pooling gross returns (forecast weights only)](#pooling-gross-returns-forecast-weights-only)
      * [Working out net costs (both instrument and forecast weights)](#working-out-net-costs-both-instrument-and-forecast-weights)
      * [Time periods](#time-periods)
      * [Moment estimation](#moment-estimation)
      * [Methods](#methods)
         * [Equal weights](#equal-weights)
         * [One period (not recommend)](#one-period-not-recommend)
         * [Bootstrapping (recommended, but slow)](#bootstrapping-recommended-but-slow)
         * [Shrinkage (okay, but tricky to calibrate)](#shrinkage-okay-but-tricky-to-calibrate)
         * [Handcrafting (recommended)](#handcrafting-recommended)
      * [Post processing](#post-processing)
   * [Estimating correlations and diversification multipliers](#estimating-correlations-and-diversification-multipliers)
   * [Capital correction: Varying capital](#capital-correction-varying-capital)
* [Reference](#reference)
   * [Table of standard system.data and system.stage methods](#table-of-standard-systemdata-and-systemstage-methods)
      * [Explanation of columns](#explanation-of-columns)
      * [System object](#system-object)
      * [Data object](#data-object)
      * [<a href="#stage_rawdata">Raw data stage</a>](#raw-data-stage)
      * [<a href="#rules">Trading rules stage (chapter 7 of book)</a>](#trading-rules-stage-chapter-7-of-book)
      * [<a href="#stage_scale">Forecast scaling and capping stage (chapter 7 of book)</a>](#forecast-scaling-and-capping-stage-chapter-7-of-book)
      * [<a href="#stage_combine">Combine forecasts stage (chapter 8 of book)</a>](#combine-forecasts-stage-chapter-8-of-book)
      * [<a href="#position_scale">Position sizing stage (chapters 9 and 10 of book)</a>](#position-sizing-stage-chapters-9-and-10-of-book)
      * [<a href="#stage_portfolio">Portfolio stage (chapter 11 of book)</a>](#portfolio-stage-chapter-11-of-book)
      * [<a href="#accounts_stage">Accounting stage</a>](#accounting-stage)
   * [Configuration options](#configuration-options)
      * [Raw data stage](#raw-data-stage-1)
         * [Volatility calculation](#volatility-calculation-1)
      * [Rules stage](#rules-stage)
         * [Trading rules](#trading-rules-1)
      * [Forecast scaling and capping stage](#forecast-scaling-and-capping-stage)
         * [Forecast scalar (fixed)](#forecast-scalar-fixed)
         * [Forecast scalar (estimated)](#forecast-scalar-estimated)
         * [Forecast cap (fixed - all classes)](#forecast-cap-fixed---all-classes)
      * [Forecast combination stage](#forecast-combination-stage)
         * [Forecast weights (fixed)](#forecast-weights-fixed)
         * [Forecast weights (estimated)](#forecast-weights-estimated)
            * [List of trading rules to get forecasts for](#list-of-trading-rules-to-get-forecasts-for)
            * [Parameters for estimating forecast weights](#parameters-for-estimating-forecast-weights)
         * [Forecast diversification multiplier  (fixed)](#forecast-diversification-multiplier--fixed)
         * [Forecast diversification multiplier  (estimated)](#forecast-diversification-multiplier--estimated)
            * [Forecast mapping](#forecast-mapping-1)
      * [Position sizing stage](#position-sizing-stage)
         * [Capital scaling parameters](#capital-scaling-parameters)
      * [Portfolio combination stage](#portfolio-combination-stage)
         * [Instrument weights (fixed)](#instrument-weights-fixed)
         * [Instrument weights (estimated)](#instrument-weights-estimated)
         * [Instrument diversification multiplier (fixed)](#instrument-diversification-multiplier-fixed)
         * [Instrument diversification multiplier (estimated)](#instrument-diversification-multiplier-estimated)
         * [Buffering](#buffering)
      * [Accounting stage](#accounting-stage-1)
         * [Buffering and position inertia](#buffering-and-position-inertia-1)
         * [Costs](#costs-1)
         * [Capital correction](#capital-correction-1)



Created by [gh-md-toc](https://github.com/ekalinin/github-markdown-toc)



<a name="how_do_i">
</a> -->

# How do I?

   * [How do I.... Experiment with a single trading rule and instrument](#how-do-i-experiment-with-a-single-trading-rule-and-instrument)
   * [How do I....Create a standard futures backtest](#how-do-icreate-a-standard-futures-backtest)
   * [How do I....Create a futures backtest which estimates parameters](#how-do-icreate-a-futures-backtest-which-estimates-parameters)
   * [How do I....See intermediate results from a backtest](#how-do-isee-intermediate-results-from-a-backtest)
   * [How do I....See how profitable a backtest was](#how-do-isee-how-profitable-a-backtest-was)
   * [How do I....Change backtest parameters](#how-do-ichange-backtest-parameters)
   * [How do I....Run a backtest on a different set of instruments](#how-do-irun-a-backtest-on-a-different-set-of-instruments)
   * [How do I....Create my own trading rule](#how-do-icreate-my-own-trading-rule)
   * [How do I....Use different data or instruments](#how-do-iuse-different-data-or-instruments)
   * [How do I... Save my work](#how-do-i-save-my-work)


<a name="guide"> </a>

# Guide


The guide section explains in more detail how each part of the system works:

1. [Data](#data) objects
2. [Config](#config) objects and yaml files
3. [System](#system) objects,
4. [Stages](#stage_general) within a system.

Each section is split into parts that get progressively trickier; varying from
using the standard objects that are supplied up to writing your own.

<a name="data"> </a>

## Data



## Configuration

## System


<a name="Processes"> </a>

# Processes

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


<a name="optimisation"> </a>

## Optimisation




<a name="capcorrection"> </a>

## Capital correction: Varying capital



<a name="reference"> </a>

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
