# Project Generator

[TOC]

## Introduction

The main purpose of this program is to automatically generate a basic project source code framework with a runnable and extendable UVM verification environment, based on a top module verilog RTL file, which could greatly save the effort and time for the initial setup work of a new project. 

## How to Use

To get help on this program:

> ./proj_gen.rb -h

Usage:   ./proj_gen.rb -n PROJ_NAME -f MODULE_FILE [-t TOP_MODULE] -o OUTPUT_DIR

Example: ./proj_gen.rb -n sample -f sample.v.erb -o ./sampleProj

* -n, --name PROJ_NAME        Specify the project name
* -f, --file MODULE_FILE         Specify the module file
* -t, --top TOP_MODULE         Specify the top module (optional)
* -o, --output OUTPUT_DIR    Specify the output directory
* -h, --help                                  Display this help screen

Note: [-t TOP_MODULE] is optional, default is the 1st module in MODULE_FILE

### Examples

Specify project name "sample", RTL file "sample.v.erb", top module "sample", and generate to dir "./sample":
> ./proj_gen.rb -n sample -f sample.v.erb -t sample-o ./sample

Could omit top module if "sample" is the 1st module defined in RTL file "sample.v.erb":
> ./proj_gen.rb -n sample -f sample.v.erb -o ./sample

------

## Exploring the Sample Project

In this section we explore the generated sample project.

### Flow and File Structure

Before running any tasks, the original clean project file structure is:

```
▾ ip/
▾ meta/
  ▾ config/
      full.cfg
  ▾ suites/
      base.rb
▾ design/
  ▾ rtl/
      sample.v.erb
▾ verif/
  ▾ tb/
      sample_tb_top.sv
  ▾ uvc/
    ▾ sample/
        sample_agent.sv
        sample_drv.sv
        sample_env.sv
        sample_if.sv
        sample_item.sv
        sample_mon.sv
        sample_pkg.sv
        sample_scoreboard.sv
        sample_seq_lib.sv
        sample_test_lib.sv.erb
  rakefile
```

### Meta files

#### config

The sample project supports different configurations. 

Configuration variables are defined in config meta files ('meta/config/*.cfg'), and utilized by task publish. See section publish[config] for details.

#### suites (and tests)

All the suite and test information is defined under directory 'meta/suites/*.rb'.

The tests in the same suite are included in one suite file. Let us take a look at 'base' suite ('meta/suites/base.rb') for example:

```ruby
suite :base

  test :sample_base_test
    desc    "The base test for all other tests"
    owner   :smith
    config  :full
    sanity  true
    regress true
    debug   false
  endtest

endsuite
```

Every test contains the following information: 

| Item    | Description                                                 |
| ------- | ----------------------------------------------------------- |
| test    | name of test, in symbol (a string starting with ':')        |
| desc    | description                                                 |
| owner   | owner, in symbol                                            |
| config  | config, in symbol                                           |
| sanity  | indicates the test belongs to sanity (false by default)     |
| regress | indicates the test belongs to regression (false by default) |
| debug   | indicates the test is in debugging (false by default)       |

*Note: The suites are described in DSL (Domain Specific Language).*

### Tasks

To get the full available task list, just change to sample project directory and type:

> rake -T

Normally the task list looks like:

```
rake clean                 # clean output
rake compile[config,dbg]   # compile
rake ip                    # get IP code (if any)
rake publish[config]       # publish files
rake regress               # run regression
rake run[dbg]              # run case
rake sanity                # run sanity
rake sim[case,config,dbg]  # simulation
rake verdi[config]         # open verdi
```

#### Dependency

Note that the tasks above have dependency relationship as:

```
ip <= publish <= compile <= sim
```

It means higher dependency tasks (right ones) rely on lower dependency tasks (left ones), and will automatically execute dependent tasks if necessary.

For example, if you execute

> rake run

from scratch, it will automatically run tasks 'ip', 'publish', and 'compile' before that.

**Benefit:** program takes care of all the underlying dependency issues so that user only needs to focus on target task.

#### ip

Pay attention to task

> rake ip

IPs are developed as separate projects and organized as nested submodules in this sample project.

**Benefit:** a centralized IP is much easier to maintain than the the same IP scattered in different projects with different versions.

#### publish[config]

All the code is published to 'out/<config>/src' directory before use (compile/sim). So the project file structure after 

> rake publish[config]

is:

```
▸ ip/
▸ meta/
▾ out/
  ▾ <config>/
    ▾ src/
      ▾ ip/
      ▸ design/
      ▸ verif/
▸ design/
▸ verif/
  rakefile
```

Note that you can specify the config meta file just as above when parsing all the files with extention .erb under 'out/src/design' &  'out/src/verif'.

If the config option is omitted in publish, a default config option :full is used, the same as

> rake publish[:full]

Then the sample project parses for instance this line in 'out/full/src/verif/uvc/sample/sample_test_lib.sv.erb':

```verilog
uvm_config_db#(int)::set(this,"sample_env0", "num_masters", <%= num_of_masters %>);
```

with the line in config meta file 'meta/config/full.cfg':

```ruby
num_of_masters = 1
```

and generates target file 'out/full/src/verif/uvc/sample/sample_test_lib.sv' with content:

```verilog
uvm_config_db#(int)::set(this,"sample_env0", "num_masters", 1);
```

**Benefit:** better flexibility and neater source code control.

#### compile[config,dbg]

A simulation snapshot will be created in 'out/<config>/sim/comp' directory for simulation of multiple cases,  after compile w.o/w. waveform:

> rake compile[config]

or

> rake compile[config, true]

```
▸ ip/
▸ meta/
▾ out/
  ▾ <config>/
    ▾ sim/
      ▾ comp/
        ▾ INCA_libs/
          ▸ irun.lnx8664.14.20.nc/
          ▸ irun.nc/
          ▸ worklib/
          irun_comp_sample.log
    ▸ src/
▸ design/
▸ verif/
  rakefile
```

#### sim[case,config,dbg]

Execute 

> rake sim[case, config]

or 
> rake sim[case, config, true]

for case simulation w.o/w. waveform. The sim log and waveform are stored in directory 'out/<config>/sim/<case>':

```
▸ ip/
▸ meta/
▾ out/
  ▾ <config>/
    ▾ sim/
      ▸ comp/
      ▾ <case>/
          irun.log
          novas_dump.log
          wave.fsdb
    ▸ src/
▸ design/
▸ verif/
  rakefile
```

If the case option is omitted in sim, a default case sample_base_test is used, the same as

> rake sim[:sample_base_test ]

**Benefit: ** multiple cases can run simultaneously based on the same snapshot to improve the simulation efficiency.

#### run[dbg, case_list]

To run a batch of cases w.o/w. waveform, execute:

> rake run[false, case1, case2, ...]

or

> rake run[true, case1, case2, ...]

#### verdi[config]

To open a verdi session for a specific config, just type:

> rake verdi[config]

#### sanity

Kick off the simulation for sanity tests (not including those ones in debug)

> rake sanity

#### regression

Kick off the simulation for regression tests (not including those ones in debug)

> rake regress

#### clean

To remove  'out' directory, execute:

> rake clean







