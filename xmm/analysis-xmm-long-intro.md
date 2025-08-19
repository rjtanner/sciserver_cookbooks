---
jupyter:
  jupytext:
    text_representation:
      extension: .md
      format_name: markdown
      format_version: '1.3'
      jupytext_version: 1.16.0
  kernelspec:
    display_name: (heasoft)
    language: python
    name: conda-env-heasoft-py
---

<!-- #region editable=true slideshow={"slide_type": ""} -->
# pySAS Introduction -- Long Version
<hr style="border: 2px solid #fadbac" />

- **Description:** A longer introduction to pySAS on sciserver.
- **Level:** Beginner
- **Data:** XMM observation of NGC 3079 (obsid=0802710101)
- **Requirements:** Must be run using the `HEASARCv6.35` (or higher) image, along with pySAS version 2.0. Run in the <tt>(xmmsas)</tt> conda environment on Sciserver. You should see <tt>(xmmsas)</tt> at the top right of the notebook. If not, click there and select <tt>(xmmsas)</tt>.
- **Credit:** Ryan Tanner (April 2024)
- **Support:** <a href="https://heasarc.gsfc.nasa.gov/docs/xmm/xmm_helpdesk.html">XMM Newton GOF Helpdesk</a>
- **Last verified to run:** 21 July 2025, for SAS v22.1 and pySAS v2.0

<hr style="border: 2px solid #fadbac" />
<!-- #endregion -->

<!-- #region editable=true slideshow={"slide_type": ""} -->
## 1. Introduction

This tutorial provides a much more detailed explanation on how to use pySAS than the one found in the [Short pySAS Introduction](./analysis-xmm-short-intro.md "Short pySAS Intro"), but like the Short Intro it only covers how to download observation data files, how to calibrate the data, and how to run any SAS task through pySAS. For explanations on how to use different SAS tasks inside of pySAS see the exmple notebooks provided. A tutorial on how to learn to use SAS and pySAS for XMM analysis can be found in [The XMM-Newton ABC Guide](./analysis-xmm-ABC-guide-ch6-p1.md "XMM ABC Guide").

#### SAS Tasks to be Used

- `sasver`[(Documentation for sasver)](https://xmm-tools.cosmos.esa.int/external/sas/current/doc/sasver/index.html)
- `startsas`[(Documentation for startsas)](https://xmm-tools.cosmos.esa.int/external/sas/current/doc/startsas/index.html)
- `cifbuild`[(Documentation for cifbuild)](https://xmm-tools.cosmos.esa.int/external/sas/current/doc/cifbuild/index.html)
- `odfingest`[(Documentation for odfingest)](https://xmm-tools.cosmos.esa.int/external/sas/current/doc/odfingest/index.html)
- `emproc`[(Documentation for emproc)](https://xmm-tools.cosmos.esa.int/external/sas/current/doc/emproc/index.html "emproc Documentation")
- `epproc`[(Documentation for epproc)](https://xmm-tools.cosmos.esa.int/external/sas/current/doc/epproc/index.html "epproc Documentation")
- `rgsproc`[(Documentation for rgsproc)](https://xmm-tools.cosmos.esa.int/external/sas/current/doc/rgsproc/index.html "rgsproc Documentation")
- `omichain`[(Documentation for omichain)](https://xmm-tools.cosmos.esa.int/external/sas/current/doc/omichain/index.html "omichain Documentation")

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
This notebook was designed to run on SciServer, but an equivelent notebook can be found on <a href="https://github.com/XMMGOF/pysas">GitHub</a>. You will need to install the development version of pySAS found on GitHub (<a href="https://github.com/XMMGOF/pysas">pySAS on GitHub</a>). There are installation instructions on GitHub and example notebooks can be found inside the directory named 'documentation'.
<br>
</div>

<div class="alert alert-block alert-warning">
    <b>Warning:</b> By default this notebook will place observation data files in your <tt>scratch</tt> space. The <tt>scratch</tt> space on SciServer will only retain files for 90 days. If you wish to keep the data files for longer move them into your <tt>persistent</tt> directory.
</div>
<!-- #endregion -->

<!-- #region editable=true slideshow={"slide_type": ""} -->
## 2. Procedure
 
Lets begin by asking three questions:

1. What XMM-Newton Observation data do I want to process?
2. Which directory will contain the XMM-Newton Observation data I want to process?
3. Which directory am I going to use to work with (py)SAS?

For the first question, you will need an Observation ID. In this tutorial we use the ObsID `0802710101`. 

For the second question, you will also have to choose a directory for your data (`data_dir`). You can set your data directory to any path you want, but for now we will use your scratch space.

For the third question, a working directory will automatically be created for each ObsID, as explained below. You can change this manually, but using the default is recommended.
___
<!-- #endregion -->

```python
# Make sure pySAS is up to date
%pip install xmmpysas --upgrade
```

```python
import os
import pysas

# To get your user name. Or you can just put your user name in the path for your data.
from SciServer import Authentication as auth
usr = auth.getKeystoneUserWithToken(auth.getToken()).userName

data_dir = os.path.join('/home/idies/workspace/Temporary/',usr,'scratch/xmm_data')
obsid = '0802710101'
```

By running the cell below, an Observation ID (`ObsID`) object is created. By itself it doesn't do anything, but it has several helpful functions to get your data ready to analyse.

```python editable=true slideshow={"slide_type": ""}
my_obs = pysas.obsid.ObsID(obsid,data_dir=data_dir)
```

## 3. Run `ObsID.basic_setup`

When you run the cell below the following things will happen.

1. `basic_setup` will check if `data_dir` exists, and if not it will create it.
2. Inside data_dir `basic_setup` will create a directory with the value for the obs ID (i.e. `$data_dir/0802710101/`).
3. Inside of that, `basic_setup` will create two directories:

    a. `$data_dir/0802710101/ODF` where the observation data files are kept.
    
    b. `$data_dir/0802710101/work` where the `ccf.cif`, `*SUM.SAS`, and output files are kept.
4. `basic_setup` will automatically transfer the data for `obsid` to `$data_dir/0802710101/ODF` from the HEASARC archive.
5. `basic_setup` will run `cfibuild` and `odfingest`.
6. `basic_setup` will then run the basic pipeline tasks `emproc`, `epproc`, and `rgsproc`. The output of these three tasks will be in the `work_dir`.

That is it! Your data is now calibrated, processed, and ready for use with all the standard SAS commands!

```python editable=true slideshow={"slide_type": ""}
my_obs.basic_setup(repo='sciserver',overwrite=False)
```

If you need to include options for either or both `cfibuild` and `odfingest`, these can be passed to `odfcompile` using the inputs `cifbuild_opts='Insert options here'` and `odfingest_opts='Insert options here'`.

Input arguments for `epproc`, `emproc`, and `rgsproc` can also be passed in using `epproc_args`, `emproc_args`, or `rgsproc_args` respectively (or `epchain_args` and `emchain_args` if using the chains). By defaut `epproc`, `emproc`, and `rgsproc` will not rerun if output files are found, but they can be forced to rerun by setting `rerun=True` as an input to `basic_setup`.
 
Another important input is `overwrite=True/False`. If set to true, it will erase **all data**, including any previous analysis output, in the obsid directory (i.e. `$data_dir/0802710101/`) and download the original files again.
 
You can also choose the level of data products you download. `ObsID` has three integrated functions for downloading data:

    1. download_ODF_data: Will download the raw, uncalibrated Observation Data Files (ODF).
    2. download_PPS_data: Will download Pipeline Processed Data Files (PPS).
    3. download_ALL_data: Will download both ODF and PPS data files.


The `my_obs` object will also store some useful information for analysis. For example, it stores `data_dir`, `odf_dir`, and `work_dir`:

```python
print("Data directory: {0}".format(my_obs.data_dir))
print("ODF  directory: {0}".format(my_obs.odf_dir))
print("Work directory: {0}".format(my_obs.work_dir))
```

The location and name of important files are also stored in a Python dictionary in the `my_obs` object.

```python editable=true slideshow={"slide_type": ""}
data_files = list(my_obs.files.keys())
print(data_files,'\n')
for list_name in data_files:
    print(f'File Type: {list_name}')
    print('>>> {0}'.format(my_obs.files[list_name]),'\n')
```

If you want more information on the function `basic_setup` run the cell below to see the function documentation.

```python
my_obs.basic_setup?
```

<!-- #region editable=true slideshow={"slide_type": ""} -->
## 4. Invoking SAS tasks from notebooks

### 4.1 MyTask

Now we are ready to execute any SAS task needed to analize our data. To execute any SAS task within a Notebook, we need to import from `pysas` a component known as `MyTask`. The following cell shows how to do that,
<!-- #endregion -->

```python editable=true slideshow={"slide_type": ""}
from pysas.sastask import MyTask
```

Any SAS task accepts arguments which can be either specific options, e.g. `--version`, which shows the task's version, or parameters with format `param=value`. When the task is invoked from the command line, these arguments follow the name of the task. However, in Notebooks we have to pass them to the task in a different way. This is done using a Python dictionary, whose name you are free to choose. Let the name of such list be `inargs`.

To pass the option `--version` to the task to be executed, we must define `inargs` as,

```python editable=true slideshow={"slide_type": ""}
inargs = {'options' : '--version'}
```

To execute the task, we will use the `MyTask` component imported earlier from <tt>pySAS</tt>, as follows,

```python editable=true slideshow={"slide_type": ""}
t = MyTask('evselect', inargs)
```

In Python terms, `t` is an *instantiation* of the object `MyTask`.

To run `evselect` [(click here for evselect documentation)](https://xmm-tools.cosmos.esa.int/external/sas/current/doc/evselect/index.html "Documentation for sasver") with the input `--version`, we can now do as follows,

```python
t.run()
```

This output is equivalent to having run `evselect` in the command line with argument `--version`.

Each SAS task, regardless of the task being a Python task or not, accepts a predefined set of options. To list which are these options, we can always invoke the task with option `--help` (or `-h` as well).

With some SAS tasks, we could define `inargs` as an empty dictionary, which is equivalent to run the task in the command line without options.


A similar result can be achieved by combining all the previous steps into a single expression, like this,

```python
MyTask('evselect', '-v').run()
```

### 4.2 Listing available options
As noted earlier, we can list all options available to any SAS task with option `--help` (or `-h`),

```python editable=true slideshow={"slide_type": ""}
MyTask('sasversion', '-h').run()
```

As explained in the help text shown here, if the task would have had any available parameters, we would get a listing of them immediately after the help text. Compare the output above with the output for `evselect` below when we pass in the option `-h`.

```python
MyTask('evselect', '-h').run()
```

<!-- #region -->
### 4.3 Log Files and Log Output

Many SAS tasks produce significant amounts of output. There are several ways of controlling the type and amount of output from SAS. The amount of output is controlled by an environment variable called `SAS_VERBOSITY`, which is a number between 0-10. With a value of 0 SAS is entirely silent and does not output anything. A value of 10 is used for debugging purposes and can produce a VERY LARGE AMOUNT OF OUTPUT! Generally a value between 1 and 7 is used. This variable can be set in a few ways.

First it can be set directly using (with whatever value you want):
```python
os.environ['SAS_VERBOSITY'] = '4'
```

Second, it can be set using:
```python
my_obs.sas_talk(verbosity=4)
```

And finally it can be passed in as an option as part of the input arguments when running `MyTask`:
```python
MyTask('epproc', {'options':'--verbosity 4'}).run()
```

The first two methods will change `SAS_VERBOSITY` for **the entire SAS session** until pySAS is restarted. *ALL* tasks run after that will be affected. The last method will *only* change `SAS_VERBOSITY` for that single run of that task.

The `MyTask` object requires two inputs, the task name and the input arguments for the task. But there are several optional arguments that can control what happens to the output. The optional arguments to `MyTask` are:

```python
logfilename = None, 
tasklogdir  = None,
output_to_terminal = True, 
output_to_file     = False
```

- `logfilename`: If this is defined, then all output will be written to this file (but only if `output_to_file=True`). If no file name is given, then the name of the log file will be the task name.
- `tasklogdir`: This is the directory where output log files will be written. If not defined then it will use the `data_dir` for all top level Python related output, and `work_dir` for all other SAS tasks.
- `output_to_terminal`: If `True` then output will be written to the terminal, if `False` then not.
- `output_to_file`: If `True` then output will be written to a log file, if `False` then not.

You can choose to have the output written to both the terminal and a log file (or neither!).

As an additional note, when you instatiate `ObsID` (e.g. `my_obs`) you can also use these same inputs (with default values):

```python
obsid (required)
data_dir    = None
logfilename = None
tasklogdir  = None
output_to_terminal = True
output_to_file     = False
```
<!-- #endregion -->

## 5. How to continue from here?

This depends on your experience level with SAS and what you are using the data for. For a tutorial on preparing and filtering your data for analysis or to make images see [The XMM-Newton ABC Guide](./analysis-xmm-ABC-guide-ch6-p1.md), or check out any of the example notebooks.

```python
os.chdir(my_obs.work_dir)
```

The most common SAS tasks to run are: `epproc`, `emproc`, and `rgsproc`. Each one can be run without inputs (but some inputs are needed for more advanced analysis). These tasks have been folded into the function `basic_setup`, but they can be run individually.


Here is an example of how to apply a "standard" filter. This is equivelant to running the following SAS command:

```
evselect table=unfiltered_event_list.fits withfilteredset=yes \
    expression='(PATTERN $<=$ 12)&&(PI in [200:12000])&&#XMMEA_EM' \
    filteredset=filtered_event_list.fits filtertype=expression keepfilteroutput=yes \
    updateexposure=yes filterexposure=yes
```
The input arguments should be in a list, with each input argument a separate string. Note: Some inputs require single quotes to be preserved in the string. This can be done using double quotes to form the string. i.e. `"expression='(PATTERN <= 12)&&(PI in [200:4000])&&#XMMEA_EM'"`. An explanation of this filter, and other filters, can be found in [The XMM-Newton ABC Guide](./analysis-xmm-ABC-guide-ch6-p1.md).

```python editable=true slideshow={"slide_type": ""}
unfiltered_event_list = "3278_0802710101_EMOS1_S001_ImagingEvts.ds"

inargs = {'table'            : unfiltered_event_list,
          'withfilteredset'  : 'yes',
          'expression'       : "(PATTERN <= 12)&&(PI in [200:4000])&&#XMMEA_EM'",
          'filteredset'      : 'filtered_event_list.fits',
          'filtertype'       : 'expression',
          'keepfilteroutput' : 'yes',
          'updateexposure'   : 'yes',
          'filterexposure'   : 'yes'}

MyTask('evselect', inargs).run()
```

## 6. Alternative to `basic_setup`

<!-- #region -->
The function `basic_setup` is there for convienvience and checks if things have already been run, all with a single command. Running `basic_setup(data_dir=data_dir,overwrite=False,repo='sciserver',rerun=True)` is the same as running the following commands:

```python
my_obs.download_ODF_data(data_dir=data_dir,overwrite=False,repo='sciserver')
my_obs.calibrate_odf()
MyTask('epproc',[]).run()
MyTask('emproc',[]).run()
MyTask('rgsproc',[]).run()
```
For more information on the functions `download_ODF_data` and `calibrate_odf` see the function documentation by running the cells below.
<!-- #endregion -->

```python
my_obs.download_ODF_data?
```

```python
my_obs.calibrate_odf?
```

## 7. A Note on Inputs

<!-- #region editable=true slideshow={"slide_type": ""} -->
Inside the code of pySAS the input arguments are stored in a dictionary, but they can be passed in as a list as well. For example these two are functionally equivalent in pySAS:

Inputs as a dictionary:
```python
inargs = {'table'            : unfiltered_event_list,
          'withfilteredset'  : 'yes',
          'expression'       : "'(PATTERN <= 12)&&(PI in [200:4000])&&#XMMEA_EM'",
          'filteredset'      : 'filtered_event_list.fits',
          'filtertype'       : 'expression',
          'keepfilteroutput' : 'yes',
          'updateexposure'   : 'yes',
          'filterexposure'   : 'yes'}
```

Inputs as a list:
```python
inargs = ['table={}'.format(unfiltered_event_list),
          'withfilteredset=yes',
          "expression='(PATTERN <= 12)&&(PI in [200:4000])&&#XMMEA_EM'",
          'filteredset=filtered_event_list.fits',
          'filtertype=expression',
          'keepfilteroutput=yes',
          'updateexposure=yes',
          'filterexposure=yes']
```

If passing in arguments as a list the parameters and values need to have an equals sign (`=`) between them. The same rules about preserving single quotes should be followed (note the input parameter `expression` above).

If the inputs are passed in as a dictionary pySAS accepts values other than strings. For example, numbers and boolean values are allowed. All other values must be passed in as a single string.
```python
inargs = {'table'          : temporary_event_list, 
          'withrateset'    : True, 
          'rateset'        : light_curve_file, 
          'maketimecolumn' : True, 
          'timecolumn'     : 'TIME', 
          'timebinsize'    : 100, 
          'makeratecolumn' : True}
```
<!-- #endregion -->
