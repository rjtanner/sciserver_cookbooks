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

# pySAS Introduction -- Short Version
<hr style="border: 2px solid #fadbac" />

- **Description:** A short introduction to pySAS on sciserver.
- **Level:** Beginner
- **Data:** XMM observation of NGC 3079 (obsid=0802710101)
- **Requirements:** Must be run using the `HEASARCv6.35` (or higher) image, along with pySAS version 2.0. Run in the <tt>(xmmsas)</tt> conda environment on Sciserver. You should see <tt>(xmmsas)</tt> at the top right of the notebook. If not, click there and select <tt>(xmmsas)</tt>.
- **Credit:** Ryan Tanner (April 2024)
- **Support:** <a href="https://heasarc.gsfc.nasa.gov/docs/xmm/xmm_helpdesk.html">XMM Newton GOF Helpdesk</a>
- **Last verified to run:** 21 July 2025, for SAS v22.1 and pySAS v2.0

<hr style="border: 2px solid #fadbac" />


## 1. Introduction
This tutorial provides a short, basic introduction to using pySAS on SciServer. It only covers how to download observation data files and how to calibrate the data.  A much more comprehensive introduction can be found in the [Long pySAS Introduction](./analysis-xmm-long-intro.md "Long pySAS Intro"). This tutorial is intened for those who are already familiar with SAS commands and want to use Python to run SAS commands. A tutorial on how to learn to use SAS and pySAS for XMM analysis can be found in [The XMM-Newton ABC Guide](./analysis-xmm-ABC-guide-ch6-p1.md "XMM ABC Guide"). In this tutorial we will demonstrate, 

1. How to select a directory for data and analysis.
2. How to copy XMM data from the HEASARC archive.
3. How to run the standard XMM SAS commands `cfibuild` and `odfingest`.

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


## 2. Import pySAS and Set `obsid`

```python
# Make sure pySAS is up to date
%pip install xmmpysas --upgrade
```

```python editable=true slideshow={"slide_type": ""}
import os
import pysas

# To get your user name. Or you can just put your user name in the path for your data.
from SciServer import Authentication as auth
usr = auth.getKeystoneUserWithToken(auth.getToken()).userName

data_dir = os.path.join('/home/idies/workspace/Temporary/',usr,'scratch/xmm_data')
obsid = '0802710101'
```

<!-- #region editable=true slideshow={"slide_type": ""} -->
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
<!-- #endregion -->

```python editable=true slideshow={"slide_type": ""}
my_obs = pysas.obsid.ObsID(obsid,data_dir=data_dir)
my_obs.basic_setup(repo='sciserver',overwrite=False)
```

<!-- #region editable=true slideshow={"slide_type": ""} -->
If you want more information on the function `basic_setup` run the cell below or see the long introduction tutorial. All the methods available in the `ObsID` object have documentation that can be accessed by putting a question mark '?' after the name of the method, as shown below.
<!-- #endregion -->

```python
my_obs.basic_setup?
```

<!-- #region editable=true slideshow={"slide_type": ""} -->
## 4. Running SAS Tasks
To run SAS tasks, pySAS has a dedicated class called `MyTask`. SAS tasks should be run from the work directory. The location of the work direcotry is stored as a variable in `my_obs.work_dir`.
<!-- #endregion -->

```python editable=true slideshow={"slide_type": ""}
from pysas.sastask import MyTask
os.chdir(my_obs.work_dir)
```

<!-- #region editable=true slideshow={"slide_type": ""} -->
The class, `MyTask`, takes at least two inputs, the name of the SAS task to run, and a Python dictionary of all the input arguments for that task. For example, to run a task with no input arguments you simply provide an empty dictionary as the second argument.
<!-- #endregion -->

```python
inargs = {}
MyTask('emproc', inargs).run()
```

The most common SAS tasks to run are: `epproc`, `emproc`, and `rgsproc`. Each one can be run without inputs (but some inputs are needed for more advanced analysis). These tasks have been folded into the function `basic_setup`, but they can be run individually.

You can list all input arguments available to any SAS task with the option `'--help'` (or `'-h'`),

```python editable=true slideshow={"slide_type": ""}
MyTask('emproc', '-h').run()
```

<!-- #region editable=true slideshow={"slide_type": ""} -->
If there are multiple input arguments then in the input dictionary they are entered as 'key'-'value' pairs. The keys are the input parameters and the values are the input values. The values should be entered as a string (or as a variable that is a string). For example, here is how to apply a "standard" filter. This is equivelant to running the following SAS command:

```
evselect table=unfiltered_event_list.fits withfilteredset=yes \
    expression='(PATTERN $<=$ 12)&&(PI in [200:12000])&&#XMMEA_EM' \
    filteredset=filtered_event_list.fits filtertype=expression keepfilteroutput=yes \
    updateexposure=yes filterexposure=yes
```

The input arguments should be in a dictionary, with each input vaule a single string. Note: Some inputs require single quotes to be preserved in the string. This can be done using double quotes to form the string. i.e. `"'(PATTERN <= 12)&&(PI in [200:4000])&&#XMMEA_EM'"`
<!-- #endregion -->

```python editable=true slideshow={"slide_type": ""}
unfiltered_event_list = "3278_0802710101_EMOS1_S001_ImagingEvts.ds"

inargs = {'table'            : unfiltered_event_list,
          'withfilteredset'  : 'yes',
          'expression'       : "'(PATTERN <= 12)&&(PI in [200:4000])&&#XMMEA_EM'",
          'filteredset'      : 'filtered_event_list.fits',
          'filtertype'       : 'expression',
          'keepfilteroutput' : 'yes',
          'updateexposure'   : 'yes',
          'filterexposure'   : 'yes'}

MyTask('evselect', inargs).run()
```
