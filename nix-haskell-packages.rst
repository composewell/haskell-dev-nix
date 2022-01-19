Getting Started with Haskell on Nix
===================================

See the `Haskell starting guide <getting-started.rst>`_ if you are new
to Haskell.  See the `Nix getting started guide <user-guide.rst>`_ first
and then `Nix Packages guide <nix-packages.rst>`_ if you are new to nix.

.. contents:: Table of Contents
   :depth: 1

Installing Haskell Packages
---------------------------

The default version of ``ghc`` is available directly under ``nixpkgs``
namespace::

  $ nix-env -iA nixpkgs.ghc

List all available Haskell compilers::

  $ nix-env -qaP -A nixpkgs.haskell.compiler

List all packages for a specific compiler version from the above list::

  $ nix-env -qaP -A nixpkgs.haskell.packages.ghc883

List all packages for the default compiler version::

  $ nix-env -qaP -A nixpkgs.haskellPackages

Note that ``haskellPackages`` is an alias to
``nixpkgs.haskell.packages.<default compiler version>``.

Installing Haskell packages::

  $ nix-env -iA nixpkgs.haskellPackages.streamly
  $ nix-env -iA nixpkgs.haskell.packages.ghc883.streamly

Haskell Development Using Nix Shell
-----------------------------------

We can use ``nix-shell`` to get a user environment with ``ghc`` and
required Haskell packages installed in it.  In the shell we can use
``ghc`` and ``cabal`` as usual to compile and build Haskell programs
that depend on that ``ghc`` and the packages we have installed with it.

Shell with nix packages (no dependencies)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Start a shell with ``ghc-8.6.5``::

  $ nix-shell --packages "haskell.compiler.ghc865"
  $ nix-shell --packages "[haskell.compiler.ghc865 haskell.packages.ghc865.mtl]"

Let's check what we have in the new shell::

  [nix-shell:~]$ ghc --version
  The Glorious Glasgow Haskell Compilation System, version 8.6.5

  [nix-shell:~]$ ghc-pkg list
  /nix/store/yh6ycb18aml15ybhsvnb4qxb6c7l28vp-ghc-8.6.5/lib/ghc-8.6.5/package.conf.d
      Cabal-2.4.0.1
      array-0.5.3.0
      base-4.12.0.0
      ...

This may not be very useful as the dependencies of the packages are not
installed.

Shell with dependencies of a nix package
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Each nix derivation in the nix repository has a shell environment
output artifact available that can be used by nix-shell to recreate the
environment used to build the package::

    $ nix-env -qaPA nixpkgs.haskellPackages.streamly.env
    nixpkgs.haskellPackages.streamly.env  ghc-shell-for-streamly-0.7.2

We can start a nix-shell by using the "<nixpkgs>" (which means find
``nixpkgs`` in ``NIX_PATH``) expression path, and using the attribute
identifying the shell environment under ``nixpkgs``::

  $ nix-shell -A haskellPackages.streamly.env "<nixpkgs>"

  [nix-shell:~]$

It creates a user environment with ``ghc`` and all the dependencies of the
package ``streamly`` installed but not the package itself. We can now use this
environment to develop the package::

  [nix-shell:~]$ which ghc
  /nix/store/y7zc21lk3l5v35w81qgz2mcrxwgf7601-ghc-8.8.3-with-packages/bin/ghc

See what you have got::

  [nix-shell:~]$ ghc-pkg list
  /nix/store/y7zc21lk3l5v35w81qgz2mcrxwgf7601-ghc-8.8.3-with-packages/lib/ghc-8.8.3/package.conf.d
      Cabal-3.0.1.0
      HUnit-1.6.0.0
      QuickCheck-2.13.2
      abstract-deque-0.3
      ...

Shell with nix packages & their dependencies
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

We can use the nix library function ``haskellPackages.ghcWithPackages``
to generate the list of packages along with the full set of their dependencies
from a given set of packages::

  $ nix-shell --packages "haskellPackages.ghcWithPackages (pkgs: with pkgs; [streamly mtl])"

It takes a function argument, the function is passed the set of all
Haskell packages (``pkgs``) and returns a list of packages (``[streamly
mtl]``) to be installed along with ``ghc``. ``ghcWithPackages`` would install
``ghc`` and register the packages passed along with all their dependencies with
ghc.

Shell with only dependencies of a set of packages
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``haskellPackages.shellFor`` starts a shell with ghc and dependencies of
a list of packages. See the ``shellFor`` documentation later in this guide.

To add additional dependencies we can create a derivation with no source but
only dependencies and add it to the list of packages for ``shellFor``::

    additionalDeps = nixpkgs.haskellPackages.mkDerivation rec {
              version = "0.1";
              pname   = "streamly-examples-additional";
              license = "BSD";
              src = builtins.filterSource (_: _: false) ./.;

              setupHaskellDepends = with nixpkgs.haskellPackages; [
                cabal-doctest
              ];
            };

    shell = nixpkgs.haskellPackages.shellFor {
        packages = p:
          [ p.streamly-examples
            additionalDeps
          ];

See https://github.com/alpmestan/ghc.nix/blob/master/default.nix for a more
complicated shell example for ghc build environment.

Shell with dependencies of a source package
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Create a custom shell environment for a local package from the cabal file::

  $ cabal2nix --shell . > shell.nix

  $ nix-shell

  [nix-shell]$ ghc-pkg list
  [nix-shell]$ cabal build

Since nix shell already installed all the dependencies and registers them
with ``ghc``, ``cabal build`` does not build any dependencies, it just
builds the current package using the pre-installed dependencies.

If you want to add additional packages, you need to exit the shell, add
the new package to ``shell.nix``, and restart the shell. For example, to
add dependencies for cabal executables::

        executableHaskellDepends = [
          cabal-doctest
        ];
        executableFrameworkDepends = [
          nixpkgs.darwin.apple_sdk.frameworks.Cocoa
        ];
        executableSystemDepends = [
          nixpkgs.pkgs.zlib
        ];

Environment variables inherited from the current shell can still influence the
build in the nix shell. To make sure that the environment is cleared in the nix
shell::

  $ nix-shell --pure

Note that with this option ``.bashrc`` (or the rc file of your shell) is
still run.

To use a different compiler than the one specified in ``shell.nix``::

  $ nix-shell --argstr compiler ghc865

Shell with dependencies of a source project
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Create a shell from a cabal.project file:

* https://gist.github.com/codebje/000df013a2a4b7c10d6014d8bf7bccf3
* https://input-output-hk.github.io/haskell.nix/reference/library/#callcabalprojecttonix

Cabal
~~~~~

Run cabal commands using the nix shell environment defined in your
``shell.nix`` file ::

  cabal --enable-nix ...

Stack
~~~~~

Build with stack using your nix environment ::

  stack --nix build

Shell Environment Variables
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Nix shell sets these environment variables::

  SHELL
  PATH
  NIX_LDFLAGS
  NIX_LDFLAGS_FOR_TARGET

The nix installed C tool chain is then able to use the linker flag
environment variables to link to the right libraries.

Haskell Language Sets
---------------------

The ``nixpkgs`` set has two sets under it for Haskell::

  nixpkgs.haskell
  nixpkgs.haskellPackages

It is really only ``haskell``, ``haskellPackages`` just refers to the set of
packages in ``nixpkgs.haskell.ghc<prime version>.packages``.

``nixpkgs.haskell``
~~~~~~~~~~~~~~~~~~~

Haskell derivations and functions live under the ``nixkpkgs.haskell``.
The attribute path ``nixpkgs.haskell`` is a set that consists of the
following subsets::

  nixpkgs.haskell.lib # A library of functions
  nixpkgs.haskell.compiler # A set of haskell compiler packages only
  nixpkgs.haskell.packages # Per compiler set of haskell packages

And the following functions::

  nixpkgs.haskell.override # override the haskell package set
  nixpkgs.haskell.overrideDerivation # override mkDerivation
  nixpkgs.haskell.packageOverrides # override a set of packages

``nixpkgs.haskell`` is defined in
``nixpkgs/pkgs/top-level/all-packages.nix`` ::

  nixpkgs = {
    ...
    haskell = callPackage ./haskell-packages.nix { };
    ...
  }

``haskell-packages.nix`` is the nix module that builds the
``nixpkgs.haskell`` set.

``nixpkgs.haskell.lib``
~~~~~~~~~~~~~~~~~~~~~~~

This is a library of functions that help in generating
haskell package derivations. It is defined in
``nixpkgs/pkgs/development/haskell-modules/lib.nix``. To see a list of
all functions use the `nix-ls utility <https://github.com/composewell/nix-utils>`_::

  $ nix-ls nixpkgs.haskell.lib

Some commonly used functions are described below::

  * overrideCabal = drv: f:

  overrideCabal lets you alter the arguments to the mkDerivation
  function::

    x = haskell.lib.overrideCabal haskellPackages.aeson (old: { homepage = old.homepage + "#readme"; })

  * packageSourceOverrides = overrides: self: super:

  ``Map Name (Either Path VersionNumber) -> HaskellPackageOverrideSet``
  Given a set whose values are either paths or version strings, produces
  a package override set (i.e. (self: super: { etc. })) that sets
  the packages named in the input set to the corresponding versions

  * overrideSrc = drv: { src, version ? drv.version }:

  Override the sources for the package and optionaly the version.

  * makePackageSet = import ./make-package-set.nix;

  This function takes a file like `hackage-packages.nix` and constructs
  a full package set out of that.

Convenience functions calling overrideCabal:

* disableCabalFlag/enableCabalFlag
* add/append/removeConfigureFlag
* doBenchmark/dontBenchmark
* doCheck/dontCheck
* markUnbroken

``nixpkgs.haskell.compiler``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Latest three versions of ghc.

``nixpkgs.haskell.packages``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The ``haskell.packages`` attribute contains package sets by ghc versions
under it. In haskell-packages.nix::

  packages = {
    ...
    ghc8104 = ... haskell-modules.nix
                ... make-package-set.nix
                  ... hackage-packages.nix

The attribute ``nixpkgs.haskellPackages`` is aliased to the package list
for the prime version of ghc. It is defined in ``all-packages.nix`` ::

  nixpkgs = {
    ...
    haskellPackages = dontRecurseIntoAttrs haskell.packages.ghc8104;
    ...
  }

Functions in haskellPackages::

  # Low level
  extend
  override
  overrideDerivation
  packageSourceOverrides
  mkDerivation

  # invoke the nix expression in a package
  # drv is the haskell package derivation (mkDerivation)
  callPackage = drv: args:

  # Generate default.nix using cabal2nix and invoke it to build
  # extraCabal2nixOptions is a string representing CLI options to the cabal2nix
  # executable
  callCabal2nixWithOptions = name: src: extraCabal2nixOptions: args:
  callCabal2nix = name: src: args: self.callCabal2nixWithOptions name src "" args;

  # Generate default.nix for sources at 'src'
  haskellSrc2nix = { name, src, sha256 ? null, extraCabal2nixOptions ? "" }:
  hackage2nix = name: version: # Use haskellSrc2nix on source from hackage
  callHackage # callPackage after hackage2nix, only versions in your all-cabal-hashes
  callHackageDirect # any package versions directly from hackage

  ghcWithPackages
  ghcWithHoogle
  hoogleLocal
  developPackage

  # Start a nix shell with dependencies of a list of packages
  # shellFor returns the envFunc of the Haskell derivation which is
  # created by stdenv.mkDerivation, attributes that can be used in
  # stdenv.mkDerivation can also be passed to shellFor.
  shellFor =
    {
      packages
    , withHoogle ? false
    , doBenchmark ? false
    , genericBuilderArgsModifier ? (args: args) # Modify haskell.mkDerivation args
    , ...
    } :

Sets in haskellPackages::

  llvmPackages

``nixpkgs.haskellPackages.mkDerivation``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The function ``haskellPackages.mkDerivation`` is defined in
``make-package-set.nix``. It overrides ``stdenv.mkDerivation``::

  mkDerivationImpl = pkgs.callPackage ./generic-builder.nix {
  ...
  mkDerivation = makeOverridable mkDerivationImpl;

Some of the argument that it takes are::

  pname
  version
  revision ? null
  sha256 ? null
  src ? fetchurl { url = "mirror://hackage/${pname}-${version}.tar.gz"; inherit sha256; }
  description ? null
  homepage ? "https://hackage.haskell.org/package/${pname}"
  license
  maintainers ? null
  changelog ? null

  # build and configure
  editedCabalFile ? null
  buildTarget ? ""
  configureFlags ? []
  buildFlags ? []
  haddockFlags ? []
  dontStrip ? (ghc.isGhcjs or false)
  profilingDetail ? "exported-functions"
  enableDeadCodeElimination ? (!stdenv.isDarwin)  # TODO: use -dead_strip for darwin
  enableHsc2hsViaAsm ? stdenv.hostPlatform.isWindows && lib.versionAtLeast ghc.version "8.4"
  enableParallelBuilding ? true
  useCpphs ? false
  coreSetup ? false # Use only core packages to build Setup.hs.
  enableLibraryForGhci ? false # faster ghci load at the expense of space
  allowInconsistentDependencies ? false # Allow multiple versions of a package

  buildDepends ? []
  setupHaskellDepends ? []
  buildTools ? []
  extraLibraries ? []

  # pkg-config depends
  pkg-configDepends ? []
  libraryPkgconfigDepends ? []
  executablePkgconfigDepends ? []
  testPkgconfigDepends ? []
  benchmarkPkgconfigDepends ? []

  # Library
  isLibrary ? !isExecutable
  enableLibraryProfiling ? !(ghc.isGhcjs or stdenv.targetPlatform.isAarch64 or false)
  enableSharedLibraries ? !stdenv.hostPlatform.isStatic && (ghc.enableShared or false)
  enableStaticLibraries ? !(stdenv.hostPlatform.isWindows or stdenv.hostPlatform.isWasm)

  libraryHaskellDepends ? []
  libraryToolDepends ? []
  librarySystemDepends ? []
  libraryFrameworkDepends ? []

  # Executable
  isExecutable ? false
  enableExecutableProfiling ? false
  enableSharedExecutables ? false

  executableHaskellDepends ? []
  executableToolDepends ? []
  executableSystemDepends ? []
  executableFrameworkDepends ? []

  # Test
  doCheck ? !isCross && lib.versionOlder "7.4" ghc.version
  doCoverage ? false
  testTarget ? ""
  testToolDepends ? []
  testDepends ? []
  testHaskellDepends ? []
  testSystemDepends ? []
  testFrameworkDepends ? []

  # Benchmark
  doBenchmark ? false
  benchmarkToolDepends ? []
  benchmarkDepends ? []
  benchmarkHaskellDepends ? []
  benchmarkSystemDepends ? []
  benchmarkFrameworkDepends ? []

  # Docs
  doHaddock ? !(ghc.isHaLVM or false)
  doHoogle ? true
  doHaddockQuickjump ? doHoogle && lib.versionAtLeast ghc.version "8.6"
  hyperlinkSource ? true

  # stdenv generic builder environment
  preCompileBuildDriver ? null
  postCompileBuildDriver ? null
  preUnpack ? null
  postUnpack ? null
  patches ? null
  patchPhase ? null
  prePatch ? ""
  postPatch ? ""
  preConfigure ? null
  postConfigure ? null
  preBuild ? null
  postBuild ? null
  preHaddock ? null
  postHaddock ? null
  installPhase ? null
  preInstall ? null
  postInstall ? null
  checkPhase ? null
  preCheck ? null
  postCheck ? null
  preFixup ? null
  postFixup ? null

  # Nix
  platforms ? with lib.platforms; all # GHC can cross-compile
  hydraPlatforms ? null
  maxBuildCores ? 16
  jailbreak ? false
  broken ? false
  hardeningDisable ? lib.optional (ghc.isHaLVM or false) "all"
  shellHook ? ""

  enableSeparateBinOutput ? false
  enableSeparateDataOutput ? false
  enableSeparateDocOutput ? doHaddock

  # Pass through attributes
  passthru ? {}

Defining a package::

  "3d-graphics-examples" = callPackage
    ({ mkDerivation, base, GLUT, OpenGL, random }:
     mkDerivation {
       pname = "3d-graphics-examples";
       version = "0.0.0.2";
       sha256 = "02d5q4vb6ilwgvqsgiw8pdc3cflsq495k7q27pyv2gyn0434rcgx";
       isLibrary = false;
       isExecutable = true;
       executableHaskellDepends = [ base GLUT OpenGL random ];
       description = "Examples of 3D graphics programming with OpenGL";
       license = lib.licenses.bsd3;
     }) {};

Custom User Environments
------------------------

A custom user environment is a bundle of packages for a specific task.
To make a custom user environment, say ``nixpkgs.streamly-dev``, that can
be installed using ``nix-env`` like any other nix packages::

    $ cat ~/.config/nixpkgs/config.nix
    {
      packageOverrides =
          # the argument super would be "nixpkgs"
          super:
              let streamlyLibs = hpkgs:
                      with hpkgs;
                      [ # library dependencies
                        atomic-primops base containers deepseq directory
                        exceptions fusion-plugin-types ghc-prim heaps
                        lockfree-queue monad-control mtl network primitive
                        transformers transformers-base zlib
                        # test dependencies
                        ghc hspec QuickCheck random
                      ];
                  streamlyBenchmarks = hpkgs:
                      with hpkgs;
                      [ base deepseq exceptions gauge mtl random
                        transformers typed-process bench-show
                      ];
                  streamlyTools = hpkgs:
                      with hpkgs;
                      [ hlint hasktags ];
                  hDeps =
                      super.pkgs.haskell.packages.ghc883.ghcWithPackages
                          (hpkgs:
                              with hpkgs;
                              (  streamlyLibs hpkgs
                              ++ streamlyTools hpkgs
                              ++ streamlyBenchmarks hpkgs
                              )
                          );
              # return a set with new package definitions
              in { streamly-dev = hDeps; };
    }

Now we can search this package by the attribute ``nixpkgs.streamly-dev``::

    $ nix-env -qaPA nixpkgs.streamly-dev
    nixpkgs.streamly-dev  ghc-8.8.3-with-packages

We can also install this package using ``nix-env -iA nixpkgs.streamly-dev``.

Custom Nix Profile
~~~~~~~~~~~~~~~~~~

We can use a dedicated nix profile for our development environment and
install our custom package distribution in this profile::

    $ nix-env -p ./streamly-dev -iA nixpkgs.nix
    $ nix-env -p ./streamly-dev -iA nixpkgs.streamly-dev

Now we can switch to our new profile to use the custom development
environment::

    $ nix-env -S ./streamly-dev
    $ ghc --version
    $ ghc-pkg list

Custom Nix Environment with Hoogle
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In our custom package distribution example, use ``ghcWithHoogle`` in
place of ``ghcWithPackages``.  When we install it, haddock documentation
and a hoogle database of all our Haskell packages in the distribution
is generated and installed at ``$HOME/.nix-profile/share/doc/hoogle/``.
Note that the ``hoogle`` binary in this profile is setup to pick the
database from this location instead of the standard ``~/.hoogle``.
The artifacts of interest in this directory are:

* The haddock docs: ``$HOME/.nix-profile/share/doc/hoogle/index.html``, use it by opening it in a browser
* The hoogle database:``default.hoo``, use it by running 
  ``hoogle server --local -p 8080``

Build and install from cabal file
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

`cabal2nix`` converts ``<package>.cabal`` to ``<package>.nix`` nix
expression file. Install ``cabal2nix`` ::

  $ nix-env -i cabal2nix
  $ cabal2nix --version
  cabal2nix 2.15.3
  $ cabal2nix --help

Convert the ``.cabal`` file of your package to ``.nix`` file::

  $ cabal2nix . > streamly.nix

Note that we used ``.`` in the argument above. If you specify the
``streamly.cabal`` file instead of ``.`` then it generates the nix file
from Hackage.

Create a ``default.nix`` to run ``nix-build`` conveniently using the nix
file generated above::

  $ cat > default.nix
  { nixpkgs  ? import <nixpkgs> {}
  , compiler ? "ghc865"
  }:
  nixpkgs.pkgs.haskell.packages.${compiler}.callPackage ./<package>.nix { }

  $ nix-build

Customizing the distribution
----------------------------

If you want to use a custom version of a package instead of the one
available from the nix channel.  Generate a nix expression that will be
used to override the package. Use `--no-check` flag if you want to avoid
running tests for the package::

  $ cabal2nix --no-check cabal://streamly-0.6.1 > ~/.nixpkgs/streamly-0.6.1.nix

Then add an override in `default.nix` for your package as follows::

  {
    packageOverrides = super:
      let self = super.pkgs;
      in {
        haskell = super.haskell // {
          packages = super.haskell.packages // {
            ghc865 = super.haskell.packages.ghc865.override {
              overrides = self: super: {
                streamly = self.callPackage ./streamly-0.6.1.nix {};
              };
            };
          };
        };
      };
  }

``.`` refers to ``~/.nixpkgs``?

Building haskell packages without doCheck::

  nixpkgs = import (builtins.fetchTarball https://github.com/NixOS/nixpkgs/archive/dcb64ea42e6.tar.gz)
      { overlays =
          [(self: super: {
                haskell = super.haskell // {
                  packageOverrides = hself: hsuper: {
                    mkDerivation = args: hsuper.mkDerivation (args // {
                      doCheck = false;
                    });
                  };
                };
              }
           )
          ];
      };

Global Override
~~~~~~~~~~~~~~~

Add the above expression in `~/.config/nixpkgs/config.nix` to override
package versions used in a package set.

Using a source repository package
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To use the git source of the streamly package and an external
library dependency on `zlib`, in your `default.nix` file use
`compiler.developPackage`::

  { compilerVersion ? "ghc865" }:
  let
    pkgs = import (fetchGit (import ./version.nix)) { };
    compiler = pkgs.haskell.packages."${compilerVersion}";
    pkg = compiler.developPackage {
      root = ./.;
      source-overrides = {
        streamly = "0.6.1";
      };
    };
    buildInputs = [ zlib ];
  in pkg.overrideAttrs
      (attrs: { buildInputs = attrs.buildInputs ++ buildInputs;})

Compiling and Linking
---------------------

If we have to use the headers and link against a C library, then we can
specify the library in the ``executableSystemDepends`` attribute of the
derivation.

If you want to link the library directly without using a nix derivation
then first install it in your nix profile and then pass the
appropriate linker flags to the tool chain. The library is installed
in ``~/.nix-profile/lib``, and the header files are installed in
``~.nix-profile/include``.

Installing headers
~~~~~~~~~~~~~~~~~~

To install the header files we may have to explicitly install the ``dev``
output of the library package which may not be installed by default.
``nix-env`` cannot select the output paths from a multi-output
derivation. See
https://github.com/NixOS/nixpkgs/pull/76794/commits/611258f063f9c1443a5f95db3fc1b6f36bbf4b52 
for a workaround.

In ``default.nix`` or in a shell expression we can use the ``dev``
component as the dependency e.g. ``gmp.dev`` instead of ``gmp``.

A brute force way to use the header files is to find the path location
of the package in the nix store and use it directly from there.

Using headers and libraries
~~~~~~~~~~~~~~~~~~~~~~~~~~~

See the "Haskell getting started guide" to know how to
use C libraries with cabal.  We can use the environment
variable ``C_INCLUDE_PATH=~/.nix-profile/include`` to find
headers installed in the profile.  Similarly, we can use
``LD_LIBRARY_PATH=~/.nix-profile/lib`` to find the libraries
to link. In a nix shell we can initialize this variable from
``NIX_LDFLAGS_FOR_TARGET`` to find the shell provided libraries.

Deployment
----------

Creating statically linked executables
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

TBD.

Export and import
~~~~~~~~~~~~~~~~~

Build the project::

  $ nix-build

It will generate the output in ``./result`` directory.  To export the
whole closure, first time you may want to do this. This may take a while
and may create a large file because it dumps all the dependencies as
well::

  $ nix-store --export $(nix-store -qR ./result) > out

After the first iteration you may want to this, it dumps only the current
project and not dependencies::

  $ nix-store --export ./result > out

Transfer the generate ``out`` to deployment machine and run this on the
deployment machine::

  $ nix-store --import < out

GHC Development with Nix
------------------------

See https://github.com/alpmestan/ghc.nix/blob/master/default.nix .

Finding sha256 of haskell packages
----------------------------------

You can find the sha256 of a package on the revisions page:

https://hackage.haskell.org/package/streamly-0.7.0/revisions/

Diagnostics
-----------

Important: Multiple packages/libraries with the same name may be
available in different namespaces and under the nix expression
(repository), make sure that you are linking with the correct
library. For example, there is a ``nixpkgs.pkgs.zlib`` and a
``nixpkgs.haskellPackages.zlib``, both are different things and
sometimes using one for the other works but may produce strange results
or errors.

Q: "Missing dependency on a foreign library" when using the nix installed GHC
outside nix.  To resolve this:

A: Do any of the following:
* Use `cabal --enable-nix`, assuming ``shell.nix`` provides the library
* Use a nix shell environment with the given library installed
* Provide the lib/include dir options as shown below

Find the nix-path for the library (e.g. zlib)::

  $ nix-build --no-out-link "<nixpkgs>" -A zlib

Then use this path in `--extra-lib-dirs=` and `--extra-include-dirs=` options
of cabal.

You can also install the library in the nix user's profile using `nix-env` and
use `LIBRARY_PATH` environment variable to tell gcc/clang about it::

  $ export LIBRARY_PATH=$HOME/.nix-profile/lib

Other environment variables that can be used to affect gcc/clang::

  $ export C_INCLUDE_PATH=$HOME/.nix-profile/include
  $ export CPLUS_INCLUDE_PATH=$HOME/.nix-profile/include

Q: When compiling with ghc/gcc/clang I see an error like this::

    Linking .../streamly-benchmarks-0.0.0/x/chart/build/chart/chart ...
    ld: library not found for -lz
    clang-7: error: linker command failed with exit code 1 (use -v to see invocation)
    `cc' failed in phase `Linker'. (Exit code: 1)
    Error: build failed

A: ``libz`` (``-lz`` in the error message) is provided by ``nixpkgs.pkgs.zlib``.
   Add ``nixpkgs.pkgs.zlib`` to ``executableSystemDepends`` in ``mkDerivation``.

Q. How to install the dev version of a library to get the include headers? For
example install gmp headers to compile ghc source?

A. ``nix-env`` cannot select the output paths from a multi-output derivation. See
https://github.com/NixOS/nixpkgs/pull/76794/commits/611258f063f9c1443a5f95db3fc1b6f36bbf4b52 
for a workaround.

Mac OS Specific
~~~~~~~~~~~~~~~

Q: On macOS, getting this error::

    cbits/c_fsevents.m:1:10: error:
         fatal error: 'CoreServices/CoreServices.h' file not found
      |
    1 | #include <CoreServices/CoreServices.h>
      |          ^
    #include <CoreServices/CoreServices.h>
             ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    1 error generated.
    `cc' failed in phase `C Compiler'. (Exit code: 1)

A: The ``CoreServices`` framework is missing::

  $ nix-env -iA nixpkgs.pkgs.darwin.apple_sdk.frameworks.CoreServices
  installing 'apple-framework-CoreServices'
  building '/nix/store/04yl425g4lp3ld5xlzv04b7n8zbmzi3y-user-environment.drv'...
  created 71 symlinks in user environment

Q: Got a "framework not found" error when linking an executable::

  Linking .../streamly-benchmarks-0.0.0/x/chart/build/chart/chart ...
  ld: framework not found Cocoa
  clang-7: error: linker command failed with exit code 1 (use -v to see invocation)
  `cc' failed in phase `Linker'. (Exit code: 1)
  Error: build failed

A: Add ``nixpkgs.pkgs.darwin.apple_sdk.frameworks.Cocoa`` to
  ``executableFrameworkDepends`` in mkDerivation.

Quick References
----------------

* `Nix getting started guide <user-guide.rst>`_
* `Nix manual Haskell section <https://nixos.org/nixpkgs/manual/#haskell>`_
* `cabal2nix: convert cabal file to nix expression <http://hackage.haskell.org/package/cabal2nix>`_
* `hackage2nix: update Haskell packages in nixpkgs <https://github.com/NixOS/cabal2nix/tree/master/hackage2nix>`_
* https://haskell4nix.readthedocs.io/index.html

Utilities
~~~~~~~~~

* `The nix-ls utility <https://github.com/composewell/nix-utils>`_

Resources
---------

* https://github.com/input-output-hk/haskell.nix
* https://github.com/cachix/cachix-action
* https://stackoverflow.com/questions/57725045/disable-building-and-running-tests-for-all-haskell-dependencies-in-a-nix-build
* https://github.com/direnv/direnv/wiki/Nix direnv
* https://github.com/target/lorri/ lorri
* `Nix based CI <https://github.com/mightybyte/zeus>`_
* https://scrive.github.io/nix-workshop/index.html
