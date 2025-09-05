---
jupyter:
  jupytext:
    text_representation:
      extension: .md
      format_name: markdown
      format_version: '1.3'
      jupytext_version: 1.16.0
  kernelspec:
    display_name: (xmmsas)
    language: python
    name: conda-env-xmmsas-py
---

# pySAS Tips, Tricks, and Special Features
<hr style="border: 2px solid #fadbac" />

- **Description:** A brief guide for some tips, tricks, and special features in pySAS.
- **Level:** Beginner
- **Data:** A random XMM observation (obsid=0556211001)
- **Requirements:** Must be run using the `HEASARCv6.35` (or higher) image, along with pySAS version 2.0.  Run in the <tt>(xmmsas)</tt> conda environment on Sciserver. You should see <tt>(xmmsas)</tt> at the top right of the notebook. If not, click there and select <tt>(xmmsas)</tt>.
- **Credit:** Ryan Tanner (August 2025)
- **Support:** <a href="https://heasarc.gsfc.nasa.gov/docs/xmm/xmm_helpdesk.html">XMM Newton GOF Helpdesk</a>
- **Last verified to run:** 23 August 2025, for SAS v22.1 and pySAS v2.0

<hr style="border: 2px solid #fadbac" />


## Introduction
This tutorial demonstrates several tips, tricks, and special features available in pySAS. This notebook assumes you are at least minimally familiar with pySAS on SciServer (see the [Long pySAS Introduction](./analysis-xmm-long-intro.md "Long pySAS Intro")). 

#### Useful Links

- [`pysas` Documentation](https://xmm-tools.cosmos.esa.int/external/sas/current/doc/pysas/index.html "pysas Documentation")
- [`pysas` on GitHub](https://github.com/XMMGOF/pysas)
- [Common SAS Threads](https://www.cosmos.esa.int/web/xmm-newton/sas-threads/ "SAS Threads")
- [Users' Guide to the XMM-Newton Science Analysis System (SAS)](https://xmm-tools.cosmos.esa.int/external/xmm_user_support/documentation/sas_usg/USG/SASUSG.html "Users' Guide")
- [The XMM-Newton ABC Guide](https://heasarc.gsfc.nasa.gov/docs/xmm/abc/ "ABC Guide")
- [XMM Newton GOF Helpdesk](https://heasarc.gsfc.nasa.gov/docs/xmm/xmm_helpdesk.html "Helpdesk") - Link to form to contact the GOF Helpdesk.

<div style='color: #333; background: #ffffdf; padding:20px; border: 4px solid #fadbac'>
<b>Running On Sciserver:</b><br>
When running this notebook inside Sciserver, make sure the HEASARC data drive is mounted when initializing the Sciserver compute container. <a href='https://heasarc.gsfc.nasa.gov/docs/sciserver/'>See details here</a>.
<br><br>
<b>Running Outside Sciserver:</b><br>
This notebook was designed to run on SciServer, but an equivelent notebook <a href="https://github.com/XMMGOF/pysas_docs">can be found on GitHub</a>. You will need to install the development version of pySAS found on GitHub (<a href="https://github.com/XMMGOF/pysas">pySAS on GitHub</a>). There are installation instructions on GitHub. 
<br>
</div>

<div class="alert alert-block alert-warning">
    <b>Warning:</b> By default this notebook will place observation data files in your <tt>scratch</tt> space. The <tt>scratch</tt> space on SciServer will only retain files for 90 days. If you wish to keep the data files for longer move them into your <tt>persistent</tt> directory.
</div>

```python
# pySAS imports
import pysas
from pysas.sastask import MyTask
import os

# To get your user name. Or you can just put your user name in the path for your data.
from SciServer import Authentication as auth
usr = auth.getKeystoneUserWithToken(auth.getToken()).userName

data_dir = os.path.join('/home/idies/workspace/Temporary/',usr,'scratch/xmm_data')
```

## Smart Inputs

A new feature in version 2.0 of pySAS is how it handles a range of input types. For example, the Obs ID can be either a string or a number. If it is a number it will add the leading zeros to make it 10 digits long.

```python
obsid = '0556211001'
my_obs = pysas.obsid.ObsID(obsid,data_dir=data_dir)
```

```python
obsid = 556211001
my_obs = pysas.obsid.ObsID(obsid,data_dir=data_dir)
```

<!-- #region -->
The same is true for input parameter values. For example
```python
inargs = {'spectrumset' : 'R1_spectra.fits',
          'rmfset'      : 'rmf1_file.fits',
          'evlist'      : 'R1_event_list',
          'emin'        : 0.4,
          'emax'        : 2.5,
          'rows'        : 4000}

MyTask('rgsrmfgen', inargs).run()
```
Also in pySAS v2.0 'yes/no' parameters can be passed in as a Python boolean:
```python
inargs = {'table'            : 'event_list.fits', 
          'withfilteredset'  : True, 
          'expression'       : "'(PATTERN <= 12)&&(PI in [200:4000])&&#XMMEA_EM'", 
          'filteredset'      : 'filtered_event_list.fits', 
          'filtertype'       : 'expression', 
          'keepfilteroutput' : True, 
          'updateexposure'   : True, 
          'filterexposure'   : True}

MyTask('evselect', inargs).run()
```
---

Because the inputs are stored as a dictionary, if a single input needs to be changed and the same SAS task run again the user could simply use:
```python
inargs['expression'] = "'(PATTERN <= 12)&&(PI in [4000:12000])&&#XMMEA_EM'"
MyTask('evselect', inargs).run()
```

---
Alternatively the input argument dictionary could be created like this:
```python
inargs = {}
inargs['table']            = 'event_list.fits'
inargs['withfilteredset']  = 'yes'
inargs['expression']       = "'(PATTERN <= 12)&&(PI in [200:4000])&&#XMMEA_EM'"
inargs['filteredset']      = 'filtered_event_list.fits'
inargs['filtertype']       = 'expression'
inargs['keepfilteroutput'] = 'yes'
inargs['updateexposure']   = 'yes'
inargs['filterexposure']   = 'yes'

MyTask('evselect', inargs).run()
```
Both are valid ways of creating a dictionary in Python.

Previous versions of pySAS required the input arguments to be collected in a list. This option is still available. Each input needs to be a single string in the list and be of the form, 'parameter=value'.

For example,
```python
inargs = ['table=event_list.fits', 
          'withfilteredset=yes', 
          "expression='(PATTERN <= 12)&&(PI in [200:4000])&&#XMMEA_EM'", 
          'filteredset=filtered_event_list.fits', 
          'filtertype=expression', 
          'keepfilteroutput=yes', 
          'updateexposure=yes', 
          'filterexposure=yes']

MyTask('evselect', inargs).run()
```
---
The user can get all the default parameters for any SAS task by using:
```python
task_name = 'insert_task_name'
inargs = pysas.param.get_input_params(task_name)
```

This will return a special obejct that behaves just like a normal Python dictionary with all possible input arguments for a given SAS task, along with their default values. The user can modify the values in this dictionary and pass it back in to `MyTask`.

<div class="alert alert-block alert-warning">
    <b>Note:</b> This is still experimental. We have tested this, but it is possible to run into some unexpected behavior. We are aware of at least one (!) case with unexpected behavior for an uncommonly used SAS task.
</div>
<!-- #endregion -->

## Default pySAS Data Directory Structure

pySAS assumes your XMM data is kept in a single directory `data_dir`. For example,
```
data_dir = '/path/to/data_dir/'
```
If you are running pySAS on SciServer the path to `data_dir` may be somthing like: `/home/idies/workspace/Temporary/rjtanner/scratch/xmm_data/`. If you are running pySAS on your local machine the path may be something like: `/home/rtanner/xmm_stuff/xmm_data/`.

Once the `data_dir` is set pySAS will download data files for individual Obs IDs into their own directory. With data from multiple Obs IDs your `data_dir` would look like this:

```
└── data_dir
    ├── 0104860501
    ├── 0112200301
    ├── 0123700101
    ├── 0400550201
    ├── 0790830101
    ├── ...
```

The directory for each individual Obs ID *can* (but it doesn't *have* to) contain subdirectories for `ODF` and `PPS` files, and a `work` directory. A single Obs ID directory may have subdirectories for just `ODF` files and a `work` directory, or a `PPS` directory and a `work` directory, or all three directories, depending on what level of data files you downloaded. The overall structure might look something like this:

```
└── data_dir
    ├── 0104860501
    │   ├── ODF
    │   ├── PPS
    │   └── work
    ├── 0112200301
    │   ├── ODF
    │   └── work
    ├── 0123700101
    │   ├── ODF
    │   └── work
    ├── 0400550201
    │   ├── PPS
    │   └── work
    ├── 0790830101
    │   ├── ODF
    │   ├── PPS
    │   └── work
    └── ...
```

You **should** run SAS tasks inside the `work` directory for the Obs ID you are working with. It is possible to run SAS tasks from **any** directory, but whichever directory you are in when you run a SAS task, that is where SAS will output any new files.

In some cases it is convenient to create a subdirectory within the `work` directory if the SAS tasks you are running will generate a very large number of output files. For example, if you are working with Optical Monitor data the file structure for the Obs ID you are working with may look like this:
```
├── 0400550201
│   ├── ODF
│   ├── PPS
│   ├── work
│   │   └── OM_files
```


## Run a Task Using the `my_obs` Object


When you create an `ObsID` object (i.e. `my_obs`) one of the functions it has available is called `run_MyTask`, and it behaves just like using `MyTask` (except you don't have to have `.run()` on the end). It also makes it so that the task uses the same output logging options set for `my_obs` (see next section).

```python
my_obs.run_MyTask('cifbuild',{})
```

<!-- #region -->
## Output Logging

The `ObsID` class accepts inputs to control output logging. The inputs (with defaults) to `ObsID` are:
```python
obsid (required)
data_dir    = None
logfilename = None
tasklogdir  = None
output_to_terminal = True
output_to_file     = False
```
  - `obsid` is required and has to be the 10-digit observation ID number for the observation you are working with.
  - `data_dir` is the directory where you want the XMM data downloaded.
  - `logfilename`, if this is defined, then all output will be written to this file (but only if `output_to_file=True`). If no file name is given then the name of the log file will be 'ObsID_'+the Obs ID you are working with. Any SAS tasks run using `basic_setup` (i.e. `cifbuild`, `odfingest`, `emproc`, `epproc`, and `rgsproc`) will have their output written to their own file in the `work_dir`.
  - `tasklogdir` is the directory where output log files will be written. If not defined then it will use the `data_dir` for all top level Python related output, and `work_dir` for all other SAS tasks.
  - `output_to_terminal`, if `True` then output will be written to the terminal, if `False` then not.
  - `output_to_file`, if `True` then output will be written to a log file, if `False` then not.

If you are running an individual task, for example `evselect`, the `MyTask` object also accepts the same logging inputs as the `ObsID` class.
```python
taskname (required)
inargs   (required)
logfilename = None, 
tasklogdir  = None,
output_to_terminal = True, 
output_to_file     = False
```
The difference is that `logfilename` will default to the task name, and `tasklogdir` will default to the current working directory (which should be the `work_dir` since that is where you will be running SAS tasks).
<!-- #endregion -->

```python
my_obs.run_MyTask('cifbuild', {}, output_to_terminal = True, output_to_file = True)
```

<!-- #region -->
### Extra Log Output from pySAS

SAS has its own verbosity that controls how much output is generated. If you change the verbosity for all of SAS it will also change the verbosity for pySAS accordingly. BUT, it is possible to set the verbosity for pySAS separately. pySAS has default configurations, and one of the options is `pysas_verbosity`. This is information specifically for pySAS, and not the individual SAS tasks you may be running. The levels of verbosity for pySAS are:

    - CRITICAL : Similar to SAS verbosity of '1'
    - ERROR    : Similar to SAS verbosity of '2' or '3'
    - WARNING  : Similar to SAS verbosity of '4' or '5'
    - INFO     : Similar to SAS verbosity of '6' or '7'
    - DEBUG    : Similar to SAS verbosity of '8', '9', or '10'

The default verbosity is set to `WARNING`. You can set the verbosity for pySAS by using the command,
```python
pysas.sas_cfg.set("sas", 'pysas_verbosity', value='INFO')
```
with the 'value' set to whatever level you need. Below we can see the difference it makes.
<!-- #endregion -->

```python
# Current pySAS verbosity  of 'WARNING'
pysas.sas_cfg.get("sas", "pysas_verbosity")
```

```python
# Output with the current pySAS verbosity
my_obs = pysas.obsid.ObsID(obsid,data_dir=data_dir)
my_obs.basic_setup(repo        = 'sciserver',
                   overwrite   = True,
                   run_epproc  = False,
                   run_emproc  = False,
                   run_rgsproc = False,
                   cifbuild_opts  = {'options':'-V 1'},
                   odfingest_opts = {'options':'-V 1'})
```

```python
# Change the pySAS verbosity to 'INFO'
pysas.sas_cfg.set("sas", 'pysas_verbosity', value='INFO')
pysas.sas_cfg.get("sas", "pysas_verbosity")
```

```python
# Output with higher level of pySAS verbosity
my_obs = pysas.obsid.ObsID(obsid,data_dir=data_dir)
my_obs.basic_setup(repo        = 'sciserver',
                   overwrite   = True,
                   run_epproc  = False,
                   run_emproc  = False,
                   run_rgsproc = False,
                   cifbuild_opts  = {'options':'-V 1'},
                   odfingest_opts = {'options':'-V 1'})
```
