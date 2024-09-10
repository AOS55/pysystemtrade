# Quick start guide

This *quick* start guide makes the following assumptions:

- you are running on a linux box with an appropriate distro (I use Mint). Windows / Mac people will have to do some things slightly differently
- you are using interactive brokers
- you are storing data using mongodb
- you have a backtest that you are happy with
- you are happy to store your data and configuration in the /private directory of your pysystemtrade installation

You need to:

- Read this document very thoroughly!
- Prerequisites:
    - Install [git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git), install or update [python3](https://docs.python-guide.org/starting/install3/linux/). You may also find a simple text editor (like emacs) is useful for fine tuning, and if you are using a headless server then [x11vnc](http://www.karlrunge.com/x11vnc/) is helpful.
    - Add the following environment variables to your `~/.profile`: (feel free to use other directories):
        - MONGO_DATA=/home/user_name/data/mongodb/
        - PYSYS_CODE=/home/user_name/pysystemtrade
        - SCRIPT_PATH=/home/user_name/pysystemtrade/sysproduction/linux/scripts
        - ECHO_PATH=/home/user_name/echos
        - MONGO_BACKUP_PATH=/media/shared_network/drive/mongo_backup
    - Add the SCRIPT_PATH directory to your PATH
    - Create the following directories (again use other directories if you like, but you must modify the .profile above and specify the proper directories in 'private_config.yaml')
        - '/home/user_name/data/mongodb/'
        - '/home/user_name/echos/'
        - '/home/user_name/data/mongo_dump'
        - '/home/user_name/data/backups_csv'
        - '/home/user_name/data/backtests'
        - '/home/user_name/data/reports'
    - Install the pysystemtrade package, and install or update, any dependencies in directory $PYSYS_CODE (it's possible to put it elsewhere, but you will need to modify the environment variables listed above). If using git clone from your home directory this should create the directory '/home/user_name/pysystemtrade/'
    - [Set up interactive brokers](/docs/IB.md), download and install their python code, and get a gateway running.
    - [Install mongodb](https://docs.mongodb.com/manual/administration/install-on-linux/). Latest v4 is recommended, as [Arctic doesn't support 5](https://github.com/man-group/arctic#requirements) yet
    - create a file 'private_config.yaml' in the private directory of [pysystemtrade](/private), and optionally a ['private_control_config.yaml' file in the same directory](#process-configuration) See [here for more details](#system-defaults--private-config)
    - [check a mongodb server is running with the right data directory](/docs/data.md#mongo-db) command line: `mongod --dbpath $MONGO_DATA`
    - launch an IB gateway (this could be done automatically depending on your security setup)
- FX data:
    - [Initialise the spot FX data in MongoDB from .csv files](/sysinit/futures/repocsv_spotfx_prices.py) (this will be out of date, but you will update it in a moment)
    - Update the FX price data in MongoDB using interactive brokers: command line:`. /home/your_user_name/pysystemtrade/sysproduction/linux/scripts/update_fx_prices`
- Instrument configuration:
    - Set up futures instrument spread costs using this script [repocsv_spread_costs.py](/sysinit/futures/repocsv_spread_costs.py).
- Futures contract prices:
    - [You must have a source of individual futures prices, then backfill them into the Arctic database](/docs/data.md#get_historical_data).
- Roll calendars:
    - [Create roll calendars for each instrument you are trading](/docs/data.md#roll-calendars)
- [Ensure you are sampling all the contracts you want to sample](#update-sampled-contracts-daily)
- Adjusted futures prices:
    - [Create 'multiple prices' in Arctic](/docs/data.md#creating-and-storing-multiple-prices).
    - [Create adjusted prices in Arctic](/docs/data.md#creating-and-storing-back-adjusted-prices)
- Use [interactive diagnostics](#interactive-diagnostics) to check all your prices are in place correctly
- Live production backtest:
    - Create a yaml config file to run the live production 'backtest'. For speed I recommend you do not estimate parameters, but use fixed parameters, using the [yaml_config_with_estimated_parameters method of systemDiag](/systems/diagoutput.py) function to output these to a .yaml file.
- Scheduling:
    - Initialise the [supplied crontab](/sysproduction/linux/crontab). Note if you have put your code or echos somewhere else you will need to modify the directory references at the top of the crontab.
    - All scripts executable by the crontab need to be executable, so do the following: `cd $SCRIPT_PATH` ; `sudo chmod +x *.*`
- Consider adding [position and trade limits](#interactive-controls)
- Review the [configuration options](#configuration-files) available


Before trading, and each time you restart the machine you should:

- [check a mongodb server is running with the right data directory](/docs/data.md#mongo-db) command line: `mongod --dbpath $MONGO_DATA` (the supplied crontab should do this)
- launch an IB gateway (this could [be done automatically](https://github.com/IbcAlpha/IBC) depending on your security setup)
- ensure all processes are [marked as 'close'](#mark-as-finished)

Note that the system won't start trading until the next day, unless you manually launch the processes that would ordinarily have been started by the crontab or other [scheduler](#scheduling). [Linux screen](https://linuxize.com/post/how-to-use-linux-screen/) is helpful if you want to launch a process but not keep the window active (eg on a headless machine).

Also see [this](#recovering-from-a-crash---what-you-can-save-and-how-and-what-you-cant) on recovering from a crash (a system crash that is, not a market crash. You're on your own with the latter).

When trading you will need to do the following:

- Check [reports ](#reports-1)
- [Roll instruments](#interactively-roll-adjusted-prices)
- Ad-hoc [diagnostics and controls](#menu-driven-interactive-scripts)
- [Manually check](#interactive-scripts-to-modify-data) large price changes


# Production system data flow

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