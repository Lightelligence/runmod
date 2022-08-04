# Overview
Lightelligence projects use [environment-modules](https://modules.readthedocs.io/en/latest/index.html) to handle environment configuration.
This allows us to create reproducible and somewhat isolated environments.

`runmod` is a small wrapper around the `module` tool that dynamically loads modules and then runs a command.
This allows for a more heterogeneous tool-suite as tools only have their own environment setup when they are being executed, and do not conflict with other tool setups or environments.
We use `runmod` as a wrapper instead of directly running `module load` because loading modulefiles can have unintended side effects on the environment.
For example, some module-based tools come prepackaged with their own version of certain libraries (C, python, C++, etc.) which can accidentally get loaded or used by other tools.

`runmod` invokes module-based tools (e.g. Xcelium or Genus) in an isolated environment without the user running a `module load` command.
The `runmod` invocation creates a new sub-shell, does one or more `module load` in that shell, runs the user's commands, then terminates the sub-shell.
All the stdout and stderr is piped to the shell that invoked `runmod`, but the original shell environment is unmodified.
This avoids interoperability problems between tools and keeps development environments clean.

# Installation
FIXME - link to correct Environment Modules install
FIXME - describe python dependencies (yaml definitely, cookiecutter optional)

# Environment Variable Dependencies
FIXME list the environment variables needed to use runmod.

* MODULESHOME (which may be set by the Environment Modules installation)
* MODULEPATH
* PROJ_DIR

FIXME Maybe add a sample bash script to the github repo that sets up MODULEPATH

# Specifying module versions
`runmod` determines which modules to load based on a `tools.yaml` file.

FIXME need to put tools.yaml somewhere.

Each project has its own `tools.yaml` file in the `env/` directory for that project, and there is a `tools.yaml` in the EDA repository in env/ as well.
`tools.yaml` contains a yaml dictionary where the first-level keys are canonical names for tool flows (e.g. `flowkit` for the digital back-end flow) and the values are another dictionary.
The sub-dictionary currently only has a single key-value pair, where the required key is 'modules', and the value is an ordered list of modules to load for that tool flow.
A tool flow might only load one module (e.g. 'jg' for 'JasperGold', which only needs to load the 'jaspergold' module), but some might need to load many modules (e.g. `flowkit`, which loads modules for synthesis, pnr, lec, DFT, etc.).

## Examples:
If tools.yaml contains the following content:
``` yaml
xrun:
  modules:
    - xcelium/1909
    - vmanager/1909
jg:
  modules:
    - jaspergold
flowkit:
  modules:
    - genus/191
    - innovus/19-12
```

Then ```runmod flowkit -- genus -helpall``` command is equivalent to:
```
module load genus/191 innovus/19-12
genus -helpall
module unload genus/191 innovus/19-12
```

Note that the `--` is the separator between `runmod` arguments and command arguments.

There also exists a shortcut for when the 'flow' or 'cannonical name' is the same name as the tool you want to invoke.

```runmod -t xrun -- -helpall``` in this example is equivalent to:

```
module load xcelium/1909 vmanager/1909
xrun -helpall
module unload xcelium/1909 vmanager/1909
```

## Creating a module for a new tool

FIXME include the cookiecutter template
Use the cookiecutter template in the FIXME directory to create a new modulefile

```
cd modulefiles
cookecutter template
<follow onscreen menu>
```

## Releasing a new version of a tool with an existing modulefile
* *cd* to the appropriate directory in modulefiles
* *cp* and existing file (ideally the latest)
* Modify the new file to do the right thing.
* Once you've qualified the release, update the .version file in that directory to change the default to your new version.
### DO NOT EDIT EXISTING MODULE FILES
Well, only do it with extreme caution. Generally, you should create a new modulefile for each tool release. There are cases where we might want toe edit and existing one, but bumping the version number is not one of those reasons.
### Flows
Flows should also not be edited in place. Please create new versions when you want to upgrade the version of a tools. This allows users to easily swap back and forth between every variant of our environment.
#### Example
Releasing a new version of genus would be something like:

``` bash
cd modulefiles/genus
cp 2010 2011
<edit 2011>
<optionally edit .version file to point at 2011 if you want to change the default version>
```