# Scheduling

Running a fully or partially automated trading system requires the use of scheduling software that can launch new scripts at regular intervals, for example:

- processes that kick off when a machine is switched on
- processes that kick off daily
- processes that kick off several times a day (eg regular reports or reconciliation)


## Issues to consider when constructing the schedule

Things to consider when constructing a schedule include:

- Machine load   (eg avoid running computationally intensive processes when also trading where latency could be important). This is less important if you are running multiple machines.
- Database thrashing (eg avoid running input intensive reporting processes on database tables that are being actively read / written to by more important live trading processes)
- File lock / integrity (eg avoid running backups whilst active writes are occurring)
- Robustness (eg it's probably better to have trading processes shutting down each night and then restarting in the morning, than trying to keep them running continuously)


## Choice of scheduling systems

You need some sort of scheduling system to kick off the various top level processes (all scripts that are prefixed with run_). Ideally this would allow us to monitor processes, record their activity, control them remotely, run them a certain number of times, wait for other processes to run first (conditionality) and so on.

### Linux cron

The linux crontab is a thing of beauty, but it can't (easily?) handle things like conditional processes, nor does it do monitoring.

### Third party scheduler

There are plenty of third party schedulers, particular if you are working with something like Docker / Puppet.

### Windows task scheduler

I have not used this product (I don't use Windows or Mac products for ideological reasons, also they're rubbish and overpriced respectively), but in theory it should do the job.

### Python

You can use python itself as a scheduler, using something like [this](https://github.com/dbader/schedule), which gives you the advantage of being platform independent. However you will still need to ensure there is a python instance running all the time. You also need to be careful about whether you are spawning new threads or new processes, since only one connection to IB Gateway or TWS can be launched within a single process.

### Manual system

It's possible to run pysystemtrade without any scheduling, by manually starting the necessary processes as required. This option might make sense for traders who are not running a fully automated system (though [you may want to keep most of the scheduling running anyway](#automation-options)).

### Hybrid of python and cron

This is the approach I use in pysystemtrade, and it's described in more detail below. It ought to be possible to replace the cron component with another scheduler.

## Pysystemtrade scheduling

The scheduler built into pysystemtrade does not launch processes (this is still be done by the cron on a daily basis), but it does everything else:

- Record when processes have started and stopped, if they are still running, and what their process ID is.
- Run only in a specified time window (start time, end time)
- Run only when another process has already close (i.e. do not run_systems until prices have been updated)
- Allow interactive_controls to STOP processes, or prevent them from starting.
- Call 'methods', which are effectively sub processes, multiple times (up to a specified limit) and at specified time intervals (if required).
- Provides a monitoring tool which can also be used from a remote machine

### Configuring the scheduling

#### The crontab

Processes still need to be launched every day, since the pysystemtrade scheduler doesn't do that. However their start time isn't critical, since separate start times can be configured in .yaml files (more of that below).

Because I use cron myself, there are is a [cron tab included in pysystemtrade](https://github.com/robcarver17/pysystemtrade/blob/master/sysproduction/linux/crontab).

Useful things to note about the crontab:

- We start the stack handler and capital update processes. These run 'all day' (you can envisage a situation in which other processes also run all day, if you are running certain kinds of intraday system). They will actually start and then stop when the process configuration (in .yaml) tells them to.
- We then start a bunch of 'once a day' processes: `run_daily_price_updates`, `run_systems`, `run_strategy_order_generator`, `run_cleaners`, `run_backups`, `run_reports`. They are started in the sequence they will run, but their behaviour will actually be governed by the process configuration in .yaml (below)
- On startup we start a mongodb instance, and run the [startup script](#start-up-script)

#### Process configuration

Process configuration is governed by the following config parameters (in [/syscontrol/control_config.yaml](/syscontrol/control_config.yaml), or these will be overridden by /private/private_control_config.yaml):

-  `process_configuration_start_time`: when the process starts (default 00:01)
- `process_configuration_stop_time`: when the process ends, regardless of any method configuration (default 23:50)
- `process_configuration_previous_process`: a process that has to have run in the previous 24 hours for the process to start (default: none)

Each of these is a dict, with process names as keys. All values are strings; start and stop times are in 24 hour format eg '23:05'. If a value is missing for any process, then we use the default. Here's the default .yaml values, with some comments:

```
process_configuration_start_time:
  default: '00:01'
  run_stack_handler: '00:01'
  run_capital_update: '01:00'
  run_daily_prices_updates: '20:00' # we start these off at 5 minute intervals, although the previous process will govern how they actually run
  run_systems: '20:05'
  run_strategy_order_generator: '20:10'
  run_backups: '20:15'
  run_cleaners: '20:20'
  run_reports: '20:25'
process_configuration_stop_time:
  default: '23:50'
  run_strategy_order_generator: '19:30' # this in case we're running it throughout the day
  run_stack_handler: '19:45' # I stop trading late in the US afternoon session to give myself a few hours for daily processes to run
  run_capital_update: '19:50'
  run_daily_prices_updates: '23:50' # these are all nominal stop times
  run_systems: '23:50'
  run_backups: '23:50'
  run_cleaners: '23:50'
  run_reports: '23:50'
process_configuration_previous_process:
  run_systems: 'run_daily_prices_updates' # no point running a backtest with stale prices.
  run_strategy_order_generator: 'run_systems' # will be no orders to generate until backtest system has run
  run_cleaners: 'run_strategy_order_generator' # wait until the main 'big 3' daily processes have run before tidying up
  run_backups: 'run_cleaners' # this can take a while, will be less stuff to back up if we've already cleaned
  run_reports: 'run_strategy_order_generator' # will be more interesting reports if we run after other stuff has close

```

The configuration of methods that run from within each process are governed by the config parameter `process_configuration_methods`. That in turn contains a dict for each relevant process, which in turn has a dict for each method, and these have the following possible values:

- `frequency`: How many minutes pass before we run a method again (default: 0, no waiting time)
- `max_executions`: How many times to run the method (default: -1, which means there is no maximum)
- `run_on_completion_only`: Don't run until the process is stopping

(Why isn't there a 'run on start only' option? Well setting max_executions will do this; and if this method has to come before any others then just list it first in the configuration)

Note for `run_systems` and `run_strategy_order_generator` the methods are actually strategy names, and there are additional parameters that are specific to these processes.

Here is the full default control config with comments:

```
process_configuration_methods:
  run_capital_update:
    update_total_capital: # every 2 hours throughout the day; in a crisis I like to keep an eye on my account value
      frequency: 120
      max_executions: 10 # nominal figure, since uptime is a little less than 20 hours
    strategy_allocation:
      max_executions: 1 # don't bother updating more often than we run backtests
  run_daily_prices_updates: # all this stuff happens once. the order matters.
    update_fx_prices:
      max_executions: 1
    update_sampled_contracts:
      max_executions: 1
    update_historical_prices:
      max_executions: 1
    update_multiple_adjusted_prices:
      max_executions: 1
  run_stack_handler: # frequency 0 and max_executions -1 means we just keep doing them over and over again until the process stops...
    refresh_additional_sampling_all_instruments:
      frequency: 60
      max_executions: -1
    check_external_position_break:
      frequency: 0
      max_executions: -1
    spawn_children_from_new_instrument_orders:
      frequency: 0
      max_executions: -1
    generate_force_roll_orders:
      frequency: 0
      max_executions: 1
    create_broker_orders_from_contract_orders:
      frequency: 0
      max_executions: -1
    process_fills_stack:
      frequency: 0
      max_executions: -1
    handle_completed_orders:
      frequency: 0
      max_executions: -1
    safe_stack_removal:
      run_on_completion_only: True   # only run this once we're done
  run_reports:  # all this stuff happens once.
    costs_report:
      max_executions: 1
    liquidity_report:
      max_executions: 1
    status_report:
      max_executions: 1
    roll_report:
      max_executions: 1
    daily_pandl_report:
      max_executions: 1
    reconcile_report:
      max_executions: 1
    trade_report:
      max_executions: 1
  run_backups:
    backup_arctic_to_csv:
      max_executions: 1
    backup_files:
      max_executions: 1
    backup_mongo_data_as_dump:
      max_executions: 1
  run_cleaners:  # all this stuff happens once.
    clean_backtest_states:
      max_executions: 1
    clean_echo_files:
      max_executions: 1
    clean_log_files:
      max_executions: 1
```

You can override any of these in /private/private_control_config.yaml, but you *must* also include the following sections in your private control config file (add more if you have more strategies), or these run processes won't work:

```
process_configuration_methods:
  run_systems:
    example:  # strategy name
      max_executions: 1
      object: sysproduction.strategy_code.run_system_classic.runSystemClassic # additional parameter
      backtest_config_filename: systems.provided.futures_chapter15.futures_config.yaml #additional parameter
  run_strategy_order_generator:
    example: # strategy_name
      object: sysexecution.strategies.classic_buffered_positions.orderGeneratorForBufferedPositions # additional parameter passed
      max_executions: 1
```

Finally you can optionally include arguments, which will be passed to certain methods within a process (or to all methods that are run on completion for a given process). For example:

```
arguments:
  run_daily_prices_updates:# name of process
    update_historical_prices: # everything in this block is passed as **kwargs to this method
      download_by_zone:
        ASIA: '07:00'
        EMEA: '18:00'
        US: '20:00'
    _methods_on_completion: # and this block is passed to all methods that run on completion only - make sure you use **kwargs to trap if required
        a: 'test'

```


### System monitor and dashboard

There is a crude monitoring tool, and a more sophisticated fancy dashboard, which you can use to monitor what the system is up to. [Read the doc file here.](/docs/dashboard_and_monitor.md)


### Troubleshooting?

Use status report and interactive_controls to investigate.

Why won't my process run?

- is it launching in the cron or equivalent scheduler?
- is it set to STOP or DONT RUN? Fix with interactive_controls
- is it before the start_time? Change the start time, or wait
- is it after the end_time? Change the end time, or wait until tomorrow
- has the previous process close? Wait, or remove dependency
- is it already running, or at least thinks it is already running because a previous iteration didn't fail? Mark the process as close with interactive_controls

Why has my process stopped?

- is it set to STOP?
- is it after the end_time?
- have all the methods close running, because they have exceeded their `max_executions`?

Why won't my method run?

- has it run out of `max_executions`?
- is it set to `run_on_completion_only`?