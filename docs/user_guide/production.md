This document is specifically about using pysystemtrade for *live production trading*.

This includes:

1. Getting prices
2. Generating desired trades
3. Executing trades
4. Getting accounting information

Related documents (which you should read before this one!):

- [Backtesting with pysystemtrade](/docs/backtesting.md)
- [Storing futures and spot FX data](/docs/data.md)
- [Connecting pysystemtrade to interactive brokers](/docs/IB.md)

And documents you should read after this one:

- [Instruments](/docs/instruments.md)
- [Dashboard and monitor](/docs/dashboard_and_monitor.md)
- [Production strategy changes](/docs/production_strategy_changes.md)
- [Recent undocumented changes](/docs/recent_changes.md)

*IMPORTANT: Make sure you know what you are doing. All financial trading offers the possibility of loss. Leveraged trading, such as futures trading, may result in you losing all your money, and still owing more. Backtested results are no guarantee of future performance. No warranty is offered or implied for this software. I can take no responsibility for any losses caused by live trading using pysystemtrade. Use at your own risk.*


Created by [gh-md-toc](https://github.com/ekalinin/github-markdown-toc)

Table of Contents
=================

- [Table of Contents](#table-of-contents)
- [Scripts](#scripts)
  - [Script calling](#script-calling)
  - [Script naming convention](#script-naming-convention)
  - [Run processes](#run-processes)
  - [Core production system components](#core-production-system-components)
    - [Get spot FX data from interactive brokers, write to MongoDB (Daily)](#get-spot-fx-data-from-interactive-brokers-write-to-mongodb-daily)
    - [Update sampled contracts (Daily)](#update-sampled-contracts-daily)
    - [Update futures contract historical price data (Daily)](#update-futures-contract-historical-price-data-daily)
      - [Set times when different regions download prices](#set-times-when-different-regions-download-prices)
    - [Update multiple and adjusted prices (Daily)](#update-multiple-and-adjusted-prices-daily)
    - [Update capital and p\&l by polling brokerage account](#update-capital-and-pl-by-polling-brokerage-account)
    - [Allocate capital to strategies](#allocate-capital-to-strategies)
    - [Run updated backtest systems for one or more strategies](#run-updated-backtest-systems-for-one-or-more-strategies)
    - [Generate orders for each strategy](#generate-orders-for-each-strategy)
    - [Execute orders](#execute-orders)
  - [Interactive scripts to modify data](#interactive-scripts-to-modify-data)
    - [Manual check of futures contract historical price data](#manual-check-of-futures-contract-historical-price-data)
    - [Manual check of FX price data](#manual-check-of-fx-price-data)
    - [Interactively modify capital values](#interactively-modify-capital-values)
    - [Interactively roll adjusted prices](#interactively-roll-adjusted-prices)
      - [Manually input instrument codes and manually decide when to roll](#manually-input-instrument-codes-and-manually-decide-when-to-roll)
      - [Cycle through instrument codes automatically, but manually decide when to roll](#cycle-through-instrument-codes-automatically-but-manually-decide-when-to-roll)
      - [Cycle through instrument codes automatically, auto decide when to roll, manually confirm rolls](#cycle-through-instrument-codes-automatically-auto-decide-when-to-roll-manually-confirm-rolls)
      - [Cycle through instrument codes automatically, auto decide when to roll, automatically roll](#cycle-through-instrument-codes-automatically-auto-decide-when-to-roll-automatically-roll)
  - [Menu driven interactive scripts](#menu-driven-interactive-scripts)
    - [Interactive controls](#interactive-controls)
      - [Trade limits](#trade-limits)
      - [Position limits](#position-limits)
      - [Trade control / override](#trade-control--override)
      - [Broker client IDs](#broker-client-ids)
      - [Process control \& monitoring](#process-control--monitoring)
        - [View processes](#view-processes)
        - [Change status of process](#change-status-of-process)
        - [Global status change](#global-status-change)
        - [Mark as close](#mark-as-close)
        - [Mark all dead processes as close](#mark-all-dead-processes-as-close)
        - [View process configuration](#view-process-configuration)
      - [Update configuration](#update-configuration)
    - [Interactive diagnostics](#interactive-diagnostics)
      - [Backtest objects](#backtest-objects)
        - [Output choice](#output-choice)
        - [Choice of strategy and backtest](#choice-of-strategy-and-backtest)
        - [Choose stage / method / arguments](#choose-stage--method--arguments)
        - [Alternative python code](#alternative-python-code)
      - [Reports](#reports)
      - [Logs, errors, emails](#logs-errors-emails)
        - [View stored emails](#view-stored-emails)
      - [View prices](#view-prices)
      - [View capital](#view-capital)
      - [Positions and orders](#positions-and-orders)
      - [Instrument configuration](#instrument-configuration)
        - [View instrument configuration data](#view-instrument-configuration-data)
        - [View contract configuration data](#view-contract-configuration-data)
    - [Interactive order stack](#interactive-order-stack)
      - [View](#view)
      - [Create orders](#create-orders)
        - [Spawn contract orders from instrument orders](#spawn-contract-orders-from-instrument-orders)
        - [Create force roll contract orders](#create-force-roll-contract-orders)
        - [Create (and try to execute...) IB broker orders](#create-and-try-to-execute-ib-broker-orders)
        - [Balance trade: Create a series of trades and immediately fill them (not actually executed)](#balance-trade-create-a-series-of-trades-and-immediately-fill-them-not-actually-executed)
        - [Balance instrument trade: Create a trade just at the strategy level and fill (not actually executed)](#balance-instrument-trade-create-a-trade-just-at-the-strategy-level-and-fill-not-actually-executed)
        - [Manual trade: Create a series of trades to be executed](#manual-trade-create-a-series-of-trades-to-be-executed)
        - [Cash FX trade](#cash-fx-trade)
      - [Netting, cancellation and locks](#netting-cancellation-and-locks)
        - [Cancel broker order](#cancel-broker-order)
        - [Net instrument orders](#net-instrument-orders)
        - [Lock/unlock order](#lockunlock-order)
        - [Lock/unlock instrument code](#lockunlock-instrument-code)
        - [Unlock all instruments](#unlock-all-instruments)
        - [Remove Algo lock on contract order](#remove-algo-lock-on-contract-order)
      - [Delete and clean](#delete-and-clean)
        - [Delete entire stack (CAREFUL!)](#delete-entire-stack-careful)
        - [Delete specific order ID (CAREFUL!)](#delete-specific-order-id-careful)
        - [End of day process (cancel orders, mark all orders as complete, delete orders)](#end-of-day-process-cancel-orders-mark-all-orders-as-complete-delete-orders)
  - [Reporting, housekeeping and backup scripts](#reporting-housekeeping-and-backup-scripts)
    - [Run all reports](#run-all-reports)
    - [Delete old pickled backtest state objects](#delete-old-pickled-backtest-state-objects)
    - [Clean up old logs](#clean-up-old-logs)
    - [Truncate echo files](#truncate-echo-files)
    - [Backup Arctic data to .csv files](#backup-arctic-data-to-csv-files)
    - [Backup state files](#backup-state-files)
    - [Backup mongo dump](#backup-mongo-dump)
    - [Start up script](#start-up-script)
  - [Scripts under other (non-linux) operating systems](#scripts-under-other-non-linux-operating-systems)
- [Scheduling](#scheduling)
  - [Issues to consider when constructing the schedule](#issues-to-consider-when-constructing-the-schedule)
  - [Choice of scheduling systems](#choice-of-scheduling-systems)
    - [Linux cron](#linux-cron)
    - [Third party scheduler](#third-party-scheduler)
    - [Windows task scheduler](#windows-task-scheduler)
    - [Python](#python)
    - [Manual system](#manual-system)
    - [Hybrid of python and cron](#hybrid-of-python-and-cron)
  - [Pysystemtrade scheduling](#pysystemtrade-scheduling)
    - [Configuring the scheduling](#configuring-the-scheduling)
      - [The crontab](#the-crontab)
      - [Process configuration](#process-configuration)
    - [System monitor and dashboard](#system-monitor-and-dashboard)
    - [Troubleshooting?](#troubleshooting)
- [Production system concepts](#production-system-concepts)
  - [Configuration files](#configuration-files)
    - [System defaults \& Private config](#system-defaults--private-config)
    - [System backtest .yaml config file(s)](#system-backtest-yaml-config-files)
    - [Control config files](#control-config-files)
    - [Broker and data source specific configuration files](#broker-and-data-source-specific-configuration-files)
    - [Instrument and roll configuration](#instrument-and-roll-configuration)
    - [Set up configuration](#set-up-configuration)
  - [Capital](#capital)
    - [Large changes in capital](#large-changes-in-capital)
    - [Withdrawals and deposits of cash or stock](#withdrawals-and-deposits-of-cash-or-stock)
    - [Change in capital methodology or capital base](#change-in-capital-methodology-or-capital-base)
  - [Strategies](#strategies)
    - [Strategy capital](#strategy-capital)
      - [Risk target](#risk-target)
      - [Changing risk targets and/or capital](#changing-risk-targets-andor-capital)
    - [System runner](#system-runner)
    - [Strategy order generator](#strategy-order-generator)
    - [Load backtests](#load-backtests)
    - [Reporting code](#reporting-code)
- [Recovering from a crash - what you can save and how, and what you can't](#recovering-from-a-crash---what-you-can-save-and-how-and-what-you-cant)
  - [General advice](#general-advice)
  - [Data recovery](#data-recovery)
- [Reports](#reports-1)
  - [Dashboard](#dashboard)
    - [Roll report (Daily)](#roll-report-daily)
    - [P\&L report](#pl-report)
    - [Status report](#status-report)
    - [Trade report](#trade-report)
    - [Reconcile report](#reconcile-report)
    - [Strategy report](#strategy-report)
    - [Risk report](#risk-report)
    - [Liquidity report](#liquidity-report)
    - [Costs report](#costs-report)
  - [Customize report generation in the run\_report process](#customize-report-generation-in-the-run_report-process)

Created by [gh-md-toc](https://github.com/ekalinin/github-markdown-toc)
























