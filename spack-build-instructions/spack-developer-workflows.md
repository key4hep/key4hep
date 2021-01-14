# Spack workflows for developing Key4HEP software

Using spack to develop software is somewhat pushing its intended usage to its limits.
However, it is not impossible and this is an area of spack that is currently under active development.
Unfortunately, this also means that the spack documentation might not be fully up-to-date on these topics.
Hence, this page tries to collect some of the experiences the Key4HEP developers have made.

## Developing a single package

When only developing on a single package it is possible to use the [`dev-build`](https://spack.readthedocs.io/en/latest/command_index.html#spack-dev-build) command of spack.
A brief tutorial can be found in [the spack documentation](https://spack-tutorial.readthedocs.io/en/lanl19/tutorial_developer_workflows.html).
It allows to build a given package directly from local sources in the same way as spack does it, and even makes this package available to other packages in the same way it does packages that have been installed by spack directly.
Here we will use [LCIO](https://github.com/iLCSoft/LCIO) as an example since it can be installed without (or with only one) dependency.

As a first step let's have a look at what installing `lcio` with spack would entail. 
Note that we explicitly disable the ROOT dictionaries in order to limit the number of dependencies
```bash
spack spec -Il lcio ~rootdict
```
```
Input spec
--------------------------------
 -   lcio~rootdict

Concretized
--------------------------------

 -   vdwx2aq  lcio@2.16%gcc@9.3.0~examples~ipo~jar~rootdict build_type=RelWithDebInfo cxxstd=17 arch=linux-ubuntu20.04-skylake
[+]  utzbuq7      ^cmake@3.16.3%gcc@9.3.0~doc+ncurses+openssl+ownlibs~qt patches=1c540040c7e203dd8e27aa20345ecb07fe06570d56410a24a266ae570b1c4c39,bf695e3febb222da2ed94b3beea600650e4318975da90e4a71d6f31a6d5d8c3d arch=linux-ubuntu20.04-skylake
[+]  pljbs5a      ^sio@0.0.4%gcc@9.3.0+builtin_zlib~ipo build_type=RelWithDebInfo cxxstd=17 arch=linux-ubuntu20.04-skylake

```

In this configuration `lcio` has only two dependencies, `sio` and `cmake`, which are both already installed in this case.
If these dependencies are not yet installed, spack will automatically install them for you when using the `dev-build` command.

### Installing a local version with `dev-build`

In order to install a local version of LCIO with spack, first we have to clone it into a local directory
```bash
git clone https://github.com/iLCSoft/LCIO
```

Now we can install this local version via
```bash
cd LCIO
spack dev-build lcio@master ~rootdict
```
This should install `lcio` and all dependencies that are not yet fulfilled, giving you the full output of all the build stages ending on something like the following
```
...
==> lcio: Successfully installed lcio-master-7dovpqn3kscbg672ham5wcqro7lg45gh
  Fetch: 0.00s.  Build: 1.62s.  Total: 1.62s.
[+] /home/tmadlener/work/spack/opt/spack/linux-ubuntu20.04-skylake/gcc-9.3.0/lcio-master-7dovpqn3kscbg672ham5wcqro7lg45gh
```

Note, that it is necessary to specify a single concrete version here for `lcio`. We use `@master`.
This version has to be one that is already available for the package (use `spack info` to find out which ones are available) and cannot be an arbitrary version.
It also does not necessarily have to correspond to the actual version of the source code you are currently installing. 
However, it is of course encouraged to use a meaningful version specifier, since this package should also be useable as desired by dependent packages.

### Using the local version as dependency

Now that the local version has been installed, it would of course be nice to be able to use it in downstream packages as well.
Spack doesn't pick package versions installed from local sources by default, but as with any other package it is possible to require a specific version of a dependency.
This is described in a bit more detail [here](https://key4hep.github.io/key4hep-doc/spack-build-instructions/spack-advanced.html#concretizing-before-installation).

Let's first find out how the version that we have just installed looks like
```bash
spack find -lv lcio
```
which will yield something similar to
```
==> 1 installed package
-- linux-ubuntu20.04-skylake / gcc@9.3.0 ------------------------
7dovpqn lcio@master~examples~ipo~jar~rootdict build_type=RelWithDebInfo cxxstd=17 dev_path=/home/tmadlener/work/ILCSoft/LCIO
```
As you can see the local path from which this version was installed has become part of the spec for the installed package (the `dev_path=...` part of the spec above).
Hence, also the hash is affected by the fact that it has been built from a local source.

### More advanced usage
Note: If you have installed lcio following the description above you might have to uninstall it again first to follow these instructions, because spack will not overwrite an already installed package.

The above instructions only dealt with installing a package from a local source, but not how to easily get a development environment allowing for a quick edit, compile cycle.
This can be achieved by using the `--drop-in` and the `--before`/`--until` arguments of the `dev-build` command:

```bash
spack dev-build --drop-in bash --until cmake lcio@master ~rootdict
```

This command will first install all necessary dependencies, then run the install process for `lcio` until **after** the `cmake` stage and then drop you into a new bash shell with the proper build environment setup.
It will also setup a build folder, which follows the naming scheme `spack-build-${hash}`, in this case: `spack-build-7dovpqn`.
To compile `lcio` simply go into this directory and run `make` in there
```bash
cd spack-build-7dovpqn
make -j4
```
You are now in an environment where you can easily edit the local source code and re-compile afterwards.

Once all the development is done, it is still necessary to install everything to the appropriate location.
This installation has to be registered in the spack database as well. Hence, simply calling `make install` in the build directory will not do the trick.
Another call to `dev-build` is necessary.
```bash
cd .. # go back to the source directory where you started
spack dev-build lcio@master ~rootdict
```
This will run the whole chain again, but it will not overwrite the build directory.
Hence, it will not recompile everything again, but simply install all the build artifacts to the appropriate location.
`spack find -lv lcio` can be used to check if the installation was successful.

**NOTE: You are probably still in the build environment at this stage. To return back to you original shell simply type `exit`.**