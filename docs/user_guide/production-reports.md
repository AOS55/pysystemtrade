# Reports

## Dashboard

A lot of the information in the reports described below can also be found in the [Web Dashboard](/docs/dashboard_and_monitor.md)

### Roll report (Daily)

The roll report can be run for all markets (default for the email), or for a single individual market (if run on an ad hoc basis). It will also be run when you run the interactive [update roll status](#menu-driven-interactive-scripts) process for the relevant market. Here's an example of a roll report, which I've annotated with comments (marked with quotes ""):

```
********************************************************************************
           Roll status report produced on 2020-10-19 17:10:13.280422            
********************************************************************************

"The roll report gives you all the information you need to decide when to roll from one futures contract to the next"


============================================================================================================================================
                                                      Status and time to roll in days                                                       
============================================================================================================================================

                        status roll_expiry price_expiry carry_expiry contract_priced contract_fwd position_priced volume_priced   volume_fwd
EDOLLAR                Passive        -143          957          866        20240600     20240900               2             1     0.853933
GAS_US_mini            No_Roll         -14           21           55        20211200     20220100              -2             1    0.0949909
BRENT-LAST             Passive         -13           27           -5        20220100     20220200               1             1    0.0907098

Roll_exp is days until preferred roll set by roll parameters. Prc_exp is days until price contract expires, Crry_exp is days until carry contract expires

"When should you roll? Certainly before the current priced contract (what we're currently trading) expires
(note for some contracts, eg fixed income, you should roll before the first notice date).
If the carry contract is younger (as here) then you will probably want to roll before that expires,
assuming that there is enough liquidity, or carry calculations will become stale.
Suggested times to roll before an expiry are shown, and these are used in the backtest to generate
historical roll dates, but you do not need to treat these as gospel in live trading"

"contract priced/contract forward This shows the contracts we're currently primarily trading (price), and will trade next (forward)"

"The position we have in the priced contract"

Contract volumes over recent days, normalised so largest volume is 1.0

"You can't roll until there is sufficient volume in the forward contract. Often a sign that
volume is falling in the price relative to the forward is a sign you should hurry up and roll!
Volumes are shown in relative terms to make interpretation easier."

********************************************************************************
                               END OF ROLL REPORT                               
********************************************************************************

```

### P&L report
    
The p&l report shows you profit and loss (duh!).  On a daily basis it is run for the previous 24 hours. On an ad hoc basis, it can be run for any time period (recent or in the past).

Here is an example, with annotations added in quotes (""):

```

********************************************************************************
P&L report produced on 2020-10-20 09:50:44.037739 from 2020-06-01 00:00:00 to 2020-10-20 09:17:16.470039
********************************************************************************

"Total p&l is what you'd expect. This comes from comparing broker valuations from the two relevant snapshot times. "

Total p&l is -2.746%


"P&L by instrument as a % of total capital. Calculated from database prices and trades.
 There is a bug in my live cattle price somewhere!"

====================================
P&L by instrument for all strategies
====================================

      codes  pandl
0   LIVECOW -18.10
1   SOYBEAN  -4.08
2      CORN  -1.34
3   EUROSTX  -0.81

".... truncated"

19      OAT   2.29
20      BTP   3.88
21    WHEAT   5.03

"If we add up our futures P&L and compare to the total p&l, we get a residual.
This could be because of a bug (as here), but also fees and interest charges,
or non futures instruments which aren't captured by the instrument p&l,
or because of a difference in timing between the broker account valuation and the relevant prices."

Total futures p&l is -12.916%
Residual p&l is 10.171%

===============================
        P&L by strategy        
===============================

"P&L versus total capital, not the capital for the specific strategy. So these should all add up to total p&l"

                   codes  pandl
0  medium_speed_TF_carry -13.42
1               ETFHedge  -0.63
2  _ROLL_PSEUDO_STRATEGY   0.00
0               residual  11.31


==================
P&L by asset class
==================

    codes  pandl
0     Ags -18.51
1  Equity  -0.81
2     Vol  -0.36
3  OilGas  -0.24
4    STIR  -0.11
5  Metals   0.16
6      FX   1.15
7    Bond   5.81


********************************************************************************
                               END OF P&L REPORT                                
********************************************************************************

```

### Status report

The status report monitors the status of processes and data acquisition, plus all control elements. It is run on a daily basis, but can also be run ad hoc. Here is an example report, with annotations in quotes(""):

```

********************************************************************************
              Status report produced on 2020-10-19 23:02:57.321674              
********************************************************************************

"A process is called by the scheduler, eg crontab. Processes have start/end times,
and can also have pre-requisite processes that need to have been run recently.
This provides a quick snapshot to show if the system is running normally"

===============================================================================================================================================================================================================================
                                                                                                     Status of processses                                                                                                      
===============================================================================================================================================================================================================================

name                 run_capital_update run_daily_prices_updates             run_stack_handler                   run_reports   run_backups                  run_cleaners               run_systems run_strategy_order_generator
running                           False                    False                         False                          True          True                         False                      True                        False
start                       10/19 01:00              10/19 20:05                   10/19 00:30                   10/19 23:00   10/19 22:20                   10/19 22:10               10/19 22:05                  10/19 21:55
end                         10/19 19:08              10/19 22:05                   10/19 19:30                   10/16 23:54   10/16 23:14                   10/19 22:10               10/16 22:55                  10/19 21:57
status                               GO                       GO                            GO                            GO            GO                            GO                        GO                           GO
finished_in_last_day               True                     True                          True                         False         False                          True                     False                         True
start_time                     01:00:00                 20:00:00                      00:01:00                      23:00:00      22:20:00                      22:10:00                  01:00:00                     01:00:00
end_time                       19:30:00                 23:00:00                      19:30:00                      23:59:00      23:59:00                      23:59:00                  23:00:00                     23:00:00
required_machine                   None                     None                          None                          None          None                          None                      None                         None
right_machine                      True                     True                          True                          True          True                          True                      True                         True
time_to_run                       False                    False                         False                          True          True                          True                     False                        False
previous_required                  None       run_capital_update  run_strategy_order_generator  run_strategy_order_generator  run_cleaners  run_strategy_order_generator  run_daily_prices_updates                  run_systems
previous_finished                  True                     True                          True                          True          True                          True                      True                        False
time_to_stop                       True                     True                          True                         False         False                         False                      True                         True


"Methods are called from within processes. We list the methods in reverse order
from when they last ran; older processes first. If something hasn't run for some
reason it will be at the top of this list. "

=============================================================================================
                                      Status of methods                                      
=============================================================================================

                                                           process_name last_run_or_heartbeat
method_or_strategy                                                                           
update_total_capital                                 run_capital_update           10/19 19:08
strategy_allocation                                  run_capital_update           10/19 19:08
handle_completed_orders                               run_stack_handler           10/19 19:26
process_fills_stack                                   run_stack_handler           10/19 19:26

"....truncated for space...."

status_report                                               run_reports           10/19 23:00
backup_arctic_to_csv                                        run_backups           10/19 23:00


"Here's a list of all adjusted prices we've generated and FX rates. Again, listed oldest first.
If a market closes or something goes wrong then the price would be stale. Notice the Asian markets
near the top for which we've had no price since this morning - not a surprise, and the
FX rates with timestamp 23:00 which means they're daily prices (I don't collect intraday FX prices). "

==============================================
Status of adjusted price / FX price collection
==============================================

                last_update
name                       
KOSPI   2020-10-19 08:00:00
KR3     2020-10-19 08:00:00
KR10    2020-10-19 08:00:00
OAT     2020-10-19 18:00:00
CAC     2020-10-19 19:00:00

"....truncated for space...."

US20    2020-10-19 21:00:00
JPYUSD  2020-10-19 23:00:00
AUDUSD  2020-10-19 23:00:00
CADUSD  2020-10-19 23:00:00
CHFUSD  2020-10-19 23:00:00
EURUSD  2020-10-19 23:00:00
GBPUSD  2020-10-19 23:00:00
HKDUSD  2020-10-19 23:00:00
KRWUSD  2020-10-19 23:00:00


"Optimal positions are generated by the backtest that runs daily; this hasn't
quite close yet hence these are from the previous friday."

=====================================================
        Status of optimal position generation        
=====================================================

                                          last_update
name                                                 
medium_speed_TF_carry/AEX     2020-10-16 22:54:20.386
medium_speed_TF_carry/AUD     2020-10-16 22:54:21.677
medium_speed_TF_carry/BOBL    2020-10-16 22:54:22.386
medium_speed_TF_carry/BTP     2020-10-16 22:54:22.999

"....truncated for space...."

medium_speed_TF_carry/V2X     2020-10-16 22:54:57.874
medium_speed_TF_carry/VIX     2020-10-16 22:54:58.551
medium_speed_TF_carry/WHEAT   2020-10-16 22:55:00.283


"This shows the status of any trade and position limits: I've just reset
 these so the numbers are pretty boring"

=========================================================================================================================================
                                                         Status of trade limits                                                          
=========================================================================================================================================

                      instrument_code  period_days  trade_limit  trades_since_last_reset  trade_capacity_remaining  time_since_last_reset
strategy_name                                                                                                                            
                                 US10            1            3                        0                         3 0 days 00:00:00.000015
medium_speed_TF_carry            US10            1            3                        0                         3 0 days 00:00:00.000011
                                  KR3            1           12                        0                        12 0 days 00:00:00.000011
                                  AEX            1            1                        0                         1 0 days 00:00:00.000010

"....truncated for space...."

                                  V2X            1            6                        0                         6 0 days 00:00:00.000010
                                  OAT            1            2                        0                         2 0 days 00:00:00.000010
                                  US5           30           35                        0                        35 3 days 06:04:32.129139
                              EUROSTX           30            8                        0                         8 3 days 06:03:49.437166

"....truncated for space...."

                               COPPER           30            3                        0                         3 3 days 05:56:49.324062
                                WHEAT           30            6                        0                         6 3 days 05:56:36.958089
                                  V2X           30           32                        0                        32 3 days 05:56:26.776116
                                  OAT           30           11                        0                        11 3 days 05:56:21.616143


"Notice where we have a position we report on the limit, even if none is set.
In this case I've set instrument level, but not strategy/instrument position limits"

=====================================================
              Status of position limits              
=====================================================

                             keys  position pos_limit
0    medium_speed_TF_carry/GAS_US      -1.0  no limit
1       medium_speed_TF_carry/AUD       1.0  no limit
2      medium_speed_TF_carry/BOBL       2.0  no limit

"....truncated for space...."

12      medium_speed_TF_carry/BTP       3.0  no limit
13      medium_speed_TF_carry/MXP       4.0  no limit
0                             V2X      -5.0        35
1                             BTP       3.0        10
2                         LEANHOG       0.0         8

"....truncated for space...."

34                           US10       0.0        16
35                        LIVECOW       0.0        11
36                        EDOLLAR      11.0        86
37                            US5       0.0        39


"Overrides allow us to reduce or eliminate positions temporarily in specific
instruments, but I'm not using these right now"

===================
Status of overrides
===================

Empty DataFrame
Columns: [override]
Index: []

"Finally we check for instruments that are locked due to a position mismatch:
 see the reconcile report for details"

Locked instruments (position mismatch): []

********************************************************************************
                              END OF STATUS REPORT                              
********************************************************************************
```


### Trade report

The trade report lists all trades recorded in the database, and allows you to analyse slippage in very fine detail.  On a daily basis it is run for the previous 24 hours. On an ad hoc basis, it can be run for any time period (recent or in the past).

Here is an example, with annotations added in quotes (""):

```

********************************************************************************
              Trades report produced on 2020-10-20 09:25:43.596580              
********************************************************************************

"Here is a list of trades with basic information. Note that due to an issue with the way
roll trades are displayed, they are shown with fill 0."

==================================================================================================================
                                                  Broker orders                                                   
==================================================================================================================

         instrument_code          strategy_name           contract_id       fill_datetime    fill     filled_price
order_id                                                                                                          
30365                V2X  medium_speed_TF_carry            [20201200] 2020-10-02 07:51:53    (-1)           (27.7)
30366                KR3  medium_speed_TF_carry            [20201200] 2020-10-05 01:01:00     (1)         (112.01)

"....truncated for space...."


30378            EDOLLAR  medium_speed_TF_carry            [20230900] 2020-10-14 13:50:32    (-1)          (99.61)
30380               CORN  _ROLL_PSEUDO_STRATEGY  [20201200, 20211200] 2020-10-15 09:25:54  (0, 0)  (397.75, 394.0)
30379                V2X  _ROLL_PSEUDO_STRATEGY  [20201100, 20201200] 2020-10-15 09:24:32  (0, 0)   (25.25, 24.25)
30383                V2X  _ROLL_PSEUDO_STRATEGY  [20201100, 20201200] 2020-10-15 09:43:30  (0, 0)     (25.5, 24.4)
30388               KR10  medium_speed_TF_carry            [20201200] 2020-10-20 02:00:21     (1)         (133.03)


================================================================================================================================================================
                                                                             Delays                                                                             
================================================================================================================================================================

"We now look at timing. When was the parent order generated (the order at instrument level
 that generated this specific order) versus when the order was submitted to the broker?
Normally this is the night before, when the backtest is run, but for roll orders there
are no parents, and also for manual orders. In our simulation we assume that orders
are generated with a one business day delay. Here we're mostly doing better than that.
Once submitted, how long did it take to fill the order? Issues with timestamps when I
ran this report mean that some orders that apparently got filled before they were submitted, we ignore these. "

         instrument_code          strategy_name parent_generated_datetime         submit_datetime       fill_datetime submit_minus_generated filled_minus_submit
order_id                                                                                                                                                        
30365                V2X  medium_speed_TF_carry   2020-10-01 21:56:29.669 2020-10-02 08:50:03.637 2020-10-02 07:51:53                  39214                 NaN
30366                KR3  medium_speed_TF_carry   2020-10-02 21:57:41.427 2020-10-05 02:00:07.262 2020-10-05 01:01:00                 187346                 NaN
30367            EDOLLAR  medium_speed_TF_carry                       NaT 2020-10-05 12:43:01.885 2020-10-05 11:48:02                    NaN                 NaN
30368            EDOLLAR  medium_speed_TF_carry   2020-10-06 21:58:12.765 2020-10-07 00:30:32.000 2020-10-06 23:35:32                9139.24                 NaN

"....truncated for space...."

30380               CORN  _ROLL_PSEUDO_STRATEGY                       NaT 2020-10-15 09:24:46.000 2020-10-15 09:25:54                    NaN                  68
30379                V2X  _ROLL_PSEUDO_STRATEGY                       NaT 2020-10-15 09:22:16.000 2020-10-15 09:24:32                    NaN                 136
30383                V2X  _ROLL_PSEUDO_STRATEGY                       NaT 2020-10-15 09:41:42.000 2020-10-15 09:43:30                    NaN                 108
30388               KR10  medium_speed_TF_carry   2020-10-16 22:54:38.166 2020-10-20 02:00:16.000 2020-10-20 02:00:21                 270338                   5


==========================================================================================================================================================================================================================================================
                                                                                                                 Slippage (ticks per lot)                                                                                                                 
==========================================================================================================================================================================================================================================================

"We can calculate slippage in many different units. We start with 'ticks', units of price
(not strictly ticks I do know that...). The reference price is the price when we generated
the parent order (usually the closing price from the day before). The mid price is the mid
 price when we submit. The side price is the price we would pay if we submitted a market order
 (the best bid if we're selling, best offer if we're buying). The limit price is whatever
 the algo submits the order for initially. Normally an algo will try and execute passively,
 so the limit price would normally be the best offer if we're selling, best bid if we're
buying. Alternatively, if the parent order has a limit (for strategies that try and
achieve particular prices) the algo should use that price. The filled price is self
explanatory. We can then measure our slippage in different ways: caused by delay (side
price versus reference price - delays tend to add a lot of variability, but usually net
out very close to zero in our backtest (checking actual delays over a long period of time
 should confirm this), caused by bid/ask spread (mid versus side price, which is what we
assume we pay in a backtest), and caused by execution (side price versus fill, if our algo
is doing it's thing this should offset some of our costs). We can also measure the quality
 of our execution (initial limit versus fill) and how we did versus the required limit order
 (if relevant). Negative numbers are bad (we paid), positive are good (we earned).
Take the first order as an example (V2X sell one contract) with no parent order limit
price, the market moved 0.225 points in our favour from 27.45 the night before to a mid
 of 27.675 (bid 27.65, offer 27.7). If we'd paid up we would have sold at 27.65 side
price (bid/ask cost -0.025). We submitted a limit order of 27.7 at the offer, and were
filled there. So our execution cost was positive 0.05. Our total trading cost was -0.025+0.05 = 0.025."

         instrument_code          strategy_name    trade parent_reference_price parent_limit_price calculated_mid_price calculated_side_price limit_price calculated_filled_price   delay bid_ask execution versus_limit versus_parent_limit total_trading
order_id                                                                                                                                                                                                                                                  
30365                V2X  medium_speed_TF_carry     (-1)                  27.45               None               27.675                 27.65        27.7                    27.7   0.225  -0.025      0.05           -0                 NaN         0.025
30366                KR3  medium_speed_TF_carry      (1)                 112.08               None              111.995                   112      111.99                  112.01   0.085  -0.005     -0.01        -0.02                 NaN        -0.015
30367            EDOLLAR  medium_speed_TF_carry     (-1)                    NaN               None              99.6725                 99.67      99.675                   99.67     NaN -0.0025        -0       -0.005                 NaN       -0.0025
30368            EDOLLAR  medium_speed_TF_carry     (-1)                 99.645               None              99.6425                 99.64      99.645                   99.64 -0.0025 -0.0025        -0       -0.005                 NaN       -0.0025

"....truncated for space...."

30380               CORN  _ROLL_PSEUDO_STRATEGY  (1, -1)                    2.5               None                    4                  4.25        3.75                    3.75    -1.5   -0.25       0.5            0                 NaN          0.25
30379                V2X  _ROLL_PSEUDO_STRATEGY  (1, -1)                   0.85               None                 1.05                  1.15           1                       1    -0.2    -0.1      0.15            0                 NaN          0.05
30383                V2X  _ROLL_PSEUDO_STRATEGY  (1, -1)                   0.85               None                 1.05                  1.15        0.95                     1.1    -0.2    -0.1      0.05        -0.15                 NaN         -0.05
30388               KR10  medium_speed_TF_carry      (1)                 132.45               None              133.035                133.04      133.03                  133.03  -0.585  -0.005      0.01            0                 NaN         0.005


=======================================================================================================================================================================
                                                         Slippage (normalised by annual vol, BP of annual SR)                                                          
=======================================================================================================================================================================

"Ticks are meaningless as it depends on how volatile an instrument is. We divide by the annual
vol of an instrument, in price terms, to get a normalised figure. This is  multiplied by 10000
to get a basis point figure. For example the V2X trade had bid/ask slippage of 0.025, and the
 annual vol is currently 11.585; that works out to 0.025 / 11.585 = 0.00216, or 21.6 basis
points. Note that ignoring holding costs using my 'speed limit' concept we'd be able to do
0.13 / 0.00216 = 60 trades a year in V2X (or 48 if you assume monthly rolls), to put it another
 way the cost budget is 1300 basis points."

         instrument_code          strategy_name    trade last_annual_vol delay_vol bid_ask_vol execution_vol versus_limit_vol versus_parent_limit_vol total_trading_vol
order_id                                                                                                                                                               
30365                V2X  medium_speed_TF_carry     (-1)         11.5805   194.292    -21.5879       43.1759               -0                     NaN           21.5879
30366                KR3  medium_speed_TF_carry      (1)        0.829709   1024.46    -60.2621      -120.524         -241.048                     NaN          -180.786
30367            EDOLLAR  medium_speed_TF_carry     (-1)        0.224771       NaN    -111.224            -0         -222.448                     NaN          -111.224
30368            EDOLLAR  medium_speed_TF_carry     (-1)        0.224771  -111.224    -111.224            -0         -222.448                     NaN          -111.224

"....truncated for space...."

30380               CORN  _ROLL_PSEUDO_STRATEGY  (1, -1)         71.6612  -209.318    -34.8864       69.7727                0                     NaN           34.8864
30379                V2X  _ROLL_PSEUDO_STRATEGY  (1, -1)         11.5805  -172.704    -86.3518       129.528                0                     NaN           43.1759
30383                V2X  _ROLL_PSEUDO_STRATEGY  (1, -1)         11.5805  -172.704    -86.3518       43.1759         -129.528                     NaN          -43.1759
30388               KR10  medium_speed_TF_carry      (1)         4.18168  -1398.96    -11.9569       23.9138                0                     NaN           11.9569


==================================================================================================================================================================================
                                                                           Slippage (In base currency)                                                                            
==================================================================================================================================================================================

"Finally we can work out the slippage in base currency, i.e. actual money cost by multiplying
 ticks by the value of a price point in base currency (GBP for me)"

         instrument_code          strategy_name    trade value_of_price_point delay_cash bid_ask_cash execution_cash versus_limit_cash versus_parent_limit_cash total_trading_cash
order_id                                                                                                                                                                          
30365                V2X  medium_speed_TF_carry     (-1)              90.8755     20.447     -2.27189        4.54377                -0                      NaN            2.27189
30366                KR3  medium_speed_TF_carry      (1)              677.198    57.5618     -3.38599       -6.77198           -13.544                      NaN            -10.158
30367            EDOLLAR  medium_speed_TF_carry     (-1)               1930.7        NaN     -4.82676             -0          -9.65352                      NaN           -4.82676
30368            EDOLLAR  medium_speed_TF_carry     (-1)               1930.7   -4.82676     -4.82676             -0          -9.65352                      NaN           -4.82676

"....truncated for space...."

30380               CORN  _ROLL_PSEUDO_STRATEGY  (1, -1)              38.6141   -57.9211     -9.65352         19.307                 0                      NaN            9.65352
30379                V2X  _ROLL_PSEUDO_STRATEGY  (1, -1)              90.8755   -18.1751     -9.08755        13.6313                 0                      NaN            4.54377
30383                V2X  _ROLL_PSEUDO_STRATEGY  (1, -1)              90.8755   -18.1751     -9.08755        4.54377          -13.6313                      NaN           -4.54377
30388               KR10  medium_speed_TF_carry      (1)              677.198   -396.161     -3.38599        6.77198                 0                      NaN            3.38599

"Then follows a very long section, which is only really useful for doing annual analysis of
trades (unless you trade a lot!). For each type of slippage (delay, bid/ask, execution,
versus limit, versus parent limit, total trading [execution + bid/ask]) we calculate
summary statistics for each instrument and strategy: the total, count, mean, lower and
upper range (+/- two standard deviations), in three ways: ticks, vol adjusted, and base currency cash."

```



### Reconcile report

The reconcile report checks the consistency of positions and trades stored in the database, and with the broker. It is run on a daily basis, but can also be run ad hoc. Here is an example, with annotations added in quotes (""):

```

********************************************************************************
            Reconcile report produced on 2020-10-19 23:31:23.329834             
********************************************************************************

"Optimal positions are set by the nightly backtest. For this strategy we set an upper and
lower buffer region, so two figures are shown for the optimal. A break occurs if the
position is outside the buffer region. For example you can see for BTP that the current
 position (long 3) is higher than the upper buffer(2.4, rounded to 2). This either means
 that the relevant market hasn't traded yet, or there is something wrong with the system
(check the status report to see if a process or method hasn't run)."

=============================================================
               Optimal versus actual positions               
=============================================================

                               current        optimal  breaks
medium_speed_TF_carry AEX          0.0   -0.029/0.029   False
medium_speed_TF_carry AUD          1.0    1.030/1.301   False
medium_speed_TF_carry BOBL         2.0    1.696/2.107   False
medium_speed_TF_carry BTP          3.0    2.211/2.432    True
medium_speed_TF_carry BUND         0.0   -0.069/0.069   False

"....truncated for space...."

medium_speed_TF_carry JPY          0.0   -0.177/0.177   False
medium_speed_TF_carry KOSPI        0.0   -0.028/0.028   False
medium_speed_TF_carry KR10         1.0    1.953/2.131    True
medium_speed_TF_carry KR3          8.0    8.655/9.567    True
medium_speed_TF_carry LEANHOG      0.0  -0.616/-0.316   False

"....truncated for space...."

medium_speed_TF_carry VIX          1.0    0.410/0.541   False
medium_speed_TF_carry WHEAT        0.0   -0.135/0.115   False

"We now look at positions at a contract level, and compare those in the database with
those that the broker has recorded"

==========================================
             Positions in DB              
==========================================

   instrument_code contract_date  position
7              AUD      20201200       1.0
4             BOBL      20201200       2.0
5              BTP      20201200       3.0

"....truncated for space...."

14             V2X      20201200      -5.0
12             VIX      20201200       1.0


==========================================
             Positions broker             
==========================================

   instrument_code contract_date  position
10             AUD      20201214       1.0
9             BOBL      20201208       2.0
5              BTP      20201208       3.0

"....truncated for space...."

11             V2X      20201216      -5.0
12             VIX      20201216       1.0

"We now check for position breaks. These are of three kinds: an instrument position
is out of line with the optimal, the instrument positions are out of line with the
aggregate across contract positions, or the broker and database disagree on what the
 contract level positions are. The first problem should be fixed automatically if the
system is running properly; the second or third may require the creation of manual trades:
see interactive_stack_handler script."

Breaks Optimal vs actual [medium_speed_TF_carry BTP, medium_speed_TF_carry KR10, medium_speed_TF_carry KR3]
 Breaks Instrument vs Contract []
 Breaks Broker vs Contract []

"We now compare the orders in the database for the last 24 hours with those the broker
 has on record. No automated check is done, but you can do this visually. No trades
were done for this report so I've pasted in trades from another day to illustrate what
it looks like. You can see the trades match up (ignore the fills shown as 0 this is
an artifact of the way trades are stored)."


=========================================================================================================
                                              Trades in DB                                               
=========================================================================================================

                         strategy_name           contract_id       fill_datetime    fill     filled_price
instrument_code                                                                                          
CORN             _ROLL_PSEUDO_STRATEGY  [20201200, 20211200] 2020-10-15 09:25:54  (0, 0)  (397.75, 394.0)
V2X              _ROLL_PSEUDO_STRATEGY  [20201100, 20201200] 2020-10-15 09:24:32  (0, 0)   (25.25, 24.25)
V2X              _ROLL_PSEUDO_STRATEGY  [20201100, 20201200] 2020-10-15 09:43:30  (0, 0)     (25.5, 24.4)


=================================================================================================
                                       Trades from broker                                        
=================================================================================================

                strategy_name           contract_id       fill_datetime     fill     filled_price
instrument_code                                                                                  
V2X                            [20201118, 20201216] 2020-10-15 09:24:32  (1, -1)   (25.25, 24.25)
CORN                           [20201214, 20211214] 2020-10-15 09:25:54  (1, -1)  (397.75, 394.0)
V2X                            [20201118, 20201216] 2020-10-15 09:43:30  (1, -1)     (25.5, 24.4)



********************************************************************************
                              END OF STATUS REPORT                              
********************************************************************************


```


### Strategy report

The strategy report is bespoke to a strategy; it will load the last backtest file generated and report diagnostics from it. On a daily basis it runs for all strategies. On an ad hoc basis, it can be run for all or a single strategy.

The strategy reporting is determined by the parameter `strategy_list/strategy_name/reporting_code/function` in default.yaml or overridden in the private config .yaml file. The 'classic' reporting function is `sysproduction.strategy_code.report_system_classic.report_system_classic`

Here is an example, with annotations added in quotes (""):

```

********************************************************************************
Strategy report for medium_speed_TF_carry backtest timestamp 20201012_215827 produced at 2020-10-12 23:15:08.677151
********************************************************************************



================================================================================================================================================================================================================================================================================================================================================================================
                                                                                                                                                                              Unweighted forecasts                                                                                                                                                                              
================================================================================================================================================================================================================================================================================================================================================================================

"This is a matrix of all forecast values for each instrument, before weighting. Not shown for space reasons"


================================================================================================================================================================================================================================================================================================================================================================================
                                                                                                                                                                                Forecast weights                                                                                                                                                                                
================================================================================================================================================================================================================================================================================================================================================================================

"This is a matrix of all forecast weights for each instrument, before weighting. Not shown for space reasons"



================================================================================================================================================================================================================================================================================================================================================================================
                                                                                                                                                                               Weighted forecasts                                                                                                                                                                               
================================================================================================================================================================================================================================================================================================================================================================================

"This is a matrix of all forecast values for each instrument, after weighting. Not shown for space reasons"

"Here we calculate the vol target for the strategy"

Vol target calculation {'base_currency': 'GBP', 'percentage_vol_target': 25.0, 'notional_trading_capital': 345040.64, 'annual_cash_vol_target': 86260.16, 'daily_cash_vol_target': 5391.26}

"Now we see how the instrument vol is calculated. These figures are also calculated independently in the risk report"

================================================================
                        Vol calculation                         
================================================================

         Daily return vol       Price  Daily % vol  annual % vol
AEX                6.3238    573.2500       1.1031       17.6504
AUD                0.0045      0.7214       0.6267       10.0272

"... truncated for space"

VIX                0.7086     27.7000       2.5580       40.9277
WHEAT             10.7527    596.5000       1.8026       28.8420


=========================================================================================================================================
                                                           Subsystem position                                                            
=========================================================================================================================================

"Calculation of subsystem positions: the position we'd have on if the entire system
 was invested in a single instrument. Abbreviations won't make sense unless you've read my first book, 'Systematic Trading'"

         Block_Value  Daily price % vol         ICV    FX      IVV  Daily Cash Vol Tgt  Vol Scalar  Combined forecast  subsystem_position
AEX          1146.50               1.10     1264.76  0.91  1094.74             5391.26        4.92               0.00                0.00
AUD           721.40               0.63      452.10  0.77   352.52             5391.26       15.29               9.02               13.79

"... truncated for space"

V2X            23.90               3.29       78.57  0.91    68.01             5391.26       79.28             -10.36              -82.15
VIX           277.00               2.56      708.56  0.77   552.48             5391.26        9.76               8.80                8.59
WHEAT         298.25               1.80      537.63  0.77   419.21             5391.26       12.86               1.07                1.38


=================================================================
                       Portfolio positions                       
=================================================================


"Final notional positions"

         subsystem_position  instr weight  IDM  Notional position
AEX                   0.000         0.022  2.5              0.000
AUD                  13.792         0.033  2.5              1.149

"... truncated for space"

V2X                 -82.154         0.025  2.5             -5.135
VIX                   8.592         0.025  2.5              0.537
WHEAT                 1.379         0.033  2.5              0.115


===============================================================================================
                                     Positions vs buffers                                      
===============================================================================================

"Shows the calculation of buffers. The position at timestamp is the position when
 the backtest was run; the current position is what we have on now"

         Notional position  Lower buffer  Upper buffer  Position at timestamp  Current position
AEX                    0.0          -0.0           0.0                    0.0               0.0
AUD                    1.1           1.0           1.3                    1.0               1.0

"... truncated for space"

V2X                   -5.1          -5.6          -4.6                   -4.0              -4.0
VIX                    0.5           0.5           0.6                    1.0               1.0
WHEAT                  0.1           0.0           0.2                    0.0               0.0

End of report for medium_speed_TF_carry

```


### Risk report

The risk report.... you're smart people, you can guess. It is run on a daily basis, but can also be run ad hoc. Here is an example, with annotations added in quotes (""):

```

********************************************************************************
               Risk report produced on 2020-10-19 23:54:09.835241               
********************************************************************************

"Our expected annual standard deviation is 10.6% a year, across everything"

Total risk across all strategies, annualised percentage 10.6

========================================
Risk per strategy, annualised percentage
========================================

"We now break this down by strategy, taking into account the capital allocated to
each strategy. The 'roll pseduo strategy' is used to generate roll trades and should
 never have any risk on. ETFHedge is another nominal strategy"


                               risk
_ROLL_PSEUDO_STRATEGY  0
medium_speed_TF_carry  10.6
ETFHedge               0


============================================================================================================================================================================================================================================================
                                                                                                                      Instrument risk                                                                                                                       
============================================================================================================================================================================================================================================================

"Detailed risk calculations for each instrument. Most of these are, hopefully, self explanatory,
but in case they aren't, from left to right: daily standard deviation in price units,
annualised std. dev in price units, the price, daily standard deviation in % units (std dev
in price terms / price), annual % std. dev, the point size (the value of a 1 point price movement)
 expressed in the base currency (GBP for me), the contract exposure value in GBP (point size * price),
 daily risk standard deviation in GBP for owning one contract (daily % std dev * exposure value,
 or daily price std dev * point size), annual risk per contract (daily risk * 16), current position,
total capital at risk, exposure of position held as % of capital (contract exposure * position / capital),
 annual risk of position held as % of capital (annual risk per contract / capital)."

         daily_price_stdev  annual_price_stdev   price  daily_perc_stdev  annual_perc_stdev  point_size_base  contract_exposure  daily_risk_per_contract  annual_risk_per_contract  position   capital  exposure_held_perc_capital  annual_risk_perc_capital
GAS_US                 0.1                 1.5     3.3               2.9               46.6           7722.8            25423.5                    740.4                   11845.6      -1.0  353675.6                        -7.2                      -3.3
EUROSTX               36.4               582.0  3207.0               1.1               18.1              9.1            29143.8                    330.5                    5288.6      -2.0  353675.6                       -16.5                      -3.0
V2X                    0.7                11.6    24.9               2.9               46.5             90.9             2262.8                     65.8                    1052.4      -5.0  353675.6                        -3.2                      -1.5
"... truncated for space"PLAT                  22.7               363.9   855.5               2.7               42.5             38.6            33034.3                    878.1                   14050.1       1.0  353675.6                         9.3                       4.0
BTP                    0.3                 5.4   149.4               0.2                3.6            908.8           135777.1                    307.7                    4923.3       3.0  353675.6                       115.2                       4.2


============================================================================================================
                                                Correlations                                                
============================================================================================================

"Correlation of *instrument* returns - doesn't care about sign of position"

          V2X   OAT   BTP   KR3  SOYBEAN  KR10   AUD  GAS_US  PLAT   VIX  BOBL  EDOLLAR   MXP  EUROSTX  CORN
V2X      1.00  0.19 -0.14  0.12    -0.22  0.21 -0.56    0.01 -0.30  0.77  0.21     0.41 -0.48    -0.71 -0.08
OAT      0.19  1.00  0.46  0.22    -0.05  0.19 -0.09   -0.10 -0.12  0.09  0.60     0.39 -0.16    -0.13 -0.11
BTP     -0.14  0.46  1.00  0.08    -0.01  0.09  0.22   -0.10  0.04 -0.04  0.17     0.02 -0.13     0.06 -0.06

"... truncated for space"

MXP     -0.48 -0.16 -0.13 -0.15     0.29 -0.13  0.48   -0.09  0.32 -0.56 -0.14    -0.33  1.00     0.44  0.11
EUROSTX -0.71 -0.13  0.06 -0.08     0.17 -0.15  0.53    0.00  0.17 -0.54 -0.32    -0.43  0.44     1.00  0.02
CORN    -0.08 -0.11 -0.06 -0.06     0.68 -0.10  0.08    0.05  0.13 -0.10 -0.25    -0.15  0.11     0.02  1.00


********************************************************************************
                               END OF RISK REPORT                               
********************************************************************************


```

### Liquidity report


This allows us to check that markets are sufficiently liquid to trade. See this [blog post](https://qoppac.blogspot.com/2021/05/adding-new-instruments-or-how-i-learned.html) for more discussion.

I require minimum volume ($1.25 million per day in risk units, and 100 contracts per day). The report uses the last two weeks of trading to determine the relevant values (not configurable - be careful if using a report just after new years day when liquidity may have fallen).


Any instruments that don't meet these thresholds you should seriously consider removing from your portfolio.


```

********************************************************************************
            Liquidity report produced on 2021-07-08 14:58:39.926324             
********************************************************************************



================================================================
 Sorted by contracts: Less than 100 contracts a day is a problem
================================================================

                   contracts          risk
EU-FOOD           139.888889      0.364353
EU-RETAIL         177.444444      0.473291
MILK              210.250000      0.535316
EU-HEALTH         227.555556      0.958930
OATIES            266.750000      1.059968
RICE              322.250000      1.313220
EU-TRAVEL         455.777778      1.234303
EU-TECH           512.000000      1.891341
PALLAD            647.125000     53.219742
EU-UTILS          912.888889      2.733389
CHF              1101.250000      6.785555
EU-DIV30         1360.222222      2.717361
.... snip....

===================================================================
Sorted by risk: Less than $1.5 million of risk per day is a problem
===================================================================

                   contracts          risk
EU-FOOD           139.888889      0.364353
EU-RETAIL         177.444444      0.473291
MILK              210.250000      0.535316    <----- milk isn't liquid :-)'
EU-HEALTH         227.555556      0.958930
OATIES            266.750000      1.059968
EU-TRAVEL         455.777778      1.234303
RICE              322.250000      1.313220    <----- everything above this line might be too illiquid
EU-TECH           512.000000      1.891341
EU-DIV30         1360.222222      2.717361
EU-UTILS          912.888889      2.733389
USIRS5           1433.375000      2.873852
BITCOIN          1522.888889      3.181693
BBCOMM           3546.875000      3.637609
.... snip....
```


### Costs report

This allows us to check that the bid/ask spread costs set in the configuration file are sufficiently conservative. It will use three different sources, using data over the last 250 days (configurable):

- Half the bid/ask spread captured before an order is entered (first column below)
- The actual spread paid between the initial mid price and the price traded at (second column below). Positive numbers refer to a loss making spread (as normal), negative implies we managed to trade at better than mid (due to the execution algo)
- Half the bid/ask spread as periodically captured by the stack handler (3rd column)

We then calculate:

- The worst (highest) of the above values (fourth column below))
- The bid/ask spread cost configured in the instrument metadata database table (which in turn is normally initialised from ) (5th column)
- The % difference between the configured and worst cost, as a % of the configured cost. +1.00 is a 100% difference, eg the highest spread is twice what is currently configured (final column)

```

********************************************************************************
Costs report produced on 2021-07-08 14:58:53.991457 from 2020-10-31 14:58:50.220961 to 2021-07-08 14:58:50.220958
********************************************************************************



=================================================================================================
                                              Costs                                              
=================================================================================================

               bid_ask_trades  total_trades  bid_ask_sampled      Worst  Configured  % Difference
CRUDE_W_mini         0.037500     -0.006250         0.024006   0.037500    0.012500      2.000000  
         <---- bid/ask before trading is 200% higher: 3x higher than configured. Actual trades have negative costs
GAS_US_mini          0.002500      0.005000         0.006408   0.006408    0.002500      1.563158
PLAT                 0.350000      0.550000         0.275935   0.550000    0.240108      1.290640  
BBCOMM                    NaN           NaN         0.100000   0.100000    0.050000      1.000000  <---- we haven't done any actual trades here in last 250 days, only sampled'
EUROSTX              1.000000      0.500000         0.261483   1.000000    0.500000      1.000000  <---- configured is same as trades, but pre trade was double
SOYBEAN              0.250000      0.250000         0.201436   0.250000    0.125000      1.000000
GBP                  0.000100     -0.000000         0.000059   0.000100    0.000050      1.000000
......
OATIES                    NaN           NaN         0.750000   0.750000    0.625000      0.200000  <---- sampled spreads are 20% worse than configured
.....
BOBL                 0.005000      0.005000         0.005000   0.005000    0.005000      0.000000  <---- spot on!
EDOLLAR              0.002500      0.002500         0.002500   0.002500    0.002500      0.000000
SHATZ                     NaN           NaN         0.002500   0.002500    0.002500      0.000000
WHEAT                0.250000      0.187500         0.192896   0.250000    0.250000      0.000000
....
GOLD_micro           0.050000      0.050000         0.066546   0.066546    0.100000     -0.334545  <----- slippage about half or two thirds of configured value
GOLD                      NaN           NaN         0.058550   0.058550    0.088530     -0.338639
US-REALESTATE             NaN           NaN         0.066095   0.066095    0.100000     -0.339047
EU-TECH                   NaN           NaN         0.196570   0.196570    0.300000     -0.344765
COPPER                    NaN           NaN         0.000385   0.000385    0.000615     -0.374535
CRUDE_W                   NaN           NaN         0.008526   0.008526    0.014533     -0.413304
EUR                       NaN           NaN         0.000028   0.000028    0.000050     -0.439945
MXP                  0.000005      0.000005         0.000006   0.000006    0.000012     -0.443000
NZD                       NaN           NaN         0.000055   0.000055    0.000107     -0.483610
US2                       NaN           NaN         0.001953   0.001953    0.004000     -0.511719
EU-BASIC                  NaN           NaN         0.141791   0.141791    1.250000     -0.886568
ASX                       NaN           NaN              NaN        NaN         NaN           NaN   <----- instrument in configuration file but no costs included
.....
KOSPI                     NaN           NaN              NaN        NaN    0.025000           NaN   <--- slippage is configured, but no sampling or trading done
....
```

## Customize report generation in the run_report process

It is possible to setup a custom report configuration. Say for example that you would like to push reports to a git repo 
[like this](https://github.com/robcarver17/reports). In that case you would need to change the default behaviour, sending reports
via email, to saving the report as a file Files would be stored in according to what is declared in private_config.yaml
`reporting_directory`. Customization is done in the private_config.yaml. Example of reporting customization is; 

```
 reports:
  slippage_report:
    title: "Slippage report"
    function: "sysproduction.reporting.slippage_report.slippage_report"
    calendar_days_back: 250
    output: "file"

  costs_report:
    title: "Costs report"
    function: "sysproduction.reporting.costs_report.costs_report"
    output: "file"
    calendar_days_back: 250

```
The available reports can be found by interrogating the `dataReports` object,
e.g.:
```python
from sysproduction.data.reports import dataReports
print(dataReports().get_default_reporting_config_dict().keys())
```
