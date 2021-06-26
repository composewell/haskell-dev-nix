Nix Package Derivation
======================

If you are new to nix read the `Nix getting started guide
<user-guide.rst>`_ first.  See `Nix Language <nix-language.rst>`_ for
nix expression language and builtin functions.

Nix is an "immutable" package manager, it can be used to build,
install, use and remove packages from different sources into the system
hosting nix. Before building a package, it builds and installs all the
dependencies of the package, if necessary. Moreover, the dependencies of
one package cannot affect other packages, each package can have its own
, potentially different, version of the same dependency.

.. contents:: Table of Contents
   :depth: 1

The purpose of the nix language is to generate derivations.

Building Packages
-----------------

Packages in nix are self contained directory trees in nix store. These
directory trees are derived from source recipes that define the sources,
dependencies, environment, tools and the build process. The resulting
output artifacts are stored in a directory tree in the nix store. We
will call this directory tree a "derived object" and this process of
building it "derivation".

The directory name of a derived object contains a hash of all the
inputs, environment, tools and anything that is used in the build
process. If we are trying to build an object we first generate the key
of the object from its inputs and see if it is already present in the
store, and use that if it is present, otherwise build it from source.

Let's now go through the basics of defining and building a derived
object using a nix expression. We generalize the term "package" to
"derived object" or "derivation". The term "derivation" can also used to
refer to the process or the recipe used to build the object.

Derivation Recipe
~~~~~~~~~~~~~~~~~

The primitive nix operation to create a derivation recipe is
``derivationStrict``. It takes a set describing the inputs to build a
derived object from a source::

  nix-repl> derivationStrict { system = "x86_64-darwin"; name = "dummy"; builder = "/usr/bin/env"; }
  { drvPath = "/nix/store/xs4l5mv0rfzidxh4d5pigka2nsjpdy1r-dummy.drv"; out = "/nix/store/2869jzplqdaipayhij966s3c5lxv83l3-dummy"; }

The key information that it requires is the name of the derived object
and how to build it. The output describes the paths in the nix store
where the generated artifacts corresponding to this derivation are
generated.

``derivation`` and ``nixpkgs.stdenv.mkDerivation`` are higher level
wrappers around ``derivationStrict``.  Let's write a simple example
using ``derivation`` and see how it works::

  $ cat > package.nix
  derivation {
      system = "x86_64-darwin";
      name = "dummy";
      builder = "/usr/bin/env";
      src = ./.;
  }

The expression in ``package.nix`` calls ``derivation`` with a ``set``
type argument (a set of key value pairs) describing the derivation. It
says our package is called ``dummy`` and its source is in ``./.``
i.e. the current directory and it is built using the ``/usr/bin/env``
program.

``/usr/bin/env`` does nothing, it just prints its environment
variables. We have chosen this just to illustrate the environment
that nix uses to call the builder, which is important to write a real
builder.

Evaluating the Recipe
~~~~~~~~~~~~~~~~~~~~~

A nix expression that evaluates to a derivation, list of derivations or
a set of derivations can be used by ``nix-instantiate`` to "realize"
those derivations.

Let's evaluate our trivial example::

    $ nix-instantiate --eval package.nix
    { system = "x86_64-darwin";
      name = "dummy";
      src = /Users/harendra/tmp;
      builder = "/usr/bin/env";

      type = "derivation";
      drvAttrs = { builder = "/usr/bin/env"; name = "dummy"; src = /Users/harendra/tmp; system = "x86_64-darwin"; };
      outputName = "out";
      drvPath = <CODE>;
      outPath = <CODE>;
      all = <CODE>;
      out = ... ;
    }

``--eval`` partially evaluated the derivation to a set. The set contains the
attributes that we specified and some more. Note that the path ``./.`` got
translated to an absolute path. Now let's evaluate it more using the
``--strict`` option::

    $ nix-instantiate --eval --strict package.nix
    { ...
      drvPath = "/nix/store/qg1q1hk3rb7ci8fq3ldkhgqvqfnmnal8-dummy.drv";
      outPath = "/nix/store/n8k5fdcgv52qqk64sz2nv8azqrfili8z-dummy";
      all = [ out ];
      out = <SELF>;
      ...
    }

It got fully evaluated now, the store paths of the derivation
``drvPath`` and ``outPath`` are generated using a unique hash based on the
build environment. ``out`` refers to the result of this derivation and
``all`` is a list of all outputs of the derivation which is just ``out``
in this case.

Building the Derivation Spec
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Let's actually generate the derivation spec now::

  $ nix-instantiate package.nix
  /nix/store/qg1q1hk3rb7ci8fq3ldkhgqvqfnmnal8-dummy.drv

Let's open ``/nix/store/qg1q1hk3rb7ci8fq3ldkhgqvqfnmnal8-dummy.drv`` and see::

  Derive
    ( [("out","/nix/store/n8k5fdcgv52qqk64sz2nv8azqrfili8z-dummy","","")]
    , []
    , ["/nix/store/9q6a8fnsqpvgp4czvby4q9pncmc88v67-tmp"]
    , "x86_64-darwin"
    , "/usr/bin/env"
    , []
    , [ ("builder","/usr/bin/env")
      , ("name","dummy")
      , ("out","/nix/store/n8k5fdcgv52qqk64sz2nv8azqrfili8z-dummy")
      , ("src","/nix/store/9q6a8fnsqpvgp4czvby4q9pncmc88v67-tmp")
      , ("system","x86_64-darwin")
      ]
    )

Everything that the final derived object depends on has to be in the nix store,
therefore, our source directory ``./.`` has been copied to
``/nix/store/9q6a8fnsqpvgp4czvby4q9pncmc88v67-tmp`` in the store, this
path is also passed to the builder as ``src`` environment variable.

The list at the end contains the environment variables that are passed as
environment of the builder when it is invoked. We can use the following command
to print the environment::

    $ nix-store --print-env /nix/store/qg1q1hk3rb7ci8fq3ldkhgqvqfnmnal8-dummy.drv

Local Path Translation
~~~~~~~~~~~~~~~~~~~~~~

An important thing to note is that we have an attribute ``src = ./.``
referring to the current directory path. Any path type attribute
referring to a local path (not in the nix store) causes the file or the
directory tree to be copied to the store and its location in the store
is put in the ``src`` environment variable::

  src=/nix/store/9q6a8fnsqpvgp4czvby4q9pncmc88v67-tmp

Also, note that the permissions of the tree are made read-only and the
timestamps are set to 01-Jan-1970.

We can access any artifacts in our current directory by using the above
translated path.

Building the Derivation
~~~~~~~~~~~~~~~~~~~~~~~

``nix-instantiate`` only created the derivation spec object and copied
the source to the nix store. The output object does not exist yet. Let's
try creating it from the derivation spec.  The ``builder`` program is
run when we realize the dervation. Our builder does nothing but prints
its environment::

  $ nix-store --realise /nix/store/qg1q1hk3rb7ci8fq3ldkhgqvqfnmnal8-dummy.drv

  ...
  NIX_BUILD_CORES=8
  NIX_LOG_FD=2
  NIX_STORE=/nix/store
  TERM=xterm-256color

  HOME=/homeless-shelter
  PATH=/path-not-set

  NIX_BUILD_TOP=/private/var/folders/p4/fdt36vy95f52t_3dnpcx8_340000gn/T/nix-build-dummy.drv-0
  PWD=/private/var/folders/p4/fdt36vy95f52t_3dnpcx8_340000gn/T/nix-build-dummy.drv-0
  TEMP=/private/var/folders/p4/fdt36vy95f52t_3dnpcx8_340000gn/T/nix-build-dummy.drv-0
  TEMPDIR=/private/var/folders/p4/fdt36vy95f52t_3dnpcx8_340000gn/T/nix-build-dummy.drv-0
  TMP=/private/var/folders/p4/fdt36vy95f52t_3dnpcx8_340000gn/T/nix-build-dummy.drv-0
  TMPDIR=/private/var/folders/p4/fdt36vy95f52t_3dnpcx8_340000gn/T/nix-build-dummy.drv-0
  ...

Our builder printed its environment variables that are passed to it by
nix-store. In addition to the above environment variables, nix also passes
the attributes used in ``derivation``'s argument set - as environment
variables with the same names::

  ...
  name=dummy
  system=x86_64-darwin
  builder=/usr/bin/env
  ...

Lastly, it passes a default ``out`` environment variable pointing to a
directory where the builder is supposed to store its output artifacts::

  ...
  out=/nix/store/n8k5fdcgv52qqk64sz2nv8azqrfili8z-dummy
  ...

Notice that nix cleans the environment before invoking the builder
process and sets only those variables that are strictly required and
even sets some of the variables (``HOME`` and ``PATH``) to "junk" values
so that defaults are not filled by the shell. This is to ensure an
isolated build environment. We used ``/usr/bin/env`` in this example for
illustration, but we are not supposed to use any path outside the nix
sandbox for building, we must supply explicit dependencies on other nix
packages and use the paths of those.

Building with ``nix-build``
~~~~~~~~~~~~~~~~~~~~~~~~~~~

``nix-build`` is a high level command built on top of the low level
``nix-instantiate`` and ``nix-store`` commands.  We can simply use
``nix-build`` to perform the above steps in one go::

    $ nix-build package.nix

The output directory ``$out`` is symlinked as ``result`` in the current
directory.

Note: ``nix-build`` without any arguments builds the derivations from
``default.nix`` in the current directory.

builtins.derivation
-------------------

`builtins.derivation <https://nixos.org/nix/manual/#ssec-derivation>`_
is a wrapper over ``derivationStrict``.

Some of the arguments are described below::

        name    # required, package name
        system  # required, e.g. "i686-linux" or "x86_64-darwin"
        builder # required, build script, a derivation or a path e.g. ./builder.sh
        args ? []    # optional, command line args to be passed to the builder
        outputs ? [] # optional, a list of symbolic outputs of the derivation
                     # e.g.  [ "lib" "headers" "doc" ]

Builder Environment and Execution
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Debugging Note: We can use ``/usr/bin/env`` as the builder script to print the
environment that is being passed to the builder.

Every attribute of ``derivation`` is passed as an environment variable
to the builder process with the following translations:

* A path (e.g., ../foo/sources.tar) type attribute causes the referenced
  file to be copied to the store; its location in the store is put in the
  environment variable.

  The tree copied in the nix store is made read-only. If the builder depends on
  the ability to write to this tree in-place then it has to make it writable
  explicitly. Or it has to copy the tree to the temporary directory.

  The copied tree in the nix store has timestamps as 01-Jan-1970, the
  beginning of the Unix epoch. So you cannot depend on the timestamps.
* A derivation type attribute causes that derivation to be built prior
  to the present derivation; its default output path is put in the
  environment variable.
* ``true`` is passed as the string ``1``, ``false`` and ``null`` are
  passed as an empty string.
* By default, a derivation produces a single output path, denoted
  as ``out``. ``outputs = [ "lib" "headers" "doc" ]`` causes ``lib``,
  ``headers`` and ``doc`` to be passed to the builder containing
  the intended nix store paths of each output.  Each output path
  is a directory in nix store whose name is a concatenation of the
  cryptographic hash of all build inputs, the name attribute and the
  output name. The output directories are created before the build
  starts, environment variables for each output name are passed to the
  build script.  The build script stores its output artifacts at those
  paths.

Other environment variables:

* ``NIX_BUILD_TOP``: path of the temporary directory for this build.
* ``NIX_STORE``: the top-level Nix store directory (typically, /nix/store).

These are set to prevent issues when they are not set:

* ``TMPDIR``, ``TEMPDIR``, ``TMP``, ``TEMP``=``$NIX_BUILD_TOP``
* ``PATH=/path-not-set``
* ``HOME=/homeless-shelter``

The builder is executed as follows:

* cd $TMPDIR/<tmp dir>/
* Clear the environment and set to the attributes as above
* If an output path already exists, it is removed
* The builder is executed with the arguments specified by the attribute args.
* If the builder exits with exit code 0, it is considered to have succeeded.
* A log of standard output and error is written to ``/nix/var/log/nix``

Post build:

* The temporary directory is removed (unless the -K option was specified).
* If the build was successful, Nix scans each output path for references
  to input paths by looking for the hash parts of the input paths. Since
  these are potential runtime dependencies, Nix registers them as
  dependencies of the output paths.

Working with Nix Store
----------------------

Nix Global Data
~~~~~~~~~~~~~~~

The whole nix distribution consists of ``/nix/var`` and ``/nix/store``.

The ``/nix/var`` directory contains top level control information about the
whole nix installation. ``/nix/var/nix`` contains:

* ``profiles`` - default user profiles, the top level point from where a user
  accesses the distribution.
* ``gcroots`` - derivations reachable from this are not removed
* ``userpool``
* a sqlite database (what does it have?)

Nix Store
~~~~~~~~~

Nix store consists of directories that may contain a self-contained
package or a derivation (.drv suffix). Each such package may depend on
other packages installed in the store. The whole tree is rooted at user
profiles. Each path in the store is a tree consisting of a package and
its dependencies.

The ``nix-store`` command can be used to manipulate the contents of the
nix store. See ``nix-store --help``.

Subtree/path level:

Create:

* ``nix-store --add`` - add a path to nix-store
* ``nix-store --realise`` - make sure the given store path tree is complete and
  valid, if not fetch it or build it.
* ``nix-store --restore`` - restore a path tree from a nix archive (tar)
* ``nix-store --import`` - import an exported archive (see --export)
* ``nix-store --load-db`` - load a nix db for the path tree (see --dump-db)

Read:

* ``nix-store --query`` - query info about a path
* ``nix-store --print-env`` - environment of a .drv path
* ``nix-store --read-log`` - print build log of a path
* ``nix-store --verify-path`` - verify a path
* ``nix-store --dump`` - dump a path tree as nix archive (tar)
* ``nix-store --export`` - export an archive for non nix-store purposes
* ``nix-store --dumpdb`` - dump nix db for the path tree

Update:

* ``nix-store --repair-path`` - repair a path

Delete:

* ``nix-store --delete`` - delete if nobody is using it

Store level:

* ``nix-store --serve`` -  provide access to the whole store over stdin/stdout
* ``nix-store --gc`` - garbage collect
* ``nix-store --verify`` - verify the consistency of the nix database
