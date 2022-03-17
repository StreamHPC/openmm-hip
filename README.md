# OpenMM HIP Plugin

This plugin adds HIP platform that allows to run [OpenMM](https://openmm.org) on CDNA and RDNA
AMD GPUs on [AMD ROCm™ open software platform](https://rocmdocs.amd.com).

## Installing with Conda

This plugin requires hipFFT, install it from ROCm repositories:

```sh
apt install hipfft
```

```sh
conda create -n openmm-env -c streamhpc -c conda-forge openmm-hip
conda activate openmm-env
```

This command creates a new environment, installs OpenMM and the plugin and activates the new
environment.

Verify your installation (HIP must be one of available platforms):

```sh
python -m openmm.testInstallation
```

If there is no HIP among available platforms check why the HIP platform fails to load:

```sh
python -c "import openmm as mm; print('---Loaded---', *mm.pluginLoadedLibNames, '---Failed---', *mm.Platform.getPluginLoadFailures(), sep='\n')"
```

Run tests:

```sh
cd $CONDA_PREFIX/share/openmm/tests/
./test_openmm_hip.sh
```

Run benchmarks (see more options `python benchmark.py --help`):

```sh
cd $CONDA_PREFIX/share/openmm/examples/
python benchmark.py --platform=HIP
```

To remove OpenMM and the HIP plugin, run:

```sh
conda uninstall openmm-hip openmm
```

## Building

This project uses [CMake](http://www.cmake.org) for its build system.

The plugin requires source code of OpenMM, it can be downloaded as an archive
[here](https://github.com/openmm/openmm/releases) or as a Git repository:

```sh
git clone https://github.com/openmm/openmm.git
```

<!-- TODO Update when HIP-related changes are merged into the main repository -->
Currently the main repository of OpenMM does not include all changes required for the HIP platform
so [this branch](https://github.com/StreamHPC/openmm/tree/develop_stream) must be used:

```sh
git clone https://github.com/StreamHPC/openmm.git -b develop_stream
```

To build the plugin, follow these steps:

1. Create a directory in which to build the plugin.

2. Run the CMake GUI or ccmake, specifying your new directory as the build directory and the top
level directory of this project as the source directory.

3. Press "Configure".

4. Set OPENMM_DIR to point to the directory where OpenMM is installed.  This is needed to locate
the OpenMM header files and libraries.

5. Set OPENMM_SOURCE_DIR to point to the directory where OpenMM source code is located.

6. Set CMAKE_INSTALL_PREFIX to the directory where the plugin should be installed.  Usually,
this will be the same as OPENMM_DIR, so the plugin will be added to your OpenMM installation.

7. Press "Configure" again if necessary, then press "Generate".

8. Use the build system you selected to build and install the plugin.  For example, if you
selected Unix Makefiles, type `make install`.

Here are all commands required for building and installing OpenMM with HIP support from the latest
source code:

```sh
mkdir build build-hip install

git clone https://github.com/StreamHPC/openmm.git -b develop_stream
cd build
cmake ../openmm/ -D CMAKE_INSTALL_PREFIX=../install -D OPENMM_BUILD_COMMON=ON -D OPENMM_PYTHON_USER_INSTALL=ON
make
make install
make PythonInstall
cd ..

git clone https://github.com/StreamHPC/openmm-hip.git
cd build-hip
cmake ../openmm-hip/ -D OPENMM_DIR=../install -D OPENMM_SOURCE_DIR=../openmm -D CMAKE_INSTALL_PREFIX=../install
make
make install
```

If you do not want to install OpenMM Python libraries into the user site-packages directory
remove `-D OPENMM_PYTHON_USER_INSTALL=ON`.

Use `ROCM_PATH` environment variable if ROCm is not installed in the default directory (/opt/rocm).

## Testing

To run all the test cases build the "test" target, for example by typing `make test`, or call
`ctest --output-on-failure --repeat until-pass:3` (retry three times so stochastic tests have
a chance).

## Troubleshooting and performance tuning

### FFT backends

There are 3 implementations (backends) of FFT, the default is hipFFT/rocFFT.  It is known that
rocFFT from ROCm 4.2 and earlier has correctness issues.  If some tests fail or you suspect that
your simulation with PME produces incorrect results, please try different backends:

* the VkFFT-based implementation (`export OPENMM_FFT_BACKEND=2`), this backend is on par with
  hipFFT/rocFFT, some FFT sizes are faster, some are slower;
* the built-in FFT implementation (`export OPENMM_FFT_BACKEND=0`), it may be faster for small
  simulations with small FFT sizes;
* the hipFFT/rocFFT-based implementation (`export OPENMM_FFT_BACKEND=1`) - default.

### Tile size

By default the HIP Platform uses tile size of 64 on CDNA and tile size of 32 on RDNA for nonbonded
interactions. However, tile size of 32 is supported on CDNA too
(`export OPENMM_FORCE_TILE_SIZE_32=1`), some (usually small) simulations work faster in this mode.

### The kernel compilation: hipcc and hipRTC

By default, the HIP Platform builds kernels with the hipcc compiler. To run the compiler, paths
in the following order are used:

* `properties['HipCompiler']`, if it is passed to Context constructor;
* `OPENMM_HIP_COMPILER` environment variable, if it is set;
* `${ROCM_PATH}/bin/hipcc`, if `ROCM_PATH` environment variable is set;
* `/opt/rocm/bin/hipcc` otherwise.

There is an alternative way to compile kernels: hipRTC, it is implemented by
`plugins/hipcompiler`.  To enable this way:

* set `properties['HipAllowRuntimeCompiler'] = 'true'`;
* set `OPENMM_USE_HIPRTC` environment variable to 1 (`export OPENMM_USE_HIPRTC=1`).

## License

The HIP Platform uses OpenMM API under the terms of the MIT License.  A copy of this license may
be found in the accompanying file [MIT.txt](licenses/MIT.txt).

The HIP Platform is based on the CUDA Platform of OpenMM under the terms of the GNU Lesser General
Public License.  A copy of this license may be found in the accompanying file
[LGPL.txt](licenses/LGPL.txt).  It in turn incorporates the terms of the GNU General Public
License, which may be found in the accompanying file [GPL.txt](licenses/GPL.txt).

The HIP Platform uses [VkFFT](https://github.com/DTolm/VkFFT) by Dmitrii Tolmachev under the terms
of the MIT License.  A copy of this license may be found in the accompanying file
[MIT-VkFFT.txt](licenses/MIT-VkFFT.txt).
