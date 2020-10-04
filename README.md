![ilexconf](https://raw.githubusercontent.com/vduseev/ilexconf/master/docs/img/logo.png)

<h2 align="center">Configuration Library 🔧 for Python</h2>

<p align="center">
<a href="https://travis-ci.org/vduseev/ilexconf"><img alt="Build Status" src="https://travis-ci.org/vduseev/ilexconf.svg?branch=master"></a>
<a href="https://coveralls.io/github/psf/black?branch=master"><img alt="Coverage Status" src="https://coveralls.io/repos/github/psf/black/badge.svg?branch=master"></a>
<a href="https://github.com/psf/black/blob/master/LICENSE"><img alt="License: MIT" src="https://black.readthedocs.io/en/stable/_static/license.svg"></a>
<a href="https://pypi.org/project/ilexconf/"><img alt="PyPI" src="https://img.shields.io/pypi/v/ilexconf"></a>
<a href="https://github.com/psf/black"><img alt="Code style: black" src="https://img.shields.io/badge/code%20style-black-000000.svg"></a>
</p>

`ilexconf` is a Python library to load and merge configs from multiple sources, access & change the values, and write them back, if needed. It has no dependencies by default but provides additional functions, relying on popular libraries to parse `yaml`, `toml`, provide `CLI` app, etc.

## Table of contents 

* <a href="#quick_start">🚀 Quick Start</a>
  * <a href="#quick_start_install">Installation</a>
  * <a href="#quick_start_create">Create `Config` object</a>
  * <a href="#quick_start_read">Read using `from_` functions</a>
  * <a href="#quick_start_access">Access values</a>
  * <a href="#quick_start_change_create">Change & create values</a>
  * <a href="#quick_start_merge">`merge` another `Mapping` into config</a>
  * <a href="#quick_start_as_dict">Convert to simple `dict` using `as_dict`</a>
  * <a href="#quick_start_write">Save to file using `to_` functions</a>
  * <a href="#quick_start_subclass">Subclass `Config` to customize</a>
* <a href="#internals">⚙️ Internals – How it Works</a>

<a id="quick_start"></a>
## Quick Start

<a id="quick_start_install"></a>
### Install

```shell
$ pip install ilexconf
```

<a id="quick_start_create"></a>
### Populate Config with values

Config object is initialized using arbitrary number of Mapping objects and keyword arguments. It can even be empty. 

```python
from ilexconf import Config, from_json

# All of these are valid methods to initialize a config
config = Config()
config = Config({ "database": { "connection": { "host": "test.local" } } })
config = Config(database__connection__port=4000)
config = from_json("settings.json")
config = from_env()

# Or, you can combine them
config = Config(
    # Take the basic settings from JSON file
    from_json("settings.json"),
    # Merge them with a dictionary
    { "database": { "connection": { "host": "test.local" } } },
    # Merge the keyword arguments on top
    database__connection__port=4000
)
```

When we initialize config all the values are merged. Every next argument is merged on top of the previous mapping values. And keyword arguments override even that.

For a settings file `settings.json` with the following content ...

```json
{
    "database": {
        "connection": {
            "host": "localhost",
            "port": 5432
        }
    }
}
```

The code above will produce a merged `config` with merged values:

```json
{
    "database": {
        "connection": {
            "host": "test.local",
            "port": 4000
        }
    }
}
```

<a id="quick_start_read"></a>
### Read from files & environment variables

Files like `.json`, `.yaml`, `.toml`, `.ini`, `.env`, `.py` as well as environment variables can all be read & loaded using a set of `from_` functions.

```python
from ilexconf import (
    from_json,      # from JSON file or string
    from_yaml,      # from YAML file or string
    from_toml,      # from TOML file or string
    from_ini,       # from INI file or string
    from_python,    # from .py module
    from_dotenv,    # from .env file
    from_env        # from environment variables
)

cfg1 = from_json("settings.json")

cfg2 = Config(
    from_yaml("settings.yaml"),
    from_toml("settings.toml")
)

cfg3 = Config(
    from_ini("settings.ini"),
    from_python("settings.py"),
    from_dotenv(".env"),
    from_env()
)
```

<a id="quick_start_access"></a>
### Access values however you like

You can access any key in the hierarchical structure using classical Python dict notation, dotted keys, attributes, or any combination of this methods.

```python
# Classic way
assert config["database"]["conection"]["host"] == "test.local"

# Dotted key
assert config["database.connection.host"] == "test.local"

# Attributes
assert config.database.connection.host == "test.local"

# Any combination of the above
assert config["database"].connection.host == "test.local"
assert config.database["connection.host"] == "test.local"
assert config.database["connection"].host == "test.local"
assert config.database.connection["host"] == "test.local"
```

<a id="quick_start_change_create"></a>
### Change existing values and create new ones

Similarly, you can set values of any key (_even if it doesn't exist in the Config_) using all of the ways above.

**Notice**, _contrary to what you would expect from the Python dictionaries, setting nested keys that do not exist is **allowed**_.

```python
# Classic way
config["database"]["connection"]["port"] = 8080
assert config["database"]["connection"]["port"] == 8080

# Dotted key (that does not exist yet)
config["database.connection.user"] = "root"
assert config["database.connection.user"] == "root"

# Attributes (also does not exist yet)
config.database.connection.password = "secret stuff"
assert config.database.connection.password == "secret stuff"
```

<a id="quick_start_merge"></a>
### Update with another Mapping object

If you just assign a value to any key, you override any previous value of that key.

In order to merge assigned value with an existing one, use `merge` method.

```python
config.database.connection.merge({ "password": "different secret" })
assert config.database.connection.password == "different secret"
```

<a id="quick_start_as_dict"></a>
### Represent as dictionary

For any purposes you might find fit you can convert entire structure of the Config object into dictionary, which will be essentially returned to you as a deep copy of the object.

```python
assert config.as_dict() == {
    "database": {
        "connection": {
            "host": "test.local",
            "port": 8080,
            "user": "root",
            "password": "different secret"
        }
    }
}
```

<a id="quick_start_write"></a>
### Write to file

You can serialize the file as JSON or other types any time using the `to_` functions.

```python
# Write updated config back as JSON file
from ilexconf import to_json

to_json(config, "settings.json")
```

**WARNING**: _This might throw a serialization error if any of the values contained in the Config are custom objects that cannot be converted to `str`. Also, obviously, you might not be able to correctly parse an object back, if it's saved to JSON as `MyObject(<function MyObject.__init__.<locals>.<lambda> at 0x108927af0>, {})` or something._

<a id="quick_start_subclass"></a>
### Subclass

Subclassing `Config` class is very convenient for implementation of your own config classes with custom logic.

Consider this example:

```python
import ilexconf

class Config(ilexconf.Config):
    """
    Your custom Configuration class
    """

    def __init__(self, do_stuff=False):
        # Initialize your custom config with JSON by default
        super().__init__(self, ilexconf.from_json("setting.json"))

        # Add some custom value depending on some logic
        if do_stuff:
            self.my.custom.key = "Yes, do stuff"

        self.merge({
            "Horizon": "Up"
        })

# Now you can use your custom Configuration everywhere
config = Config(do_stuff=True)
assert config.my.custom.key == "Yes, do stuff"
assert config.Horizon == "Up"
```

<a id="internals"></a>
## Internals

Under the hood `ilexconf` is implemented as a `defaultdict` where every key with Mapping value is represented as another `Config` object. This creates a hierarchy of `Config` objects.

`__getitem__`, `__setitem__`, `__getattr__`, and `__setattr__` methods are overloaded with custom logic to support convenient get/set approach presented by the library.

<a id="alternatives"></a>
## Alternative Libraries

Below is a primitive analysis of features of alternative libraries doing similar job.

| Library                           | **ilexconf** | dynaconf | python-configuration |
| --------------------------------- | ----- | -------- | -- |
| **Read from `.json`**             | x     | x        | x  |
| **Read from `.toml`**             | x     | x        | x  |
| **Read from `.ini`**              | x     | x        | x  |
| **Read from env vars**            | x     | x        | x  |
| **Read from `.py`**               |       | x        | x  |
| **Read from `.env`**              |       | x        | x  |
| **Read from dict object**         | x     |          | x  |
| **Read from Redis**               |       | x        |    |
| **Read from Hashicorp Vault**     |       | x        |    |
| **Default values**                | x     | x        |    |    
| **Multienvironment**              |       | x        |    |
| **Attribute access**              | x     | x        | x  |
| **Dotted key access**             | x     | x        | x  |
| **Merging**                       | x     | x        | x  |
| **Interpolation**                 |       | x        | x  |
| **Saving**                        | x     | x        |    |
| **CLI**                           | x     | x        |    |
| **Printing**                      | x     | x        |    |
| **Validators**                    |       | x        |    |
| **Masking sensitive info**        |       | x        | x  |
| **Django integration**            |       | x        |    |
| **Flask integration**             |       | x        |    |
| **Hot reload**                    |       |          |    |
| *Python 3.6*                      |       |          | x  |
| *Python 3.7*                      |       |          | x  |
| *Python 3.8*                      | x     |          | x  |

## Contributing

Contributions are welcome!

## Kudos

`ilexconf` heavily borrows from `python-configuration` library and is inspired by it.

## License

MIT