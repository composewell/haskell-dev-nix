The nixpkgs Library
===================

``nixpkgs`` is a nix module that defines a function taking a
configuration set as argument and generating a set of derivations,
utility functions to generate derivations. The source of this nix module
is hosted at https://github.com/NixOS/nixpkgs.

This module is installed in the ``NIX_PATH`` by the ``nix-channel``
command.  When importing it we can specify a config::

  let pkgs =
        import <nixpkgs>
            { config.allowBroken = true;
              config.allowUnfree = true;
            }

It returns a set (pkgs) which consists of a library of functions and
package derivations.

Finding a function's definition
-------------------------------

To find more details about any function in nixpkgs. Load ``nixpkgs`` in
the nix repl and type the function name::

    nix-repl> :l <nixpkgs>
    nix-repl> stdenv.mkDerivation
    «lambda @ /nix/store/gnkd9i59pswalkflb647fnjjnxgyl1n9-nixpkgs-20.09pre228453.dcb64ea42e6/nixpkgs/pkgs/stdenv/generic/make-derivation.nix:22:5»

Then you can open the file and check the definition.

Fetchers
--------

nixpkgs.fetch* ::

  nix-repl> :l <nixpkgs>
  nix-repl> fetch<tab>

Trivial Builders
----------------

The following functions wrap stdenv.mkDerivation, making it easier to
use in certain cases.

Running commands:

* runCommand*

Writing files into nix store:

* write*
* symlinkJoin

* buildFHSUserEnv
* pkgs.mkShell

nixpkgs.lib.*
-------------

Attribute path ``nixpkgs.lib`` or
``nixpkgs.pkgs.lib`` contains `a library of functions
<https://nixos.org/nixpkgs/manual/#chap-functions>`_ to help in writing
package definitions. To see a list of all::

    nix-repl> :l <nixpkgs>
    nix-repl> lib.<tab>

Some common function sets:

* nixpkgs.lib.asserts.*
* nixpkgs.lib.attrsets.*
* nixpkgs.lib.strings.*
* nixpkgs.lib.trivial.*
* nixpkgs.lib.lists.*
* nixpkgs.lib.debug.*
* nixpkgs.lib.options.*

nixpkgs.stdenv
--------------

Attribute path ``nixpkgs.stdenv`` or `nixpkgs.pkgs.stdenv
<https://nixos.org/nixpkgs/manual/#chap-stdenv>`_ contains a nix package that
provides a standard build environment including gcc, GNU coreutils, GNU
findutils and other basic tools::

    $ nix-env -qaP -A nixpkgs.stdenv
    nixpkgs.stdenv  stdenv-darwin

nixpkgs.stdenv.mkDerivation
---------------------------

``stdenv`` provides a wrapper around `builtins.derivation
<https://nixos.org/nix/manual/#ssec-derivation>`_
called `stdenv.mkDerivation
<https://nixos.org/nixpkgs/manual/#sec-using-stdenv>`_.  It adds
a default value for ``system`` and always uses ``bash`` as the
``builder``, to which the supplied builder is passed as a command-line
argument. mkDerivation extends ``derivation``, it adds some additional
attributes and passthru attributes::

      lib.extendDerivation validity.handled
        ({
           overrideAttrs = f: mkDerivation (attrs // (f attrs));
           inherit meta passthru;
           ...
         } // passthru)
        (derivation derivationArg);

See ``nixpkgs/pkgs/stdenv/generic/make-derivation.nix``.

mkDerivation arguments
~~~~~~~~~~~~~~~~~~~~~~

See ``nixpkgs/pkgs/stdenv/generic/make-derivation.nix``.

In addition to ``derivation`` arguments::

  stdenv.mkDerivation {
    # if pname and version are specified the derivation "name" is
    # automatically set to "${pname}-${version}"
    pname   # optional package name
    version # optional package version
    builder ? # use your own builder script instead of genericBuild

    , depsBuildBuild              ? [] # -1 -> -1
    , depsBuildBuildPropagated    ? [] # -1 -> -1
    , nativeBuildInputs           ? [] # -1 ->  0  N.B. Legacy name
    , propagatedNativeBuildInputs ? [] # -1 ->  0  N.B. Legacy name
    , depsBuildTarget             ? [] # -1 ->  1
    , depsBuildTargetPropagated   ? [] # -1 ->  1

    , depsHostHost                ? [] #  0 ->  0
    , depsHostHostPropagated      ? [] #  0 ->  0
    , buildInputs                 ? [] #  0 ->  1  N.B. Legacy name
    , propagatedBuildInputs       ? [] #  0 ->  1  N.B. Legacy name

    , depsTargetTarget            ? [] #  1 ->  1
    , depsTargetTargetPropagated  ? [] #  1 ->  1

    , checkInputs                 ? []
    , installCheckInputs          ? []

    # Configure Phase
    , configureFlags ? []
    , cmakeFlags ? []
    , mesonFlags ? []
    , configurePlatforms ? lib.optionals
    , doCheck ? config.doCheckByDefault or false
    , doInstallCheck ? config.doCheckByDefault or false

    , strictDeps ? stdenv.hostPlatform != stdenv.buildPlatform
    , meta ? {}
    , passthru ? {}
    , pos ? # position used in error messages and for meta.position
    , separateDebugInfo ? false
    , outputs ? [ "out" ]
    , __darwinAllowLocalNetworking ? false
    , __impureHostDeps ? []
    , __propagatedImpureHostDeps ? []
    , sandboxProfile ? ""
    , propagatedSandboxProfile ? ""

    , hardeningEnable ? []
    , hardeningDisable ? []

    , patches ? []

    , __contentAddressed ?

Generic Builder Attributes
~~~~~~~~~~~~~~~~~~~~~~~~~~

The generic builder provided by stdenv uses environment variables to control
its behavior. These environment variables are passed as attributes of the
derivation. Note that we can pass any arbitrary attributes to the derivation
arg set.

Refer to the nix manual for details about the builder. Some of the attributes
that we can use::

    src     # source directory containing the package source

Builder environment and execution
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Environment of the builder: In addition to the environment provided by
``derivation``:

* ``stdenv`` contains the path to ``stdenv`` package. The shell script ``$stdenv/setup`` is
  typically sourced by the builder script to setup the ``stdenv`` environment.
* ``buildInputs`` attribute ensures that the bin subdirectories of these
  packages appear in the ``PATH`` environment variable during the build,
  that their include subdirectories are searched by the C compiler, and so
  on.

Builder script execution:

* If ``builder`` is not set, then the ``genericBuild`` function from
  ``$stdenv/setup`` is called as build script. ``buildPhase``, ``installPhase``
  customizations in ``mkDerivation`` are used by ``genericBuild`` allowing
  customization of its behavior. `See the manual
  <https://nixos.org/nixpkgs/manual/#sec-stdenv-phases>`_ to check out
  more details about the build phases.
* If ``builder`` is set then the specified builder script is invoked with
  ``bash``. You can source ``$stdenv/setup`` in the script. You can still
  define ``buildPhase``, ``installPhase`` etc as shell functions and then
  invoke ``genericBuild`` in your script.

To checkout the shell functions and environments available in ``$stdenv/setup``
install ``stdenv`` and visit its store path.

An Example Package
------------------

Let's now try to build a small real Haskell source package. `packcheck
<http://hackage.haskell.org/package/packcheck>`_ is a minimal Haskell
package that contains a shell script ``packcheck.sh`` which can build
any Haskell package. We will use that script to build ``packcheck`` itself::

  $ mkdir nix-play
  $ cd nix-play
  $ cat > default.nix
  {}:
      with import <nixpkgs> {};
      let src = fetchurl {
            url = http://hackage.haskell.org/package/packcheck-0.5.1/packcheck-0.5.1.tar.gz;
            sha256 = "79e7cfc63e70b627be8c084b3223fdd261a5d79ddd797d5ecc2cee635e651c16";
          };

          path =
                "${bash}/bin"
              + ":${which}/bin"
              + ":${coreutils}/bin"
              + ":${gnused}/bin"
              + ":${gawk}/bin"
              + ":${gnutar}/bin"
              + ":${gzip}/bin"
              + ":${curl}/bin"
              + ":${llvmPackages.bintools}/bin"
              + ":${ghc}/bin"
              + ":${cabal-install}/bin";

      in derivation {
          name = "packcheck-0.5.1";
          system = "x86_64-darwin";
          builder = "${bash}/bin/bash";
          args =
              [ "-c"
                ''set -e
                  export HOME=$TMP
                  export PATH=${path}
                  tar -zxvf ${src}
                  cd packcheck-0.5.1
                  bash packcheck.sh cabal-v2
                  mkdir -p $out/bin
                  touch $out/bin/hello
                ''
              ];
      }

``with`` is a nix language keyword. ``import``, ``fetchurl`` and
``derivation`` are nix builtin functions. We can use them with or without
``builtins.`` prefix e.g. we can use ``builtins.import`` or just ``import``.

``<nixpkgs>`` is a syntax that is used to refer to the first nix module
(better known as nix expression) named ``nixpkgs`` found in
``NIX_PATH``.  By default it would be the nix expression in
``$HOME/.nix-defexpr/channels/nixpkgs``. The evaluation of this
expressions returns a set named ``nixpkgs``. ``nixpkgs.*`` in the code
is just accessing members of this set.

The builtin function ``import`` brings in the result of a nix expression
in the current scope. For example, to bring in the ``nixpkgs`` set and
refer to it by the name ``nixpkgs`` we can use::

  let nixpkgs = import <nixpkgs> {};
  in nixpkgs.dockerTools.buildImage { ... }

``with import <nixpkgs> {};`` brings all the members of the set imported
by ``import <nixpkgs> {}`` into the current scope. For example, the package
``nixpkgs.ghc`` comes into the current scope as the name ``ghc`` and we
can refer to it using ``${ghc}``.

``builtins.fetchurl`` downloads the file referred to by the URL and assigns
the path location of the downloaded file to the ``src`` variable.

We setup the ``path`` variable to a ``PATH`` string containing the paths of all
the required utilities needed by the build script.

``derivation`` uses ``bash`` as the builder which is invoked with the
``-c`` option passing an inline bash script as argument. The script
untars the source tarball, changes directory to the source and then
invokes its build script ``packcheck.sh`` to build the package. Finally,
it creates a dummy ``hello`` artifact inside the output directory passed
by nix.

callPackage
~~~~~~~~~~~

In the above example, for simplicity we used ``with import <nixpkgs> {}``
which brought all the package names under ``nixpkgs`` as variables
in our scope.  Instead of clobbering the namespace with all those
variables we should pass them as arguments, as follows::

  $ cat packcheck.nix
  { fetchurl, bash, which, coreutils, gnused, gawk, gnutar, gzip, curl
  , llvmPackages, ghc, cabal-install }:
  ...

Then we can call the function defined in ``packcheck.nix`` supplying the
arguments using ``nix-build`` as follows::

    $ nix-build -E 'with import <nixpkgs> {}; nixpkgs.pkgs.callPackage ./packcheck.nix {}'

``callPackage`` calls ``./packcheck.nix``, automatically filling the
arguments that are not explicitly supplied in the arguments to
``callPackage`` (i.e. ``{}`` in the above example). The argument
variables are filled from the variables of the same names available in
the current scope i.e. the ones brought in scope by the ``with`` clause
in the command above.

We can write this expression in ``default.nix`` so that we can use
``nix-build`` without any arguments::

  $ cat default.nix
  { nixpkgs ? import <nixpkgs> {} }:
      nixpkgs.pkgs.callPackage ./packcheck.nix {}
  $ nix-build

Installing the package
~~~~~~~~~~~~~~~~~~~~~~

::

    $ nix-env -i ./result

Building user environments
--------------------------

We now know how to build a derived object from a recipe using
``nix-build``.  The derived object output from ``nix-build`` is stored
in the nix store and a ``result`` link to the object is made available
in the current directory or as specified on the command line.

We can go further and also create a user environment for the object and
link its artifacts from a user profile, making the artifacts available
for general use.

A user environment is a collection of derived objects linked into a standard
file system hierarchy under one root. ``.nix-profile`` is a user environment.

::

  $ cat myprofile.nix
  let nixpkgs = import <nixpkgs> {};
  in nixpkgs.buildEnv {
        name = "my-packages";
        paths = [ nixpkgs.pkgs.bc nixpkgs.pkgs.coreutils ];
        pathsToLink = [ "/share" "/bin" ];
        extraOutputsToInstall = [ "man" "doc" ];
     }

It would create a derived object ``my-packages`` containing ``/share``,
``/bin`` directories of the ``bc`` and ``coreutils`` packages.

The ``nix-env`` command creates new user environments whenever we install or
uninstall packages.

Build functions and derivations
-------------------------------

See `Nix Language Tutorial <nix-language.rst>`_ for nix expression
language and builtin functions.

The set ``nixpkgs`` consists of a lot of nix functions/builders in
addition to package derivations. These functions can be used in various
custom derivations.  See the reference guide mentioned above for
some common ones. For an authoritative source of all functions see
``$HOME/.nix-defexpr/channels/nixpkgs``.

Building Nix shell
------------------

``nix-shell file.nix`` starts a shell from the nix expression in
``file.nix`` ::

  with (import <nixpkgs> {});
  mkShell {
    buildInputs = [
      coreutils
      gmp
    ];

    shellHook = ''
      alias ll = "ls -l"
      export C_INCLUDE_PATH = "${gmp}/include"
    '';
  }

By default nix-shell spawns a shell from ``shell.nix`` if the filename argument
is not specified.

The file must specify a derivation. ``mkShell`` above generates a derivation.

Customizing Nix distribution
----------------------------

`Nix getting started guide <user-guide.rst>`_ describes how the
nix distribution works. The whole distribution or collection of packages
visible to nix commands are defined by the nix expression obtained by
evaluating ``$HOME/.nix-defexpr``. Packages derived from this source are
fetched, built and stored in the nix store. When packages are available in the
binary cache they are downloaded from the cache.

Picking a Nix distribution
~~~~~~~~~~~~~~~~~~~~~~~~~~

Within a nix expression, instead of picking nixpkgs from NIX_PATH or
configured nix channels, we can pick a specific version of nixpkgs::

  nixpkgs = import (fetchTarball "https://github.com/NixOS/nixpkgs/archive/4da09d369baa2200edb9df27fe9c88453b0ea6cf.tar.gz") {}

This can be used to pin the code to a specific version. For stability use a
stable nixos release version or for most current release use nixos-unstable.

Customizing the Nix distribution
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

What nix packages are available to you is determined by the
``NIX_PATH``. The directories in the ``NIX_PATH`` are combined together
in a single nix expression, this nix expression is used by the nix
utilities to show available packages or to install packages.

We can customize the distribution we are using by:

* Specifying a config when importing nixpkgs ``import <nixpkgs> config``
* Using a global configuration file in ``~/.config/nixpkgs/config.nix``
* Specifying overlays using the ``~/.config/nixpkgs/overlays.nix`` file
* Specifying overlays using individual overlay files in the
   ``~/.config/nixpkgs/overlays directory.``
* Using environment variables

Config specification
~~~~~~~~~~~~~~~~~~~~

Configuration to customize nixpkgs is specified as a set with attributes ::

  {
    allowUnfree =
    allowUnfreePredicate =
    allowBroken =
    allowUnsupportedSystem =
    whitelistedLicenses =
    blacklistedLicenses = 
    allowInsecurePredicate = 
    permittedInsecurePackages =
    packageOverrides =
    overlays =
  }

Usually we skip the config when importing nixpkgs and default values of these
attributes are used::

  import <nixpkgs> {};

However we can use a config::

  import <nixpkgs> { allowUnfree = true; };

Configuration file
~~~~~~~~~~~~~~~~~~

XXX todo: move the distracting parts out in a let caluse. Explain those in
separate sections before the config example.

We can modify the source nix expression defining the nix distribution by using
the nix configuration file ``~/.config/nixpkgs/config.nix``. That way we
can change or override the packages visible to the system, and add our
own packages to it::

  {
    allowUnfree = true;
    allowUnfreePredicate =
        pkg: builtins.elem (lib.getName pkg) [ "flashplayer" "vscode" ];
    allowBroken = true;
    allowUnsupportedSystem = true;
    whitelistedLicenses = with stdenv.lib.licenses; [ amd wtfpl ];
    blacklistedLicenses = with stdenv.lib.licenses; [ agpl3 gpl3 ];
    allowInsecurePredicate = pkg: builtins.stringLength (lib.getName pkg) <= 5;
    # Checked only if allowInsecurePredicate is not defined
    permittedInsecurePackages =
        [
            "hello-1.2.3"
        ];
    # takes all available pkgs as an argument and returns a modified set
    # of packages.
    packageOverrides = pkgs:
        with pkgs;
        {
            # Write a shell script in nix store to setup paths
            # This is an example, you may not need this as this may already be
            # setup by nix.sh.
            myProfile =
                writeText "my-profile"
                    ''
                    export PATH=$HOME/.nix-profile/bin:/nix/var/nix/profiles/default/bin:$PATH
                    export MANPATH=$HOME/.nix-profile/share/man:/nix/var/nix/profiles/default/share/man:$MANPATH
                    export INFOPATH=$HOME/.nix-profile/share/info:/nix/var/nix/profiles/default/share/info:$INFOPATH
                    '';
            # define a custom package bundle
            myBundle = pkgs.buildEnv {
                name = "my-packages";
                paths = [
                  bc
                  coreutils
                  gdb
                  texinfoInteractive # for install-info command

                  # copy our shell script to user profile i.e. $out
                  (runCommand "profile" {}
                      ''
                      mkdir -p $out/etc/profile.d
                      cp ${myProfile} $out/etc/profile.d/my-profile.sh
                      ''
                  )
                ];
            pathsToLink = [ "/share" "/bin" ];
            extraOutputsToInstall = [ "man" "doc" ];

            # Copy info files to the info root node i.e. $out/share/info/dir
            postBuild =
                ''
                if [ -x $out/bin/install-info -a -w $out/share/info ]
                then
                  shopt -s nullglob
                  for i in $out/share/info/*.info $out/share/info/*.info.gz
                  do
                      $out/bin/install-info $i $out/share/info/dir
                  done
                fi
                '';
            };
        };
  }

See ``~/.nix-defexpr/channels/nixpkgslib/licenses.nix`` for a complete
list of licenses.

Environment variables
~~~~~~~~~~~~~~~~~~~~~

::

  $ export NIXPKGS_ALLOW_BROKEN=1
  $ export NIXPKGS_ALLOW_UNSUPPORTED_SYSTEM=1
  $ export NIXPKGS_ALLOW_UNFREE=1
  $ export NIXPKGS_ALLOW_INSECURE=1

Overrides
~~~~~~~~~

A package set is a dependency tree. Packages at the top of the tree
depend on packages below. If we override a package in this tree the
whole tree should be rebuilt to use the changed definition wherever the
package is used.

Note that overriding a package lower below may cause rebuilding of all
the packages that depend on it. To avoid rebuilding the whole world we
can push the override as far above in the tree as possible. For example,
if one of the packages that depends on "git" requires a changed definition
of git then we can override that package to use a new "git" instead of
overriding the original "git".

The functions below are basic low level constructs to override
individual packages in the package set.

Override is used on a function to override its arguments.  Wherever a
function is called to build the whole package set, it is effectively
replaced by its overridden definition. ``makeOverridable`` can be used to make
a function overridable, providing a ``override`` attribute that can be called
to override its arguments.

::

  <pkg>.override          # override the arguments passed to an overridable function "pkg".
  <pkg>.overrideAttrs     # override the attribute set passed to a stdenv.mkDerivation call
  <pkg>.overrideDerivation # override a derivation using an old derivation
  lib.makeOverridable


* https://nixos.org/manual/nixpkgs/stable/#chap-overrides
* https://nixos.org/guides/nix-pills/override-design-pattern.html
* https://nixos.org/guides/nix-pills/nixpkgs-overriding-packages.html

Overlays
~~~~~~~~

Override is used to override function definitions whereas overlays
override sets. We can combine a set definition with a new overridden
definition to create a new resulting set. This can be used to override
the entire set of packages (``nixpkgs``).

Overlays are Nix functions which accept two arguments, conventionally
called ``self`` and ``super``, and return a set of packages. The first
argument (self) corresponds to the final package set. The second
argument (super) corresponds to the result of the evaluation of the
previous stages of Nixpkgs. It does not contain any of the packages
added by the current or following overlays::

  self: super:
      {
        boost = super.boost.override {
          python = self.python3;
        };
        rr = super.callPackage ./pkgs/rr {
          stdenv = self.stdenv_32bit;
        };
      }

The value returned by this function should be a set similar to
``pkgs/top-level/all-packages.nix``, containing overridden and/or new
packages.

* See https://nixos.wiki/wiki/Overlays for a good explanation

Applying Overlays
.................

1) When importing nixpkgs::

    import <nixpkgs> { overlays = [ overlay1 overlay2 ]; }.

2) Using ``~/.config/nixpkgs/overlays.nix`` file
3) By creating individual overlay files in the
   ``~/.config/nixpkgs/overlays`` directory.
4) By calling the following::

    pkgs.extend
    pkgs.appendOverlays

This is more expensive as it recomputes the nixpkgs fixed point.

packageOverrides
~~~~~~~~~~~~~~~~

``packageOverrides`` acts as an overlay with only the ``super``
argument. It is therefore appropriate for basic use, but overlays are
more powerful and easier to distribute.

We can modify the attibutes of a package derivation or add new package
derivations to the set of packages in ``nixpkgs`` ::

  {
    packageOverrides = pkgs: rec {
      coreutils = pkgs.coreutils.override { ... };
    };
  }

Installing Additional Package Components
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

For example, if we want to install the dev version of the gmp package to get
the gmp.h header file installed in ~/.nix-profile/include ::

  {
    packageOverrides = super:
    {
        gmp =
            super.gmp.overrideAttrs (oldAttrs:
                {
                  meta = oldAttrs.meta // { outputsToInstall = oldAttrs.meta.outputsToInstall or [ "out" ] ++ [ "dev" ]; };
                }
            );
    };
  }

Finding sha256
--------------

::

  $ nix-prefetch-url http://hackage.haskell.org/package/beam-core-0.9.0.0/beam-core-0.9.0.0.tar.gz
  path is '/nix/store/w5ipisq7bq6zmjjfmjzvws62wkwnp7hs-beam-core-0.9.0.0.tar.gz'
  0ixaxjmgg162ff7srvwmkv5lp1kfb0b6wmrpaz97rsmlpa5vf6ji

Specify the sha256 given above. Then evaluate the expression.::

  $ nix-shell
  building '/nix/store/gcp4vxffxfadb5sx1j5cfcws52m1nc1z-source.drv'...

  hash mismatch in fixed-output derivation '/nix/store/a6i38xchxpdp1y1mg700j9ijg3cb5101-source':
    wanted: sha256:0ixaxjmgg162ff7srvwmkv5lp1kfb0b6wmrpaz97rsmlpa5vf6ji
    got:    sha256:0d79ca1rxnq2hg1ap7mx3l3qg3hwfaby4g3cckk4y3ml86asw6jh

If the sha256 mismatches use the hash in the "got" field above.

Further Reading
---------------

You are now equipped with all the basic knowledge of Nix and
Nix packaging, you can now move on to the `Nix Haskell Development Guide
<nix-haskell-packages.rst>`_.

References
----------

* https://nix.dev/tutorials/towards-reproducibility-pinning-nixpkgs.html#pinning-nixpkgs
* https://ghedam.at/15978/an-introduction-to-nix-shell
* https://medium.com/@MrJamesFisher/nix-by-example-a0063a1a4c55 Nix by example
