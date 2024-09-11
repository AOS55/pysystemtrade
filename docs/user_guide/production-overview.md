This section is specifically about using PySystemTrade for *live production trading*.

This includes:

1. Getting prices
2. Generating desired trades
3. Executing trades
4. Getting accounting information

Related documents (which you should read before this one!):

- [Backtesting with pysystemtrade](backtesting-overview.md)
- [Storing futures and spot FX data](data.md)
- [Connecting pysystemtrade to interactive brokers](IB.md)

And documents you should read after this one:

- [Instruments](instruments.md)
- [Dashboard and monitor](dashboard_and_monitor.md)
- [Production strategy changes](production_strategy_changes.md)
- [Recent undocumented changes](recent_changes.md)

!!! Warning
    Make sure you know what you are doing. All financial trading offers the possibility of loss. Leveraged trading, such as futures trading, may result in you losing all your money, and still owing more. Backtested results are no guarantee of future performance. No warranty is offered or implied for this software. I can take no responsibility for any losses caused by live trading using pysystemtrade. Use at your own risk.


## Production system data flow

*[Update FX prices]()*
- Input: IB fx prices
- Output: Spot FX prices

*[Update roll adjusted prices](#get-spot-fx-data-from-interactive-brokers-write-to-mongodb-daily)*
- Input: Manual decision, existing multiple price series
- Output: Current set of active contracts (price, carry, forward), Roll calendar (implicit in updated multiple price series)

*[Update sampled contracts](#update-sampled-contracts-daily)*
- Input: Current set of active contracts (price, carry, forward) implicit in multiple price series
- Output: Contracts to be sampled by historical data

*[Update historical prices](#update-futures-contract-historical-price-data-daily)*
- Input: Contracts to be sampled by historical data, IB futures prices
- Output: Futures prices per contract

*[Update multiple adjusted prices](#update-multiple-and-adjusted-prices-daily)*
- Input: Futures prices per contract, Existing multiple price series, Existing adjusted price series
- Output: Adjusted price series, Multiple price series

*[Update account values](#update-capital-and-pl-by-polling-brokerage-account)*
- Input: Brokerage account value from IB
- Output: Total capital. Account level p&l

*[Update strategy capital](#allocate-capital-to-strategies)*
- Input: Total capital
- Output: Capital allocated per strategy

*[Update system backtests](#run-updated-backtest-systems-for-one-or-more-strategies)*
- Input: Capital allocated per strategy, Adjusted futures prices, Multiple price series, Spot FX prices
- Output: Optimal positions and buffers per strategy, pickled backtest state

*[Update strategy orders](#generate-orders-for-each-strategy)*
- Input:  Optimal positions and buffers per strategy
- Output: Instrument orders

*[Run stack handler](#execute-orders)*
- Input: Instrument orders
- Output: Trades, historic order updates, position updates