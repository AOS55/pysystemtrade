# Data

A data object is used to provide data to a system. Data objects are designed to work with a specific **type** of data, typically based on the asset class (e.g., futures), and from a specific **source** (e.g., CSV files, databases, etc.).

## Using the Standard Data Objects

PySystemTrade provides two types of specific data objects in the current version:

- `csvFuturesSimData` for .csv files
- `dbFuturesSimData` for database storage

For working with futures data, refer to [this guide](data.md).

### Generic Data Objects

You can directly import and use data objects as follows (using `csvFuturesSimData` as an example):

```python
from sysdata.sim.csv_futures_sim_data import csvFuturesSimData

data = csvFuturesSimData()

# Getting data out
data.methods()  # List of methods

data.get_raw_price(instrument_code)
data[instrument_code]  # Same as get_raw_price

data.get_instrument_list()
data.keys()  # Also gets the instrument list

data.get_value_of_block_price_move(instrument_code)
data.get_instrument_currency(instrument_code)
data.get_fx_for_instrument(instrument_code, base_currency)  # Get fx rate between instrument currency and base currency
```

You can also use data objects within a system:

```python
## Using with a system
from systems.provided.futures_chapter15.basesystem import futures_system
system = futures_system(data=data)

system.data.get_instrument_currency(instrument_code) # And so on
```

(Note: When specifying a data item within a trading [rule](#rules), omit the system, e.g., `data.get_raw_price`)

If you set the `start_date` configuration option, only a subset of the data will be shown:

```python
## Using with a system
from systems.provided.futures_chapter15.basesystem import futures_system
system = futures_system(data=data)

# We could also do this in the .yaml file. Note that the formatting used must be the same
system.config.start_date = '2000-01-19'

## Or as a datetime (won't work in yaml obviously)
import datetime
system.config.start_date = datetime.datetime(2000,1,19)
```

<a name="csvdata"> </a>

#### The csvFuturesSimData object

The `csvFuturesSimData` object works like this:

```python
from sysdata.sim.csv_futures_sim_data import csvFuturesSimData

## with the default folders
data=csvFuturesSimData()

## OR with different folders, by providing a dict containing the folder(s) to use
data=csvFuturesSimData(csv_data_paths = dict(key_name = "pathtodata.with.dots"))

# Permissible key names are 'csvFxPricesData' (FX prices), 'csvFuturesMultiplePricesData' 
# (for carry and forward prices),
# 'csvFuturesAdjustedPricesData' and 'csvFuturesInstrumentData' (configuration and costs).
# If a keyname is not present then the system defaults will be used

# An example to override with FX data stored in /psystemtrade/private/data/fxdata/:

data=csvFuturesSimData(csv_data_paths = dict(csvFxPricesData="private.data.fxdata"))

# WARNING: Do not store multiple_price_data and adjusted_price_data in the same directory
#          They use the same file names!

## getting data out
data.methods() ## will list any extra methods
data.get_instrument_raw_carry_data(instrument_code) ## specific data for futures

## using with a system
from systems.provided.futures_chapter15.basesystem import futures_system
system=futures_system(data=data)
system.data.get_instrument_raw_carry_data(instrument_code)
```

Each relevant pathname must contain .csv files of the following four types (where code is
the instrument_code):

1. Static configuration and cost data- `instrument_config.csv` headings: Instrument, Pointsize,
  AssetClass, Currency. Additional headings for costs: Slippage, PerBlock,
  Percentage, PerTrade. See ['costs'](#costs) for more detail.
2. Roll parameters data. See [storing futures and spot FX data](data.md) for more detail.
3. Adjusted price data- `code.csv` (eg SP500.csv) headings: DATETIME, PRICE
4. Carry and forward data - `code.csv` (eg AEX.csv): headings:
  DATETIME, PRICE,CARRY,FORWARD,CARRY_CONTRACT PRICE_CONTRACT, FORWARD_CONTRACT
5. Currency data - `ccy1ccy2fx.csv` (eg AUDUSDfx.csv) headings: DATETIME,
  FXRATE

DATETIME should be something that `pandas.to_datetime` can parse. Note that the
price in (2) is the continuously stitched price (see [volatility
calculation](#vol_calc) ), whereas the price column in (3) is the price of the
contract we're currently trading.

At a minimum we need to have a currency file for each instrument's currency
against the default (defined as "USD"); and for the currency of the account
we're trading in (i.e. for a UK investor you'd need a `GBPUSDfx.csv` file). If
cross rate files are available they will be used; otherwise the USD rates will
be used to work out implied cross rates.

See data in subdirectories [pysystemtrade/data/futures](/data/futures) for files you can modify:

<!-- Needs to be the external link to directory on master -->
- [adjusted prices](/data/futures/adjusted_prices_csv),
- [configuration and costs](/data/futures/csvconfig),
- [Futures specific carry and forward prices](/data/futures/multiple_prices_csv)
- [Spot FX prices](/data/futures/fx_prices_csv)

For more information see the [futures data document](/docs/data.md#csvFuturesSimData).

<a name="arctic_data"> </a>

#### The arcticSimData object

This is a simData object which gets its data out of [Mongo DB](https://mongodb.com) (static) and [Arctic](https://github.com/manahl/arctic) (time series) (*Yes the class name should include both terms. Yes I shortened it so it isn't ridiculously long, and most of the interesting stuff comes from Arctic*). It is better for live trading.

For production code, and storing large amounts of data (eg for individual futures contracts) we probably need something more robust than .csv files.
[MongoDB](https://mongodb.com) is a no-sql database which is rather fashionable at the moment, though the main reason I selected it for this purpose is that it is used by Arctic. [Arctic](https://github.com/manahl/arctic) is a superb open source time series database which sits on top of Mongo DB) and provides straightforward and fast storage of pandas DataFrames. It was created by my former colleagues at [Man AHL](https://www.ahl.com/) (in fact I beta tested a very early version of Arctic), and then very generously released as open source.

There is more detail on this in the [futures data documentation](/docs/data.md): [Mongo DB](/docs/data.md#mongoDB) and [Arctic](/docs/data.md#arctic).

##### Setting up your Arctic and Mongo DB databases

Obviously you will need to make sure you already have a Mongo DB instance running. You might find you already have one running, in Linux use `ps wuax | grep mongo` and then kill the relevant process. You also need to get [Arctic](https://github.com/manahl/arctic).

Because the mongoDB data isn't included in the github repo, before using this you need to write the required data into Mongo and Arctic.
You can do this from scratch, as per the ['futures data workflow'](/docs/data.md#a-futures-data-workflow). Alternatively you can run the following scripts which will copy the data from the existing github .csv files:

- [Instrument configuration and cost data](/sysinit/futures/repocsv_instrument_config.py)
- [Adjusted prices](/sysinit/futures/repocsv_adjusted_prices.py)
- [Multiple prices](/sysinit/futures/repocsv_multiple_prices.py)
- [Spot FX prices](/sysinit/futures/repocsv_spotfx_prices.py)

Of course it's also possible to mix these two methods.

##### Using dbFuturesSimData

Once you have the data it's just a matter of replacing the default csv data object:

```python
from systems.provided.futures_chapter15.basesystem import futures_system
from sysdata.sim.db_futures_sim_data import dbFuturesSimData

# with the default database
data = dbFuturesSimData()

# using with a system
system = futures_system()
print(system.accounts.portfolio().sharpe())
```

### Creating your own data objects

You should be familiar with the python object orientated idiom before reading
this section.

The [`simData()`](/sysdata/sim/sim_data.py) object is the base class for data used in simulations. From that we
inherit data type specific classes such as those
[for futures](/sysdata/sim/futures_sim_data.py) object. These in turn are inherited from
for specific data sources, such as for csv files: [csvFuturesSimData()](/sysdata/sim/csv_futures_sim_data.py).

It is helpful if this naming scheme was adhered to: sourceTypeSimData. For example if we had
some single equity data stored in a database we'd do `class
EquitiesSimData(simData)`, and `class dbEquitiesSimData(EquitiesSimData)`.

So, you should consider whether you need a new type of data, a new source of
data or both. You may also wish to extend an existing class. For example if you
wished to add some fundamental data for futures you might define: `class
fundamentalFuturesSimData(futuresSimData)`. You'd then need to inherit from that for a
specific source.

This might seem a hassle, and it's tempting to skip and just inherit from
`simData()` directly, however once your system is up and running it is very
convenient to have the possibility of multiple data sources and this process
ensures they keep a consistent API for a given data type.

It's worth reading the [documentation on futures data](/docs/data.md#modify_SimData) to understand how [csvFuturesSimData()](/sysdata/sim/csv_futures_sim_data.py) is constructed before modifying it or creating your own data objects.

#### The Data() class

Methods that you'll probably want to override:

- `get_raw_price` Returns Tx1 pandas data frame
- `get_instrument_list` Returns list of str
- `get_value_of_block_price_move` Returns float
- `get_raw_cost_data` Returns a dict cost data
- `get_instrument_currency`: Returns str
- `_get_fx_data(currency1, currency2)` Returns Tx1 pandas data frame of
  exchange rates
- 'get_rolls_per_year': returns int

You should not override `get_fx_for_instrument`, or any of the other private fx
related methods. Once you've created a `_get_fx_data method`, then the methods
in the `Data` base class will interact to give the correct fx rate when
external objects call `get_fx_for_instrument()`; handling cross rates and
working them out as needed.

Neither should you override 'daily_prices'.

Finally data methods should not do any caching. [Caching](#caching) is done
within the system class.

