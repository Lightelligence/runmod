# Overview
Lightelligence projects use [Environment Modules](https://modules.readthedocs.io/en/latest/index.html) to handle environment configuration.
This allows us to create reproducible and somewhat isolated environments.

`runmod` is a wrapper around the `module` tool that dynamically loads modules in an isolated shell and then runs a command.
This allows for a more heterogeneous tool-suite because tools only have their own environment setup when they are being executed via modulefiles.
Using isolate environments reduces conflicts with other tool setups and environments.
We use `runmod` as a wrapper instead of directly running `module load` because loading modulefiles can have unintended side effects on the environment.
For example, some tools come prepackaged with their own version of certain libraries (C, python, C++, etc.) which can accidentally get loaded or used by other tools.

`runmod` invokes tools (e.g. Xcelium or Genus) in an isolated environment without the user running a `module load` command.
The `runmod` invocation creates a new sub-shell, does one or more `module load` in that shell, runs the user's commands, then terminates the sub-shell.
All the stdout and stderr is rendered to the shell that invoked `runmod`, but the original shell environment is unmodified.
This avoids interoperability problems between tools and keeps development environments clean.

# Installation
See the Environment Modules' documentation for [Linux](https://modules.readthedocs.io/en/latest/INSTALL.html) and for [Windows](https://modules.readthedocs.io/en/latest/INSTALL-win.html) for details.

`runmod` depends on the python [yaml](https://pypi.org/project/PyYAML/) package, which is not included in the Python Standard Library.
Additionally, there is a [cookiecutter](https://cookiecutter.readthedocs.io/en/stable/) template included in this repository for creating new modulefiles.
The cookiecutter template is optional, but is useful for automating the generation of new modulefiles.

# Environment Variable Dependencies
`runmod` depends on the [MODULESHOME](https://modules.readthedocs.io/en/latest/module.html#envvar-MODULESHOME) environment variable.

`runmod` can also optionally use an environment variable called `PROJ_DIR`.
`runmod` optionally uses `$PROJ_DIR` to point to a per-project `tools.yaml` file.
If `$PROJ_DIR` is not set, then `runmod` defaults to using `tools.yaml` in this repository.
It is highly recommended to use a per-project `tools.yaml` via `$PROJ_DIR` instead of modifying this repository's `tools.yaml`.

# Specifying module versions
`runmod` determines which modules to load based on a `tools.yaml` configuration file.
This repository includes a `tools.yaml` as a reference, but will not be usable out-of-the-box.
It will either need to be locally modified to point to reasonable default module versions, or superceded by a `tools.yaml` file in each project.
If a project defines its own `tools.yaml`, it must be located in the `env/` directory for that project.

`tools.yaml` contains a yaml dictionary where the first-level keys are canonical names for tool flows (e.g. `flowkit` for the Cadence digital back-end flow) and the values are another dictionary.
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

There is a shortcut for when the 'flow' or 'cannonical name' is the same name as the tool you want to invoke.

```runmod -t xrun -- -helpall``` in this example is equivalent to:

```
module load xcelium/1909 vmanager/1909
xrun -helpall
module unload xcelium/1909 vmanager/1909
```

In this case, `xrun` doesn't appear after the `-- ` in the `runmod` command line.
The `-t` implicitly adds `xrun` at the start of the command.

## Creating a module for a new tool

This repository contains a cookiecutter template for creating new modulefiles.
To create the first modulefile for a tool, run the following commands:
```
cd modulefiles
cookecutter template
<follow onscreen menu>
```

The first prompt will ask for the 'toolname'.
This is the name of the directory that will be created in `modulefiles/<toolname>`.
The second prompt will ask for the 'version'.
This is the modulefile that will be created in `modulefiles/<toolname>/<version>`.
`tools.yaml` will reference `<toolname>/<version>`, e.g. `xcelium/1909`.
The last prompt will ask for 'toolbase_version'.
This sets the 'toolbase' module version that is loaded in the newly created modulefile.
The 'toolbase' module, included in this repository, is provided as a place to put setup or environment variables that are common to all modules.
You will likely not need to modify the toolbase_version, so you can press 'enter' without typing anything for this prompt.

After running those commands, you will have a new `modulefiles/<toolname>` directory, with two files in it - `<version>` (e.g. `1909`) and `.version`.
`.version` sets the default version of the module, but it is recommended to always use a specific module version in `tools.yaml`.
You may need to modify the `<version>` file with tool-specific commands or environment file changes.

## Releasing a new modulefile for an existing tool

Releasing a new modulfile for an existing tool is simpler than creating one from scratch:
1. `cd` to the tool's directory in `modulefiles/`.
2. `cp` an existing version file (ideally the latest).
3. Modify the new file with the appropriate changes.
4. Validate the new modulefile and tool release
5. If necessary, update the .version file to change the default to the new tool verison.

For example, you could follow these steps to release a new version of Xcelium:

1. `cd modulefiles/xcelium`
2. `cp 1909 2103`
3. Modify `2103`
4. Validate Xcelium version 2103 by running DV regressions

### Editing existing modulefiles
It's generally considered bad practice to modify an existing modulefile.
In the overwhelming majority of cases, you should create a new modulefile for each tool release.
Modifying an existing modulefile impacts people already using those modulefiles, and your changes could disrupt their work.