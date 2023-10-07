---
layout: default
title: "How to build NumPy from source with Intel MKL"
date: 2023-10-07
---

# Instructions for Ubuntu

Since it took quite some time to figure out which tools are necessary to build NumPy from source such that Intel MKL is enabled,
I'm documenting the approach which worked for me here.

## Step 1: Prepare the environment

The NumPy build process requires `python3-dev`.
```bash
sudo apt-get install python3-dev
```
In addition, it makes sense to install NumPy into a new Python virtual environment.
Python virtual environments can for example be created using `venv`.
```bash
sudo apt-get install python3-venv
```
We can now create a virtual environment as follows
```bash
python3 -m venv numpymkl
```
Activate it prior to building NumPy
```bash
source numpymkl/bin/activate
```

## Step 2: Get Intel MKL

The Intel MKL can for example be obtained by installing the Intel oneAPI Base Toolkit.
There are various ways of doing this which Intel documents on their site.
I chose the apt variant.
It is probably best to look up Intel's instructions instead of following the following steps:

First, some basic tools are required
```bash
sudo apt update
sudo apt -y install cmake pkg-config build-essential
```
Second, allow the apt client to use the Intel repository
```bash
wget https://apt.repos.intel.com/intel-gpg-keys/GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB
sudo apt-key add GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB
rm GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB
sudo add-apt-repository "deb https://apt.repos.intel.com/oneapi all main"
```
Third, install the basic compilers
```bash
sudo apt install intel-basekit
```
The recommendation is to put 
```bash
source /opt/intel/oneapi/setvars.sh
```
into the `~/.bashrc`.
Alternatively, just make sure to run this command before you compile or use NumPy.

## Step 3: Install NumPy

Clone the [NumPy git repository](https://github.com/numpy/numpy).
It is tempting to download the zip instead of cloning the repository.
However, you will be missing submodules which might lead to cryptic compilation errors.
So, after cloning the repository, also pull the submodules
```bash
git submodule update --init
```
Now, checkout the NumPy version which you would like to install.
It makes sense to choose a particular tag.
For example, I installed version `1.26.0`.
```bash
git fetch --all --tags
git checkout tags/v1.26.0
```

With the Python virtual environment active as well as with the Intel oneAPI variables set (`source /opt/intel/oneapi/setvars.sh`), navigate into the NumPy repository and install the Python packages that are required for the build process
```bash
python3 -m pip install -r build_requirements.txt
```
Then, install NumPy
```bash
python3 setup.py install
```
If it worked, `python3 -m pip list` will list `numpy` and the correct version number.

Navigate into a different folder and check if Intel MKL is enabled
```bash
python3
>>> import numpy
>>> numpy.show_config()
```
Some information about the BLAS and LAPACK libraries should be printed which tells you that Intel MKL is used.

Note: Do not `import numpy` while you are still in the numpy repository folder.

Note: The Intel oneAPI variables need to be set before you `import numpy`.

[back](./../)
