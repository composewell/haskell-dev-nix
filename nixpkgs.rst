The nixpkgs Library
===================

The ``nixpkgs`` channel's top level expression when evaluated returns a
function which takes a config set as an argument.  When importing it we
can specify the config::

  let pkgs =
        import <nixpkgs>
            { config.allowBroken = true;
              config.allowUnfree = true;
            }

It returns a set (pkgs) which consists of a library functions and
package derivations.

Finding a function's definition
-------------------------------

To find more details about any function. Load ``nixpkgs`` in the nix repl and
type the function name::

    nix-repl> :l <nixpkgs>
    nix-repl> stdenv.mkDerivation
    «lambda @ /nix/store/gnkd9i59pswalkflb647fnjjnxgyl1n9-nixpkgs-20.09pre228453.dcb64ea42e6/nixpkgs/pkgs/stdenv/generic/make-derivation.nix:22:5»

Then you can open the file and check the definition.

TBD: scripts to print all functions or all sets in a nixpkgs path, and their
documentations.

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
<https://nixos.org/nixpkgs/manual/#sec-using-stdenv>`_.
It adds a default value for ``system`` and always uses ``bash`` as the
``builder``, to which the supplied builder is passed as a command-line
argument::

  stdenv.mkDerivation {
    name    # name of the package, if pname and version are specified this is
            # automatically set to "${pname}-${version}"
    pname   # package name
    version # package version
    src     # source directory containing the package source
    builder ? # use your own builder script instead of genericBuild
    buildInputs ? # dependencies e.g. [libbar perl ncurses]
    buildPhase ? # build phase script
    installPhase ? # install phase script
    ...
  }

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
The source of ``mkDerivation`` can be found in
``$HOME/.nix-defexpr/channels/nixpkgs/pkgs/stdenv/generic/make-derivation.nix``.

Quick References
----------------

* https://nixcloud.io/tour/ A tour of Nix (language)
* https://medium.com/@MrJamesFisher/nix-by-example-a0063a1a4c55 Nix by example
* https://nix.dev/anti-patterns/language.html
