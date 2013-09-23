MultiRun
==========

A Matlab tool for facilitating parameter calibration of detim/debam.


v 0.0.7
-------

Usage
------
MultiRun provides a method for running the models over a range of
parameter values, as well as a helper class which manages each
individual run of the model, and ensures that the model is not
run more times than necessary.

To run the model multiple times the function ```modelMultiRun``` is available.

```matlab
function [hashes, status, err, changes, runs] = modelMultiRun(modelpath, basefile, varargin)

 Args: modelpath - fully qualified path to a model's executable
       basefile - fully qualified path to a valid config file for the
       model, this will be modified based upon the the list of key-value
       pairs passed to varargin
       varargin - a list of key-value pairs which are modified, e.g.
           modelMultiRun('debam', 'input.txt', 'icekons', [5:0.1:6])
         will run the model with icekons set to each value in [5:0.1:6]
 Returns: hashes - Cell array of hashes of each run
          status - status(i) = Array of return status of run with hash hashes{i}
          err - Error messages associated by incomplete runs
          changes - array of changes made to input.txt
          runs - a container.Maps indexed by hashes of HashedRun objects,
           each corresponding to a single model run.
```

### What you need:
  1. Compiled versions of DEBaM or DETIM.  The full directory path to the
     binary is passed as ```modelpath```.
  2. A valid ```input.txt``` file for said model. The config
     should __not__ perform any of the optimization routines;
     at the moment, MultiRun is pretty stupid, and won't turn this off when it
     generates configuration files for the model. It's a good idea to set as many
     parameters as possible in this file to the ones you'd like them to be
     in any single run of the model.
  3. Model inputs to go along with the model. See the model documentation for
     setting these up, in practice, you'll usually  have a single set of these
     for every model run.
  4. A list of parameters to be changed in the base config, and values they
     will take on for the model runs. The syntax for this should be familiar
     from, for example, Matlab's ```plot``` function.  If we would like the model
     to read from ```base_input.txt``` run ```debam``` and run every combination
     of parameters, where ```icekons``` takes on every value in ```[5.0 : 0.1 : 6.0]```
     and ```firnkons``` takes on every value in ```[350 : 0.1 : 351]```
     we would make the call:
     ```matlab
     runs = modelMultiRun("detim", "base_input.txt", "icekons", [5.0:0.1:6.0], "firnkons", [350:0.1:351]);
     ```
     This will result in one hundred model runs, wherein the configuration files passed to
     DETIM are identical to ```base_input.txt```, except that icekons and firnkons
     have been changed to match the values passed to modelMultiRun, and with
     output directories determined by a hashing function. In all, 100
     runs will occur, corresponding to the 100 pairings of an element of ```[5.0:0.1:6.0]```
     and ```[350:0.1:351]```.

### What happens:
For each run, MultiRun uses the helper class ```MultiRun.HashedRun``` to
generate a nearly-unique alpha-numeric identifier for that particular model run.
This is done by:
- Generating an ```input.txt``` file from the passed parameters,
- Taking the SHA-1 hash of the text in this file
- Modifying the ```outpath``` parameter of the ```containers.Map``` containing
  the configuration to ```<original-outpath>/<SHA hash>/outpath/```.

MultiModelRun returns several cell-arrays; entries with the same index correspond to the same model run
  1. ```hashes```: the SHA-1 hash of the ```input.txt``` configuration files
  2. ```status```: Returned status of the model run.
  3. ```err```: Sting detailing any errors encountered; is empty if none occur
  4. ```changes```: Strings listing any changes made to ```base_input``` for this run
  5. ```runs```: a ```containers.Map``` object, whose keys are the hashes of each run, and whose
    values are the HashedRun objects of each run.

- HashedRun checks to see whether or not there is already a config in the new output path.
  This is done by checking a lockfile ```runstatus.lockfile```, the contents of which
  tell whether or not this config has been run, is currently running, or is waiting to
  be run.
- If no lockfile is found, the new configuration file is written
  to disk at ```<original-outpath>/<SHA hash>/input.txt```.

- If a lockfile is found, it's status is returned and an error is posted.
- The model is run as a system subprocess, if any errors occur ```status[i]``` is
 set, and an error message is returned.
- In particular, if every entry in ```status``` is ```1```, we know that
 every model run exited successfully.

### E.g.
Suppose we run 
```matlab
basefile = '/home/luser/work/hock_mass_balance/config/base_input.txt';
modelpath = '/home/luser/local/bin/detim';

 [hashes, status, err, changes, runs] = MultiRun.modelMultiRun(modelpath, basefile, 'icekons', [5, 6.0], 'firnkons', [350, 351]);
```
at the Matlab repl. MultiRun will take the config file
from ```/home/luser/work/hock_mass_balance/config/base_input.txt```
and generate ```input.txt``` files which contain every combination of the parameters
```icekons``` and ```firnkons```, given the values you've assigned them
(in this case there are four combinations).  To decrease the possibility
that one run will overwrite another, each generated file is placed in a directory
with name determined by the SHA-1 hash of the modified config file,
and the model's output is directed to that folder as well.
This model run will result in a directory structure which looks like:

```
mytest/
└── output
    ├── a729f3b8529edf74e2d57cf64ba8cc91fc64907e
    │   └── outpath
    ├── cdad489ead2ec45681e9a5ec91516623c88433ce
    │   └── outpath
    ├── d5c3a35650dcde462cc5eaea301cfbb62cd050d9
    │   └── outpath
    └── fbf399de8a6ac388aad1647ad918d37f9a3d81f1
        └── outpath
```
Since the long alpha-numeric directory names can be difficult to traverse, especially
when there are lots of them, ```multiModelRun``` also writes a file ```changes.txt```
which lists the changes made to to the base configuration for that run, i.e.:
```
$ cd mytest/output/a729f3b8529edf74e2d57cf64ba8cc91fc64907e/
$ ls
changes.txt     input.txt      outpath        runstatus.lock
$ cat changes.txt
Base config file: /home/luser/Documents/work/hock_mass_balance/mytest/input.txt
New config file: /home/luser/Documents/work/hock_mass_balance/mytest/output/a729f3b8529edf74e2d57cf64ba8cc91fc64907e/index.txt
Changes made:
  icekons = 6
  firnkons = 350
```

Each run is managed by a helper class called ```HashedRun```. The ```HashedRun``` class
manages the run; building a directory structure for that run alone, and ensuring that
each configuration is only run once.


Installation
------------
1. Download and compile the latest version of the 
[Hock Melt Models](https://github.com/regine/meltmodel).
2. Download MultiRun.
3. Make sure the folder ```+MultiRun``` is in your Matlab Path.
4. Done.

License
-------
BSD
