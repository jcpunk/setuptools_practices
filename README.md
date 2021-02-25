# Python Packaging Practices (Best?)

This document is focused on sustainable practices for python packaging as of Jan 2021.

The majority of systems deployed at FNAL use python-3.6 as their default python 3 implementation.  As a result, elements of PEP-518, PEP-621, and PEP-631 are supported.  However, the complete integration of `pyproject.toml` is not complete on python-3.6.

Additionally, you should consider other helpful tools (such as pre-commit) to automate some common workflows.

## Metadata

The python package metadata can be defined in a few ways.  We will focus on the `setup.py` options here for maximum accessibility and compatibility.

In python-3.9 and later, `importlib.metadata` can be used as a clean abstraction layer for direct access to the package attributes.  A backport is availible for earlier versions of python.

### about.py

Storing your package metadata in a consistant place makes introspection easier.  This example `about.py` provides a forward looking place to centralize these elements.

You should customize the Defaults listed here to meet your needs.

```python
"""
Store our reusable package metadata in a single location.
"""

__all__ = [
    "Name",
    "Version",
    "Author",
    "Author_Email",
    "Description",
    "License",
    "URL",
]

import sys

if float(str(sys.version_info[0]) + "." + str(sys.version_info[1])) < 3.9:
    from importlib_metadata import PackageNotFoundError
    from importlib_metadata import metadata as get_metadata
else:
    from importlib.metadata import PackageNotFoundError
    from importlib.metadata import metadata as get_metadata

parent_name = ".".join(__name__.split(".")[:-1])

# Defaults to permit self generation
Name = parent_name
Version = "DEVELOPMENT"
Author = None
Author_Email = None
Description = None
License = None
URL = None

try:
    _md = get_metadata(parent_name)
    Name = _md["Name"]
    Author = _md["Author"]
    Author_Email = _md["Author-Email"]
    Description = _md["Description"]
    License = _md["License"]
    URL = _md["URL"]
except PackageNotFoundError:
    pass

try:
    # This is built by setuptools_scm
    from .version import version as Version  # noqa: F401
except ImportError:
    pass
```

### \_\_version\__

PEP-396 notes that your module should provide a `__version__` attribute for review and troubleshooting.

This can be done in `__init__.py` as follows:

```python
from .about import Version as __version__  # noqa: F401
```

## File Layout

This is an example of a minimalist layout for a python module called MYPACKAGENAME.

```
├── config/
├── data/
├── doc/
├── LICENSE.txt
├── packaging/
│   ├── dpkg/
│   ├── README.md
│   ├── rpm/
│   └── systemd/
├── pyproject.toml
├── README.md
├── setup.py
├── src/
│   └── MYPACKAGENAME/
│       ├── __init__.py
│       ├── about.py
│       └── SUBMODULE/
│           └── __init__.py
└── tests/
    ├── __init__.py
    └── MYPACKAGENAME/
        ├── __init__.py
        ├── SUBMODULE/
        │   ├── __init__.py
        │   ├── test_SUBMODULE_behavior_a.py
        │   └── test_SUBMODULE.py
        ├── test_MYPACKAGENAME_behavior_a.py
        └── test_MYPACKAGENAME.py
```

We'll dive a bit into the purpose of the various directories first.  The `pyproject.toml` and `setup.py` will have their own specific discussion on usage and practices.

### config

Any config files your package needs should be placed here.

### data

Any static data files your package needs should be placed here.  See `setuptools` feature `package_data` for more information.

### doc

Your project documentation should be placed here.  Python has generally standardized around `sphinx` as the documentation tool of choice.  To reduce duplication, `sphinx` metadata can be extracted from `about.py` (see the `setup.py` example).

### packaging

Specific elements related to package distribution (not wheels) should be placed here.

Elements such as `systemd` services (which cannot be included in wheels) and packaging scriptlets (to make user accounts) can be centralized here.

### src/MYPACKAGENAME

Placing the module source under a `src` directory first creates a clear place to locate module code.  It also provides a consistant layout for automation across multiple projects.  If every project has a `src` directory, then you can isolate all module code with a trivial script and use a template for packaging.

You'll see we use this abstraction later on in `setup.py`.

### tests

Code should also include unit tests.  There are two places they could be put:

* Along side the code being tested.  This increases the visibility of the tests to the developers for making any changes.
* In a dedicated `tests` directory at the same level as the `src` directory.  This decreases the clutter for complex programs.

This is largly a matter of personal preference.  The python community generally tends towards placing the tests in a seperate directory.  Our examples here will follow this pattern.

## Getting Started With Setuptools

Setuptools is the expected method for packaging python modules with python-3.6.

The configuration of setuptools can come from a number of places.  We'll focus on `setup.py` and `pyproject.toml`.

The `setup.py` file is compatible with most python versions.  The `pyproject.toml` file is the most forward looking.

NOTE: The version of `pip` and `setuptools` that ships with EL7 does not support the `pyproject.toml` file.  Folks doing development on these platforms should run `python3 -m pip install --user --upgrade pip setuptools`.  This will permit use of modern practices and forward compatibility.

### setuptools\_scm

The `setuptools_scm` python package is a plugin to `setuptools` that will automatically version your package based on tags and limit the manual work required in releases.

It is strongly recommended and will be used the examples provided here.

### setup.py

The `setuptools` package does not yet support using `pyproject.toml` for all elements.

Example `setup.py`:

```python
import importlib
import os
import pathlib
import sys
from setuptools import setup, find_packages

here = pathlib.Path(__file__).parent.resolve()

sys.path.insert(0, str(here / "src"))
pkg_name = os.listdir("src")[0]
about = importlib.import_module(pkg_name + ".about")

# Get the long description from the README file
long_description = (here / "README.md").read_text(encoding="utf-8")

setup(
    setup_requires=[
        "setuptools >= 45",
        "wheel >= 0.36",
        "setuptools_scm[toml] >=3.4",
    ],  # noqa: E501
    name=about.Name,
    url=about.URL,
    author=about.Author,
    author_email=about.Author_Email,
    license=about.License,
    description=about.Description,
    long_description=long_description,
    long_description_content_type="text/markdown",
    package_dir={"": "src"},
    packages=find_packages(
        where="src", exclude=("*.tests", "*.tests.*", "build.*", "doc.*")
    ),
    use_scm_version={
        "version_scheme": "post-release",
        "write_to": str("src/" + pkg_name + "/version.py"),
    },
    install_requires=["importlib_metadata >= 3.4 if python_version < 3.9", ],
)
```

This is a very minimal `setup.py` that should work with our sample project layout.  With this you should be able to run `python3 setup.py bdist_wheel` to produce a trivial test wheel.

#### Adding Dependencies on Other Modules

To add dependencies we add extra parameters to the `setup()` function.

Setuptools breaks dependencies out into two types `install_requires` (aka your minimum requirements) and `extras_require`.

The `install_requires` argument takes a `list()` that can be passed into `pip`.  For example:

```python
    install_requires=["importlib_metadata >= 3.4 if python_version < 3.9", "requests >= 2.18"]
```

The `extras_require` argument takes a `dict()` of extra features and then a `list()` of modules required.

To simplify the developer expirence, you should add a `develop` feature with all the modules required for your unit tests.  For example:

```python
    extras_require={
        "develop": [
            "pytest >= 6.2",
            "pytest-flake8 >= 1.0",
            "pytest-timeout >= 1.4",
        ],
    },
```

#### Adding End User Scripts

Scripts for end users can be added with the `entry_points` command.  In particular the `console_scripts` option for native python code is of note.  Please review the upstream documentation on these.  When possible, `console_scripts` is preferable due to its ability to hook into native testing.

```python
    entry_points={
        "console_scripts": [
            "myscript=MYPACKAGENAME.utils.myscript:main",
            "myotherscript=MYPACKAGENAME.utils.myotherscript:main",
        ],
        "scripts": [
            "bin/myshellscript.sh",
            "bin/myotherscript.sh",
        ],
    },
```

#### Customizing RPM builds

The RPM generated by setuptools can be customized a number of ways.  In theory this snippet added to the arguments of `setup()` should provide a clear way to express what is being changed.

```python
    options={
        "bdist_rpm": {
            "build_requires": "python3",
            "install_script": "package/rpm/install_section",
            "post_install": "package/rpm/post_install_section",
            "post_uninstall": "package/rpm/post_uninstall_section",
            "requires": ["packagename", "otherpackage"],
        },
```

With this in place, `python3 setup.py bdist_rpm` should generate a usable RPM with your custom changes.  If the default templates are sufficient for you, no customization is required.

### pyproject.toml

Eventurally all the build specifics from `setup.py` will be available with `pyproject.toml`.  This is not the case today.

However, a number of useful defaults for development can still be set here.  In particular, `pytest` can read it settings from here.

```toml
[build-system]
requires = [ "setuptools >= 45", "wheel >= 0.36", "setuptools_scm[toml] >=3.4" ]
build-backend = "setuptools.build_meta"

[tool.pytest.ini_options]
minversion = "6.0"
addopts = "-l -v --durations=0 --tb=native"
log_level = "debug"
testpaths = "tests"

timeout = "300"

flake8-max-line-length = "120"
flake8-ignore = "E501 E303"
flake8-show-source = "True"
flake8-statistics = "True"
```

This will permit automatic detection of any unit tests listed under `tests/`, set a default test timeout of 300 seconds (if `pytest-timeout` is installed), and permit running `pytest -m flake8 --flake8` (if `pytest-flake8` is installed).

## Using a Setuptools Enabled Module

### For End Users

If you are not doing development, you should install a wheel or other supported package generated from the repo.

```shell
python3 setup.py bdist_wheel
```

The file generated (`dist/MYPACKAGENAME-VERSION.whl`) can be transported to any compatible python and installed directly via `python3 -m pip`.

```shell
python3 -m pip install --user MYPACKAGENAME-VERSION.whl
```

### For Developers

Developers will often want to make tweaks, run tests, make more tweaks, with minimal fuss.  This workflow should provide a quick route to development.

```shell
python3 setup.py develop --user
python3 -m pip install --user MYPACKAGENAME[develop]
```

If the templates here were followed, you should now have MYPACKAGENAME within your default `PYTHONPATH`, but that path will point to your git repo where you can make and test your changes.

As a developer, you may wish to perform the package building process (`bdist_wheel` / `bdist_rpm`) yourself and provide those files directly to your users.
