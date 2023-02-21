# Dataclass Binder

Library to bind TOML data to Python dataclasses in a type-safe way.


## Features

Currently it has the following properties that might set it apart from other data binding libraries:

- requires Python 3.10+
- relies only on dataclasses from the Python standard library
- detailed error messages which mention location, expected data and actual data
- strict parsing which considers unknown keys to be errors
- support for durations (`timedelta`)
- support for immutable (frozen) dataclasses
- can bind data from files, I/O streams or pre-parsed dictionaries
- can generate configuration templates from dataclass definitions

This library was originally designed for parsing configuration files.
As TOML's data model is very similar to JSON's, adding support for JSON in the future would be an option and would make the library useful for binding HTTP API requests.


## Maturity

This library is fully type-checked, has unit tests which provide 100% branch coverage and is used in production, so it should be reliable.

The API might still change in incompatible ways until the 1.0 release.
In particular the following aspects are subject to change:

- use of key suffixes for `timedelta`: this mechanism doesn't work for arrays
- the handling of separators in keys: currently `-` in TOML is mapped to `_` in Python and `_` is forbidden in TOML; most applications seem to accept both `-` and `_` in TOML instead


## Why Dataclasses?

A typical TOML, JSON or YAML parser returns the parse results as a nested dictionary.
You might wonder why you would want to use a data binding library rather than just getting the values directly from that dictionary.

Let's take the following example code for a service that connects to a database using a connection URL configured in a TOML file:

```py
import tomllib  # or 'tomli' on Python <3.11


def read_config() -> dict:
    with open("config.toml", "rb") as f:
        config = tomllib.load(f)
    return config

def handle_request(config: dict) -> None:
    url = config["database-url"]
    print("connect to database:", url)

config = read_config()
...
handle_request(config)
```

If the configuration is missing a `database-url` key or its value is not a string, this service would start up without complaints and then fail when the first requests comes in.
It would be better to instead check the configuration on startup, so let's add code for that:

```py
def read_config():
    with open("config.toml", "rb") as f:
        config = tomllib.load(f)

    url = config["database-url"]
    if not isinstance(url, str):
        raise TypeError(
            f"Value for 'database-url' has type '{type(url).__name__}', expected 'str'"
        )

    return config
```

Imagine you have 20 different configurable options: you'd need this code 20 times.

Now let's assume that you use a type checker like `mypy`.
Inside `read_config()`, the type checker will know that `url` is a `str`, but if you fetch the same value elsewhere in the code, that information is lost:

```py
def handle_request(config: dict) -> None:
    url = config["database-url"]
    reveal_type(url)
    print("connect to database:", url)
```

When you run `mypy` on this code, it will output 'Revealed type is "Any"'.
Falling back to `Any` means type checking will not be able to find type mismatches and autocomplete in an IDE will not work well either.

Declaring the desired type in a dataclass solves both these issues:
- the type can be verified at runtime before instantiating the dataclass
- tooling knows the type when you read the value from the dataclass

Having the dataclass as a central and formal place for defining the configuration format is also an advantage.
For example, it enables automatic generation of a documented configuration file template.


## Usage

The `dataclass_binder` module contains the `Binder` class which makes it easy to bind TOML data, such as a configuration file, to Python [dataclasses](https://docs.python.org/3/library/dataclasses.html).

The binding is a two-step process:
- specialize the `Binder` class by using your top-level dataclass as a type argument
- call the `parse_toml()` method, providing an I/O stream for the configuration file as its argument

Put together, the code looks like this:

```py
import logging
import sys
from pathlib import Path

from dataclass_binder import Binder


logger = logging.getLogger(__name__)

if __name__ == "__main__":
    config_file = Path("config.toml")
    try:
        config = Binder[Config].parse_toml(config_file)
    except Exception as ex:
        logger.critical("Error reading configuration file '%s': %s", config_file, ex)
        sys.exit(1)
```

### Binding a Pre-parsed Dictionary

If you don't want to bind the contents of a full file, there is also the option to bind a pre-parsed dictionary instead.
For this, you can use the `bind()` method on the specialized `Binder` class.

For example, the following service uses a hybrid configuration format where a single file configures both the service itself and logging system:

```py
import logging.config

import tomllib  # or 'tomli' on Python <3.11
from dataclass_binder import Binder


with open("config.toml", "rb") as f:
    config = tomllib.load(f)
service_config = Binder[ServiceConfig].bind(config["service"])
logging.config.dictConfig(config["logging"])
```

To keep these examples short, from now on `import` statements will only be included the first time a particular imported name is used.

### Basic Types

Dataclass fields correspond to TOML keys. In the dataclass, underscores are used as word separators, while dashes are used in the TOML file. For example, the following TOML fragment:

```toml
database-url = 'postgresql://user:password@host/db'
port = 5432
```

can be bound to the following dataclass:

```py
from dataclasses import dataclass

@dataclass
class Config:
    database_url: str
    port: int
```

Fields can be made optional by assigning a default value. Using `None` as a default value is allowed too:

```py
@dataclass
class Config:
    verbose: bool = False
    webhook_url: str | None = None
```

The `float` type can be used to bind floating point numbers.
Support for `Decimal` is not there at the moment but would be relatively easy to add, as `tomllib`/`tomli` has an option for that.

### Dates and Times

TOML handles dates and timestamps as first-class values.
Date, time and date+time TOML values are bound to `datetime.date`, `datetime.time` and `datetime.datetime` Python objects respectively.

There is also support for time intervals using `datetime.timedelta`:

```py
from datetime import timedelta

@dataclass
class Config:
    retry_after: timedelta
    delete_after: timedelta
```

Intervals shorter than a day can be specified using a TOML time value.
Longer intervals are supported by adding an `-hours`, `-days`, or `-weeks` suffix.
Other supported suffixes are `-minutes`, `-seconds`, `-milliseconds` and `-microseconds`, but these are there for completeness sake and less likely to be useful.
Here is an example TOML fragment corresponding to the dataclass above:

```toml
retry-after = 00:02:30
delete-after-days = 30
```

### Collections

Lists and dictionaries can be used to bind TOML arrays and tables.
If you want to make a `list` or `dict` optional, you need to provide a default value via the `default_factory` mechanism as usual, see the [dataclasses documentation](https://docs.python.org/3/library/dataclasses.html#mutable-default-values) for details.

```py
from dataclasses import dataclass, field

@dataclass
class Config:
    tags: list[str] = field(default_factory=list)
    limits: dict[str, int]
```

The dataclass above can be used to bind the following TOML fragment:

```toml
tags = ["production", "development"]
limits = {ram-gb = 1, disk-gb = 100}
```

An alternative to `default_factory` is to use a homogeneous (single element type) tuple:

```py
@dataclass
class Config:
    tags: tuple[str, ...] = ()
    limits: dict[str, int]
```

Heterogeneous tuples are supported too: for example `tuple[str, bool]` binds a TOML array that must always have a string as its first element and a Boolean as its second and last element.
It is generally clearer though to define a separate dataclass when you need more than one value to configure something:

```py
@dataclass
class Webhook:
    url: str
    token: str

@dataclass
class Config:
    webhooks: tuple[Webhook, ...] = ()
```

The extra keys (`url` and `token` in this example) provide the clarity:

```
webhooks = [
    {url = "https://host1/notify", token = "12345"},
    {url = "https://host2/hook", token = "frperg"}
]
```

TOML's array-of-tables syntax can make this configuration a bit easier on the eyes:

```
[[webhooks]]
url = "https://host1/notify"
token = "12345"

[[webhooks]]
url = "https://host2/hook"
token = "frperg"
```

Always define additional dataclasses at the module level in your Python code: if the class is for example defined inside a function, the `Binder` specialization will not be able to find it.

### Plugins

To select plugins to activate, you can bind Python classes or modules using `type[BaseClass]` and `types.ModuleType` annotations respectively:

```py
from dataclasses import dataclass, field
from types import ModuleType

from supertool.plugins import PostProcessor


@dataclass
class PluginConfig:
    postprocessors = tuple[type[PostProcessor], ...] = ()
    modules: dict[str, ModuleType] = field(default_factory=dict)
```

In the TOML, you specify Python classes or modules using their fully qualified names:

```toml
postprocessors = ["supertool_addons.reporters.JSONReport"]
modules = {lint = "supertool_addons.linter"}
```

There is no mechanism yet to add configuration to be used by the plugins.

### Immutable

If you prefer immutable configuration objects, you can achieve that using the `frozen` argument of the `dataclass` decorator and using abstract collection types in the annotations. For example, the following dataclass will be instantiated with a `tuple` object for `tags` and an immutable dictionary view for `limits`:

```py
from collections.abc import Mapping, Sequence


@dataclass(frozen=True)
class Config:
    tags: Sequence[str] = ()
    limits: Mapping[str, int]
```

### Generating a Configuration Template

To provide users with a starting point for configuring your application/service, you can automatically generate a configuration template from the information in the dataclass.

For example, when the following dataclass defines your configuration:

```py
@dataclass
class Config:
    database_url: str
    """The URL of the database to connect to."""

    port: int = 12345
    """TCP port on which to accept connections."""
```

You can generate a template configuration file using:

```py
from dataclass_binder import format_template


for line in format_template(Config):
    print(line)
```

Which will print:

```toml
# The URL of the database to connect to.
# Mandatory.
database-url = '???'

# TCP port on which to accept connections.
# Default:
# port = 12345
```

It is also possible to provide placeholder values by passing a dataclass instance rather than the dataclass itself to `format_template()`:

```py
TEMPLATE = Config(
    database_url="postgresql://<username>:<password>@<hostname>/<database name>",
    port=8080,
)

for line in format_template(TEMPLATE):
    print(line)
```

Which will print:

```toml
# The URL of the database to connect to.
# Mandatory.
database-url = 'postgresql://<username>:<password>@<hostname>/<database name>'

# TCP port on which to accept connections.
# Default:
# port = 12345
port = 8080
```

### Troubleshooting

Finally, a troubleshooting tip: instead of the full `Binder[Config].parse_toml()`, first try to execute only `Binder[Config]`.
If that fails, the problem is in the dataclass definitions.
If that succeeds, but the `parse_toml()` call fails, the problem is that the TOML file does not match the format defined in the dataclasses.


## Development Environment

[Poetry](https://python-poetry.org/) is used to set up a virtual environment with all the dependencies and development tools that you need:

    $ cd dataclass-binder
    $ poetry install

You can activate a shell which contains the development tools in its search path:

    $ poetry shell

We recommend setting up pre-commit hooks for Git in the `dataclass-binder` work area.
These hooks automatically run a few simple checks and cleanups when you create a new commit.
After you first set up your virtual environment with Poetry, run this command to install the pre-commit hooks:

    $ pre-commit install


## Changelog

### 0.1.0 - 2023-02-21:

- First open source release
