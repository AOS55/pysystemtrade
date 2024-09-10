# Overview of a production system

Here are the steps you need to follow to set up a production system. I assume you already have a backtested system in pysystemtrade, with appropriate python libraries etc.

1. Consider your implementation options
2. Ensure you have a private area for your system code and configuration
3. Finalise and store your backtested system configuration
4. If you want to automatically execute, or get data from a broker, then set up a broker
5. Set up any other data sources you need.
6. Set up a database for storage, including a backup
7. Have a strategy for reporting, diagnostics, and logs
8. Write some scripts to kick off processes to: get data, get accounting information, calculate optimal positions, execute trades, run reports, and do backups and housekeeping.
9. Schedule your scripts to run regularly
10. Regularly monitor your system, and deal with any problems


## Implementation options

Standard implementation for pysystemtrade is a fully automated system running on a single local machine. In this section I briefly describe some alternatives you may wish to consider.

My own implementation runs on a Linux machine, and some of the implementation details in this document are Linux specific. Windows and Mac users are welcome to contribute with respect to any differences.

### Automation options

You can run pysystemtrade as a fully automated system, which does everything from getting prices through to executing orders.
If running fully automated, [IBC](https://github.com/IbcAlpha/IBC) is very useful. But other patterns make sense. In particular you may wish to do your trading manually, after pulling in prices and generating optimal positions manually. It will also possible to trade manually, but allow pysystemtrade to pick up your fills from the broker rather than entering them manually. For example, you might not trust the system (I wouldn't blame you), it gives you more control, you might think your execution is better than an algo, you might be doing some testing, or you simply want to use a broker that doesn't offer an API.

I suggest the following:

- From `run_stack_handler`.yaml process configuration (#process-configuration) in your `private_control_config.yaml` file, remove the method `create_broker_orders_from_contract_orders`
- Run `interactive_order_stack` to check what contract orders have been created.
- Do the trade
- Use 'manually fill broker or contract order' in `interactive_order_stack` to enter the fill details.

Everything else should be allowed to run as normal.


### Machines, containers and clouds

Pysystemtrade can be run locally in the normal way, on a single machine. But you may also want to consider containerisation (see [my blog post](https://qoppac.blogspot.com/2017/01/playing-with-docker-some-initial.html)), or even implementing on AWS or another cloud solution. You could also spread your implementation across several local machines.

If spreading your implementation across several machines bear in mind:

- Interactive brokers
   - interactive brokers Gateway will need to have the ip address of all relevant machines that connect to it in the whitelist
   - you will need to modify the `private_config.yaml` system configuration file so it connects to a different IP address `ib_ipaddress: '192.168.0.10'`
- Mongodb
   - Add an ip address to the `bind_ip` line in the `/etc/mongod.conf` file to allow connections from other machines `eg bind_ip=localhost, 192.168.0.10`
   - You may need to change your firewall settings, either UFW (`sudo ufw enable`, `sudo ufw allow 27017 from 192.168.0.10`) or iptables
   - you will need to modify the `private_config.yaml` system configuration file so it connects to a different IP address `mongo_host: 192.168.0.13`
   - you may want to enforce [further security protocol](https://docs.mongodb.com/manual/administration/security-checklist/)
- [Process configuration](#process-configuration); you will want to specify different machine names for each process in your private yaml config file.

### Backup machine

If you are running your implementation locally, or on a remote server that is not a cloud, then you should seriously consider a backup machine. The backup machine should have an up to date environment containing all the relevant applications, code and libraries, and on a regular basis you should update the local data stored on that machine (see [backup](#data-backup)). The backup machine doesn't need to be turned on at all times, unless you are trading in such a way that a one hour period without trading would be critical (in my experience, one hour is the maximum time to get a backup machine on line assuming the code is up to date, and the data is less than 24 hours stale). I also encourage you to perform a scheduled 'failover' on regular basis: stop the live machine running (best to do this at a weekend), copy across any data to the backup machine, start up the backup machine. The live machine then becomes the backup.

### Multiple systems

You may want to run multiple trading systems on a single machine. Common use cases are:

- You want to run relative value systems *
- You want different systems for different time frames (eg intra day and slower trading) *
- You want different systems for different asset classes eg stocks and ETFs, or futures
- You want to run the same system, but for different trading accounts (pysystemtrade can't handle multiple accounts natively)
- You want a paper trading and live trading system

*for these cases I plan to implement functionality in pysystemtrade so that it can handle them in the same system.

To handle this I suggest having multiple copies of the pysystemtrade environment. You will have a single crontab, but you will need multiple script, echos and other directories. You will need to change the private config file so it points to different mongo_db database names. If you don't want multiple copies of certain data (eg prices) then you should hardcode the database_name in the relevant files whenever a connection is made eg mongo_db = mongoDb(database_name='whatever'). See storing futures and spot FX data for more detail.

Finally you should set the field `ib_idoffset` in the private config file private/private_config.yaml so that there is no chance of duplicate clientid connections; setting one system to have an id offset of 1, the next offset 1000, and so on should be sufficient.

## Code and configuration management

Your trading strategy will consist of pysystemtrade, plus some specific configuration files, plus possibly some bespoke code. You can either implement this as:

- separate environment, pulling in pysystemtrade as a 'yet another library'
- everything in pysystemtrade, with all specific code and configuration in the 'private' directory that is excluded from git uploads.

Personally I prefer the latter as it makes a neat self contained unit, but this is up to you.

### Managing your separate directories of code and configuration

I strongly recommend that you use a code repo system or similar to manage your non pysystemtrade code and configuration. Since code and configuration will mostly be in text (or text like) yaml files a code repo system like git will work just fine. I do not recommend storing configuration in database files that will need to be backed up separately, because this makes it more complex to store old configuration data that can be archived and retrieved if required.

### Managing your private directory

Since the private directory is excluded from the git system (since you don't want it appearing on github!), you need to ensure it is managed separately. I have a separate repo for my private stuff, for which I have a local clone in directory ~/private. Incidentally, github are now offering free private repos, so that is another option.
I then use a bash script which I run in lieu of a normal git add/ commit / push cycle, to commit both private and public code:

```
# pass commit quote as an argument
# For example:
# mygitpush "this is a commit description string"
#
# copy the contents of the private directory to another, git controlled, directory
#
# we use rsync so we can exclude the git directory; which will screw things up as there is already one there
#
rsync -av ~/pysystemtrade/private/ ~/private --exclude .git
#
# git add/commit/push cycle on the main pysystemtrade directory
#
cd ~/pysystemtrade/
git add -A
git commit -m "$1"
git push
#
# git add/commit/push cycle on the copied private directory
#
cd ~/private/
git add -A
git commit -m "$1"
git push
```

A second script is run instead of a git pull:

```
# git pull within git controlled private directory copy
cd ~/private/
git pull
# copy the updated contents of the private directory to pysystemtrade private directory
# use rsync to avoid overwriting git metadata
rsync -av ~/private/ ~/pysystemtrade/private --exclude .git
# git pull from main pysystemtrade github repo
cd ~/pysystemtrade/
git pull
```

### Custom private directory

If you prefer to keep your private config outside the *pysystemtrade* directory structure, this is possible too. Set
the environment variable `PYSYS_PRIVATE_CONFIG_DIR` with the full path to the custom directory:

```
PYSYS_PRIVATE_CONFIG_DIR=/home/user_name/private_custom_dir
```

or to set a custom config directory in the context of a single script

```
PYSYS_PRIVATE_CONFIG_DIR=/home/user_name/private_custom_dir python sysproduction/whatever.py
```


## Finalise your backtest configuration

You can just re-run a full daily backtest to generate your positions. This will probably mean that you end up refitting parameters like instrument weights and forecast scalars. This is pointless, slow, a waste of time, and potentially dangerous. Instead I'd suggest using fixed values for all fitted parameters in a live trading system.

The following convenience function will take your backtested system, and create a dict which includes fixed values for all estimated parameters:

```python
# Assuming futures_system already contains a system which has estimated values
from systems.diagoutput import systemDiag

sysdiag = systemDiag(system)
sysdiag.yaml_config_with_estimated_parameters('someyamlfile.yaml',
                                              attr_names=['forecast_scalars',
                                                                  'forecast_weights',
                                                                  'forecast_div_multiplier',
                                                                  'forecast_mapping',
                                                                  'instrument_weights',
                                                                  'instrument_div_multiplier'])

```

Change the list of attr_names depending on what you want to output. You can then merge the resulting .yaml file into your production backtest .yaml file.

Don't forget to turn off the flags for `use_forecast_div_mult_estimates`, `use_forecast_scale_estimates`, `use_forecast_weight_estimates`, #`use_instrument_div_mult_estimates`, and `use_instrument_weight_estimates`.  You don't need to change flag for forecast mapping, since this isn't done by default.


## Linking to a broker

You are probably going to want to link your system to a broker, to do one or more of the following things:

- get prices
- get account value and profitability
- do trades
- get trade fills

... although one or more of these can also be done manually.

You should now read [connecting pysystemtrade to interactive brokers](/docs/IB.md). The fields `broker_account`,`ib_ipaddress`, `ib_port` and `ib_idoffset` should be set in the [private config file](/private/private_config.yaml).


## Other data sources

You might get all your data from your broker, but there are good reasons to get data from other sources as well:

- multiple sources can improve accuracy
- multiple sources can provide redundancy in case of feed issues
- you can't get the relevant data from your broker
- the relevant data is cheaper elsewhere

You should now read [getting and storing futures and spot FX data](/docs/data.md) for some hints on writing API layers for other data sources.


## Data storage

Various kinds of data files are used by the pysystemtrade production system. Broadly speaking they fall into the following categories:

- accounting (calculations of profit and loss)
- diagnostics
- prices (see [storing futures and spot FX data](/docs/data.md))
- positions
- other state and control information
- static configuration files

The default option is to store these all into a mongodb database, except for configuration files which are stored as .yaml and .csv files. Time series data is stored in [arctic](https://github.com/man-group/arctic) which also uses mongodb. Databases used will be named with the value of parameter `mongo_db` in the private config file /private/private_config.yaml. A separate Arctic database will have the same name, with the suffix `_arctic`.

## Data backup

### Mongo data

Assuming that you are using the default mongob for storing, then I recommend using [mongodump](https://docs.mongodb.com/manual/reference/program/mongodump/#bin.mongodump) on a daily basis to back up your files. Other more complicated alternatives are available (see the [official mongodb man page](https://docs.mongodb.com/manual/core/backups/)). You may also want to do this if you're transferring your data to e.g. a new machine.

To avoid conflicts you should [schedule](#scheduling) your backup during the 'deadtime' for your system (see [scheduling](#scheduling)).


Linux:
```
# dumps everything into dump directory
# make sure a mongo-db instance is running with correct directory, but ideally without any load; command line: `mongod --dbpath $MONGO_DATA`
mongodump -o ~/dump/

# copy dump directory to another machine or drive. This will create a directory $MONGO_BACKUP_PATH/dump/
cp -rf ~/dump/* $MONGO_BACKUP_PATH
```

This is done by the scheduled backup process (see [scheduling](#scheduling)), and also by [this script](#backup-files)

Then to restore, from a linux command line:
```
cp -rf $MONGO_BACKUP_PATH/dump/ ~
# Now make sure a mongo-db instance is running with correct directory
# If required delete any existing instances of the databases. If you don't do this the results may be unpredictable...
mongo
# This starts a mongo client
> show dbs
admin              0.000GB
arctic_production  0.083GB
config             0.000GB
local              0.000GB
meta_db            0.000GB
production         0.000GB
# Most likely we want to remove 'production' and 'arctic_production'
> use production
> db.dropDatabase()
> use arctic_production
> db.dropDatabase()
> exit
# Now we run the restore (back on the linux command line)
mongorestore
```


### Mongo / csv data

As I am super paranoid, I also like to output all my mongo_db data into .csv files, which I then regularly backup. This will allow a system recovery, should the mongo files be corrupted.

This currently supports: FX, individual futures contract prices, multiple prices, adjusted prices, position data, historical trades, capital, contract meta-data, spread costs, optimal positions. Some other state information relating to the control of trading and processes is also stored in the database and this will be lost, however this can be recovered with a little work: roll status, trade limits, position limits, and overrides. Log data will also be lost; but archived [echo files](#echos-stdout-output) could be searched if necessary.


Linux script:
```
. $SCRIPT_PATH/backup_arctic_to_csv
```

## Echos, Logging, diagnostics and reporting

We need to know what our system is doing, especially if it is fully automated. Here are the methods by which this should be done:

- Echos of stdout output from processes that are running
- Logging output in a file, tagged with keys to identify them
- Storage of diagnostics in a database, tagged with keys to identify them
- the option to run reports both scheduled and ad-hoc, which can optionally be automatically emailed

### Echos: stdout output

The [supplied crontab](/sysproduction/linux/crontab) contains lines like this:

```
SCRIPT_PATH="$HOME:/workspace3/psystemtrade/sysproduction/linux/scripts"
ECHO_PATH="$HOME:/echos"
#
0 6  * * 1-5 $SCRIPT_PATH/updatefxprices  >> $ECHO_PATH/updatefxprices.txt 2>&1
```

The above line will run the script `updatefxprices`, but instead of outputting the results to stdout they will go to `updatefxprices.txt`. These echo files are most useful when processes crash, in which case you may want to examine the stack trace. Usually however the log files will be more useful.

#### Cleaning old echo files

Over time echo files can get... large (my default position for logging is verbose). To avoid this there is a [daily cleaning process](#truncate-echo-files) which archives old echo files with a date suffix, and deletes anything more than a month old.

Note: the configuration variable echo_extension will need changing in `private_config.yaml` if you don't use .txt file extensions, otherwise cleaning won't work.

### Logging

pysystemtrade uses the [Python logging module](https://docs.python.org/3.10/library/logging.html). See the [user guide for more detail](/docs/backtesting.md#logging) about logging in sim. Python logging is powerful and flexible, and log messages can be [formatted as you like, and sent virtually anywhere](https://docs.python.org/3.10/howto/logging.html#logging-advanced-tutorial) by providing your own config. But this section describes the default provided production setup. 

In production, the requirements are more complex than in sim. As well as the context relevant attributes (that we have with sim), we also need
- ability to log to the same file from different processes
- output to console for echo files
- critical level messages to trigger an email

Configure the default production setup with:

```
PYSYS_LOGGING_CONFIG=syslogging.logging_prod.yaml
```

At the client side, (pysystemtrade) there are three handlers: socket, console, and email. There is a server (separate process) for the socket handler. More details on each below

### socket

Python doesn't support more than one process writing to a file at the same time. So, on the client side, log messages are serialised and sent over the wire. A simple TCP socket server receives, de-serialises, and writes them to disk. The socket server needs to be running first. The simplest way to start it:

```
python -u $PYSYS_CODE/syslogging/server.py
```

But that would write logs to the current working directory. Probably not what you want. Instead, pass the log file path 

```
python -u $PYSYS_CODE/syslogging/server.py --file /home/path/to/your/pysystemtrade.log
```

By default, the server accepts connections on port 6020. But if you want to use another

```
python -u $PYSYS_CODE/syslogging/server.py --port 6021 --file /home/path/to/your/pysystemtrade.log
```

The socket server also handles rotating the log files daily; the default setup rotates creates a new log at midnight each day, keeping the last 5 days' files. So after a week, the log directory file listing would look something like 

```
-rw-r--r--  1 user group 19944754 May  4 15:42 pysystemtrade.log
-rw-r--r--  1 user group 19030250 Apr 24 22:16 pysystemtrade.log.2023-04-24
-rw-r--r--  1 user group  6178163 Apr 25 22:16 pysystemtrade.log.2023-04-25
-rw-r--r--  1 user group  9465225 Apr 26 22:16 pysystemtrade.log.2023-04-26
-rw-r--r--  1 user group  4593885 Apr 27 16:53 pysystemtrade.log.2023-04-27
-rw-r--r--  1 user group  4414970 May  3 22:16 pysystemtrade.log.2023-05-03
```

The server needs to be running all the time. It needs to run in the background, start up on reboot, restart automatically in case of failure, etc. So a better way to do it would be to make it a service

#### socket server as a service

There is an example Linux systemd service file provided, see `examples/logging/logging_server.service`. And a setup guide [here](https://tecadmin.net/setup-autorun-python-script-using-systemd/). Basic setup for Debian/Ubuntu is:

- create a new file at `/etc/systemd/system/logging_server.service`
- paste the example file into it
- update the paths in `ExecStart`. If using a virtual environment, make sure to use the correct path to Python 
- update the `User` and `Group` values, so the log file is not owned by root
- update the path in `Environment`, if using a custom private config directory
- run the following commands to start/stop/restart etc

```
# reload daemon
sudo systemctl daemon-reload

# enable service (restart on boot)
sudo systemctl enable log_server.service

# view service status
sudo systemctl status log_server.service 

# start service
sudo systemctl start log_server.service

# stop service
sudo systemctl stop log_server.service

# restart
sudo systemctl restart log_server.service

# view service log (not pysystemtrade log)
sudo journalctl -e -u log_server.service
```

### console

All log messages also get sent to console, as with sim. The supplied `crontab` entries would therefore also pipe their output to the echo files

### email

There is a special SMTP handler, for CRITICAL log messages only. This handler uses the configured pysystemtrade email settings to send those messages as emails


#### Adding logging to your code

See the [logging docs](https://docs.python.org/3.10/library/logging.html) for usage examples. There are four ways to manage context attributes: 
* *overwrite* - passed attributes are merged with any existing, overwriting duplicates (the default)
* *preserve* - passed attributes are merged with any existing, preserving duplicates
* *clear* - existing attributes are cleared, passed ones added
* *temp* - passed attributes will only be used for one invocation

#### Examples

```python
# merging attributes: method 'overwrite' (default if no method supplied)
overwrite = get_logger("Overwrite", {"type": "first"})
overwrite.info("overwrite, type 'first'")
overwrite.info(
    "overwrite, type 'second', stage 'one'",
    method="overwrite",
    type="second",
    stage="one",
)

# merging attributes: method 'preserve'
preserve = get_logger("Preserve", {"type": "first"})
preserve.info("preserve, type 'first'")
preserve.info(
    "preserve, type 'first', stage 'one'", method="preserve", type="second", stage="one"
)

# merging attributes: method 'clear'
clear = get_logger("Clear", {"type": "first", "stage": "one"})
clear.info("clear, type 'first', stage 'one'")
clear.info("clear, type 'second', no stage", method="clear", type="second")
clear.info("clear, no attributes", method="clear")

# merging attributes: method 'temp'
temp = get_logger("temp", {"type": "first"})
temp.info("type should be 'first'")
temp.info(
    "type should be 'second' temporarily",
    method="temp",
    type="second",
)
temp.info("type should be back to 'first'")
```

#### Cleaning old logs

##### Echos

There is some code to clean up **echo** files. This code is run automatically from the [daily cleaning process](#clean-up-old-logs).

Python:

```python
from sysproduction.clean_truncate_log_files import clean_truncate_log_files
clean_truncate_log_files()
```

It defaults to deleting anything more than 30 days old.

##### Logs

With the provided production logging config, the cleaning of **log** files is managed by the Python logging server. Default config is to keep the last 5 days of production logs. To adjust this, see [syslogging/server.py](syslogging/server.py) 


### Reporting

Reports are run regularly to allow you to monitor the system and decide if any action should be taken. You can choose to have them emailed to you. To do this the email address, server and password *must* be set in `private_config.yaml`, as well as the address the email is being sent to (which can be the same as the sending email account):

```
email_address: "somebloke@anemailadress.com"
email_pwd: "h0Wm@nyLetter$ub$tiute$"
email_server: 'smtp.anemailadress.com'
email_to: "someotherbloke@anothermail.com"
```

Pysystemtrade will automatically try to negotiate TLS encryption when connecting to SMTP server and will resort to unencrypted communication only as a last resort.

To use Google SMTP server without trusting the config file with your plain text password, you can create an 'App password' specifically for pysystemtrade:
- Go to [Manage my Google account](https://myaccount.google.com/security) and its 'Signing in to Google' subsection
- Ensure that '2-step verification' is On
- In 'App passwords' section generate a new one: Select app: Mail. Select device: Other, and name it 'pysystemtrade'. Click 'Generate' and copy the 16-character password (such as 'abcd efgh ijkl mnop').

For sending notifications via gmail to yourself, edit the config file as below:
```
email_address: "youraddress@gmail.com"
email_pwd: "abcdefghijklmnop"
email_server: 'smtp.gmail.com'
email_to: "youraddress@gmail.com"
```


Reports are run automatically every day by the [run reports](#run-all-reports) process, but you can also run ad-hoc reports in the [interactive diagnostics](#reports) tool. Ad hoc reports can be emailed or displayed on screen.

Full details of reports are given [here](#reports-1).