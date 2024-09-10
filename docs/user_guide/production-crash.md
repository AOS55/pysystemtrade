# Recovering from a crash - what you can save and how, and what you can't

## General advice

Here's some general advice about recovering from a crash:

- If you're not using IBC restart the IB Gateway; and if you are check it has started ok
- Temporarily turn off the crontab to stop processes from spawning before you are ready
- Check you have a mongoDB instance running okay
- Run a full set of reports, and carefully check them, especially the status and reconcile reports, to see that all is well.
- If necessary take steps to recover data (see next section). 
- If this goes well you will have an empty order stack. Run update_strategy_orders to repopulate it.
- You should turn the crontab back on when everything is working fine
- Processes are started by the scheduler, eg Cron, you will need to start them manually if their normal start time has passed (I find [linux screen](https://linuxize.com/post/how-to-use-linux-screen/) helpful for this on my headless server). Everything should work normally the following day.



## Data recovery

Let's first consider an awful case where your mongo DB is corrupted, and the backups are also corrupted. In this case you can use the backed up .csv database dump files to recover the following: FX, individual futures contract prices, multiple prices, adjusted prices, position data, historical trades, capital, contract meta-data, instrument data, optimal positions. Note that scripts don't necessarily exist to do all this automatically yet FIX ME TO DO.

Some other state information relating to the control of trading and processes is also stored in the database and this will be lost, however this can be recovered with a little work: roll status, trade limits, position limits, and overrides. Log data will also be lost; but archived [echo files](#echos-stdout-output) could be searched if necessary.

The better case is when the mongo DB is fine. In this case (once you've [restored](#mongo-data) it) you will have only lost everything from your last nightly backup onwards. Here is what you do to get it back (if possible)

- As the database isn't SQL it's possible for inconsistencies to creep in, so it's generally better to revert to the last good full backup even if some data appears to be up to date
- Log entries will be lost, shrug, deal with it.
- Any changes made to trade limits, position limits and overrides will be lost and will need to be redone.
- You may want to copy across the backtest state files.
- IMPORTANT:Any changes made to roll status will be lost; any back adjusted price rolls will have reverted. Do this before any trading takes place, or you may confuse the system!
- IMPORTANT: The stack handler may contain incomplete orders. Run interactive_order_stack and run the end of day process. Do this before any trading takes place, or you may confuse the system!
- IMPORTANT:Even after finishing the stack handler, position data and historical data will be missing the effect of any trades, including orders that were subsequently filled but for which the fill was lost. Run interactive_order_stack and check to see if view positions. If any breaks come up, you will need to enter create either a balance trade (contract level break between broker and database) or balance instrument trade (instrument level break between strategy and contract positions) using interactive_order_stack. Get the fill prices from your brokerage website. Do this before any trading takes place or the system will lock and won't trade the instruments with breaks.
- FX, individual futures contract prices, multiple prices, adjusted prices: data will be backfilled once run_daily_price_updates has run.
- Capital: any intraday p&l data will be lost, but once run_capital_update has run the current capital will be correct.
- Optimal positions: will be correct once run_systems has run.
- You can use update_*  processes to run skipped processes before the normal scheduled process will do so. Don't forget to run them in the correct order: update_fx_prices (has to be before run_systems), update_sampled_contracts, update_historical_prices, update_multiple_adjusted_prices, update_strategy_backtests
- IMPORTANT: State information about processes running may be wrong; you may need to manually FINISH processes using interactive_controls otherwise processes won't run for fear of conflict (but the startup script should do this for you)