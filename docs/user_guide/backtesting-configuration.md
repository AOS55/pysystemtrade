Configuration (`config`) objects determine how a system behaves. Configuration
objects are very simple; they have attributes which contain either parameters,
or nested groups of parameters.

### Creating a configuration object

There are 5 main ways to create a configuration object:

1. [Interactively from a dictionary](#creating-a-configuration-object-with-a-dictionary)
2. [By pulling in a YAML file](#creating-a-configuration-object-from-a-file)
3. [From a 'pre-baked' system](#creating-a-configuration-object-from-a-pre-baked-system)
4. [By joining together multiple configurations in a list](#creating-a-configuration-object-from-a-list)
5. [From a CSV file](#creating-configuration-files-from-csv-files)

#### Creating a configuration object with a dictionary

```python
from sysdata.config.configdata import Config

my_config_dict=dict(optionone=1, optiontwo=dict(a=3.0, b="beta", c=["a", "b"]), optionthree=[1.0, 2.0])
my_config=Config(my_config_dict)
```

There are no restrictions on what is nested in the dictionary, but if you
include arbitrary items like the above they won't be very useful!. The section
on [configuration options](backtesting-reference.md#configuration-options) explains what configuration
options would be used by a system.

#### Creating a configuration object from a file

This simple file will reproduce the useless config we get from a dictionary in
the example above.

```yaml
optionone: 1
optiontwo:
a: 3.0
b: "beta"
c:
   - "a"
   - "b"
optionthree:
- 1.0
- 2.0
```

!!! note

      As with python the indentation in a yaml file shows how things are
      nested. If you want to learn more about yaml check [this
      out](https://pyyaml.org/wiki/PyYAMLDocumentation#YAMLsyntax).

```python
from sysdata.config.configdata import Config
my_config=Config("private.filename.yaml") ## assuming the file is in "pysystemtrade/private/filename.yaml"
```

See [here](backtesting-process.md#file-names) for how to specify filenames in pysystemtrade.

In theory there are no restrictions on what is nested in the dictionary (but
the top level must be a dict); although it is easier to use str, float, int,
lists and dicts, and the standard project code only requires those (if you're a
PyYAML expert you can do other python objects like tuples, but it won't be
pretty).

You should respect the structure of the default config with respect to nesting, as
otherwise [the defaults](#defaults_how) won't be properly filled in.

The section on [configuration options](backtesting-reference.md#configuration-options) explains what
configuration options are available.


#### Creating a configuration object from a pre-baked system

```python
from systems.provided.futures_chapter15.basesystem import futures_system
system=futures_system()
new_config=system.config
```

Under the hood this is effectively getting a configuration from a .yaml file - 
[this one](https://github.com/robcarver17/pysystemtrade/blob/master/systems/provided/futures_chapter15/futuresconfig.yaml).

Configs created in this way will include all [the defaults
populated](#defaults_how).


#### Creating a configuration object from a list

We can also pass a list into `Config()`, where each item of the list contains a
dict or filename. For example we could do this with the simple filename example
above:

```python
from sysdata.config.configdata import Config

my_config_dict=dict(optionfour=1, optionfive=dict(one=1, two=2.0))
my_config=Config(["filename.yaml", my_config_dict])
```

!!! note
   
      If there are overlapping keynames, then those in latter parts of the list of configs will override earlier versions.
      This can be useful if, for example, we wanted to change the instrument weights *on the fly* but keep the rest of the configuration unchanged.

#### Creating configuration files from .csv files

Sometimes it is more convenient to specify certain parameters in a .csv file, then push them into a .yaml file. If you want to use this method then you can use these two functions:

```python
from sysinit.configtools.csvweights_to_yaml import instr_weights_csv_to_yaml  # for instrument weights
from sysinit.configtools.csvweights_to_yaml import forecast_weights_by_instrument_csv_to_yaml  # forecast weights for each instrument
from sysinit.configtools.csvweights_to_yaml import forecast_mapping_csv_to_yaml # Forecast mapping for each instrument
```

These will create .yaml files which can then be pasted into your existing configuration files.

<a name="defaults"> </a>

### Project defaults and private configuration

Many (but not all) configuration parameters have defaults which are used by the
system if the parameters are not in the object. These can be found in the
[defaults.yaml file](https://github.com/robcarver17/pysystemtrade/blob/master/systems/provided/futures_chapter15/futuresconfig.yaml). The section on
[configuration options](#Configuration_options) explains what the defaults are,
and where they are used.

!!! Warning
      I recommend that you do not change these defaults. It's better to use the
      settings you want in each system configuration file, or use a private configuration file if this is something you want to apply to all your backtests.

      If this file exists, `/private/private_config.yaml`, it will be used as a private configuration file.

      Basically, whenever a configuration object is added to a system, if there is a private config file then we add the elements from that. Then for any remaining missing elements we add the elements from the defaults.yaml.


<a name="config_function_defaults"> </a>

#### Handling defaults when you change certain functions

In certain places you can change the function used to do a particular
calculation, eg volatility estimation (This does *not* include trading rules -
the way we change the functions for these is quite different). This is
straightforward if you're going to use the same arguments as the original
argument. However if you change the arguments you'll need to change the project
defaults .yaml file. I recommend keeping the original parameters, and adding
new ones with different names, to avoid accidentally breaking the system.


<a name="defaults_how"> </a>

#### How the defaults and private configuration work


When added to a system the config class fills in parameters that are missing
from the original config object, but are present in (i) the private .yaml file and (ii) the default .yaml file. For
example if forecast_scalar is missing from the config, then the default value
of 1.0 will be used. This works in a similar way for top level config items
that are lists, str, int and float.

This will also happen if you miss anything from a dict within the config (eg if
`config.forecast_div_mult_estimate` is a dict, then any keys present in this
dict in the default .yaml, but not in the config will be added). Finally it
will work for nested dicts, eg if any keys are missing from
`config.instrument_weight_estimate['correlation_estimate']` then they'll be filled
in from the default file. If something is a dict, or a nested dict, in the
config but not in the default (or vice versa) then values won't be replaced and
bad things could happen. It's better to keep your config files, and the default
file, with matching structures (for the items you want to change at least!). Again this is a good argument for adding new
parameters, and retaining the original ones.

Note this means that the config before, and after, it goes into a system object
will probably be different; the latter will be populated with defaults.

```python
from sysdata.config.configdata import Config
my_config=Config()
print(my_config) ## empty config
```

``` bash
 Config with elements:
```

Now within a system:

```python
from systems.provided.futures_chapter15.basesystem import futures_system
system=futures_system(config=my_config)

print(system.config) ## full of defaults.
print(my_config) ## same object
```

```
 Config with elements: average_absolute_forecast, base_currency, buffer_method, buffer_size, buffer_trade_to_edge, forecast_cap, forecast_correlation_estimate, forecast_div_mult_estimate, forecast_div_multiplier, forecast_scalar, forecast_scalar_estimate, forecast_weight_estimate, instrument_correlation_estimate, instrument_div_mult_estimate, instrument_div_multiplier, instrument_weight_estimate, notional_trading_capital, percentage_vol_target, use_SR_costs, use_forecast_scale_estimates, use_forecast_weight_estimates, use_instrument_weight_estimates, volatility_calculation
```

Note this isn't enough for a working trading system as trading rules aren't
populated by the defaults:


```python
system.accounts.portfolio()
```

```bash
# deleted full error trace
Exception: A system config needs to include trading_rules, unless rules are passed when object created
```


### Viewing configuration parameters

Regardless of whether we create the dictionary using a yaml file or
interactively, we'll end up with a dictionary. The keys in the top level
dictionary will become attributes of the config. We can then use dictionary
keys or list positions to access any nested data. For example using the simple
config above:

```python
my_config.optionone
my_config.optiontwo['a']
my_config.optionthree[0]
```


### Modifying configuration parameters

It's equally straightforward to modify a config. For example using the simple
config above:

```python
my_config.optionone=1.0
my_config.optiontwo['d']=5.0
my_config.optionthree.append(6.3)
```

You can also add new top level configuration items:

```python
my_config.optionfour=20.0
setattr(my_config, "optionfour", 20.0) ## if you prefer
```

Or remove them:

```python
del(my_config.optionone)
```

With real configs you need to be careful with nested parameters:


```python
config.instrument_div_multiplier=1.1 ## not nested, no problem

## Heres an example of how you'd change a nested parameter
## If the element doesn't yet exist in your config
## If the element did exist, then obviously doing this would overwrite all other parameters in the config - so don't do it!

config.volatility_calculation=dict(days=20)

## If it does exist you can do this instead:
config.volatility_calculation['days']=20
```

This is especially true if you're changing the config that has been included within a system, which
will already include all the defaults:

```python
system.config.instrument_div_multiplier=1.1 ## not nested, no problem

## If we change anything that is nested, we need to change just one element to avoid clearing the defaults:
# So, do this:
system.config.volatility_calculation['days']=20

# Do NOT do this:
# system.config.volatility_calculation=dict(days=20)
```


### Using configuration in a system

Once we're happy with our configuration we can use it in a system:

```python
from systems.provided.futures_chapter15.basesystem import futures_system
system=futures_system(config=my_config)
```
Note it's only when a config is included in a system that the private configuration and defaults are populated.


### Including your own configuration options

If you develop your own stages or modify existing ones you might want to
include new configuration options. Here's what your code should do:


```python

## Assuming your config item is called my_config_item; in the relevant method:

    parameter=system.config.my_config_item

    ## You can also use nested configuration items, eg dict keyed by instrument_code (or nested lists)
    parameter=system.config.my_config_dict[instrument_code]

    ## Lists also work.

    parameter=system.config.my_config_list[1]

    ## (Note: it's possible to do tuples, but the YAML is quite messy. So I don't encourage it.)


```

You would then need to add the following kind of thing to your config file:

```yaml
my_config_item: "ni"
my_config_dict:
   US10: 45.0
   US5: 0.10
my_config_list:
   - "first item"
   - "second item"
```

Similarly if you wanted to use project defaults for your new parameters you'll
also need to include them in the [defaults.yaml
file](https://github.com/robcarver17/pysystemtrade/blob/master/systems/provided/futures_chapter15/futuresconfig.yaml). Make sure you understand [how the
defaults work](#defaults_how).


<a name="save_config"> </a>

### Saving configurations

You can also save a config object into a yaml file:

```python
from systems.provided.futures_chapter15.basesystem import futures_system
import yaml
from syscore.fileutils import resolve_path_and_filename_for_package

system = futures_system()
my_config = system.config

## make some changes to my_config here

filename = resolve_path_and_filename_for_package("private.this_system_name.config.yaml")

with open(filename, 'w') as outfile:
   outfile.write(yaml.dump(my_config, default_flow_style=True))
```

This is useful if you've been playing with a backtest configuration, and want
to record the changes you've made. Note this will save trading rule functions
as functions; this may not work and it will also be ugly. So you should use
strings to define rule functions (see [rules](backtesting-stages.md#rules) for more information)

You can also save the final optimised parameters into fixed weights for live trading:


```python
# Assuming system already contains a system which has estimated values
from systems.diagoutput import systemDiag

sysdiag = systemDiag(system)
sysdiag.yaml_config_with_estimated_parameters('someyamlfile.yaml',
   attr_names=['forecast_scalars',
   'forecast_weights',
   'forecast_div_multiplier',
   'forecast_mapping',
   'instrument_weights',
   'instrument_div_multiplier']
)

```
Change the list of attr_names depending on what you want to output. You can then merge the resulting .yaml file into your simulated .yaml file. Don't forget to turn off the flags for `use_forecast_div_mult_estimates`,`use_forecast_scale_estimates`,`use_forecast_weight_estimates`,`use_instrument_div_mult_estimates`, and `use_instrument_weight_estimates`.  You don't need to change flag for forecast mapping, since this isn't done by default.


### Modifying the configuration class

It shouldn't be necessary to modify the configuration class since it's
deliberately lightweight and flexible.