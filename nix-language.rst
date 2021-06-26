Nix Language
------------

You can use ``nix repl`` to try out the language interactively. Any expression
that is typed in the repl is evaluated and its result is printed.

Operators: See https://nixos.org/manual/nix/stable/#table-operators

Comments::

    # this is a comment
    /* this is a comment */

Null : ``null``

Number Literals::

    1
    1.1
    0.1e1
    (- 1)

Arithmetic operations::

    1 + 2
    1 - 2
    1 * 2
    1 / 2 (note 1/2 would be a path type string)
    1 < 2
    1 <= 2
    1 > 2
    1 >= 2

Strings
~~~~~~~

Strings::

    "hello world"
    'hello world'

Indented Strings::

  ''
    This is the first line.
    This is the second line.
      This is the third line.
  ''

Special strings: URI Strings can be written without quotes::

    http://hello.world

Special strings: Paths (at least one "/" is needed to deem it as a path type)::

    ./.       # current directory
    /.        # root directory
    a/b
    ~/a       # expand ~ to home directory
    <nixpkgs> # search nixpkgs in NIX_PATH

String concatenation::

    "hello" + "world"

String interpolation, expand an expression in a string::

    let expr = "world" in "hello ${expr}"

Since ${ and '' have special meaning in indented strings, you need a way
to quote them. $ can be escaped by prefixing it with '' (that is, two
single quotes), i.e., ''$. '' can be escaped by prefixing it with ',
i.e., '''. $ removes any special meaning from the following $. Linefeed,
carriage-return and tab characters can be written as ''\n, ''\r, ''\t,
and ''\ escapes any other character.

Local variables
~~~~~~~~~~~~~~~

  let
    x = "foo";
    y = "bar";
  in x + y

Lists
~~~~~

whitespace separated items in square brackets::

    [ 123 ./foo.nix "abc" (f { x = y; }) ]

Concatenating lists::

    [ 1 2 ] ++ [ 3 4 ]

Booleans
~~~~~~~~

Booleans: ``true``, ``false``

Conditionals::

    if e1 then e2 else e3

Logical Operations::

    ! x
    x == y
    x != y
    x && y
    x || y
    x -> y (!x || y)

Sets
~~~~

Sets: name value pairs (attributes), terminated by ``;`` and enclosed in
curly brackets::

  { x = 123;
    text = "Hello";
    y = f { bla = 456; };
  }

Recursive sets (defined with a ``rec`` keyword), attributes can refer to
each other::

  rec {
    x = y;
    y = 123;
  }.x

SELECT operator: Attributes can be selected from a set using the ``.``
operator.  Default value in an attribute selection can be provided using
the ``or`` keyword. For example::

  { a = "Foo"; b = "Bar"; }.c or "Xyzzy"

Inherit: In a set or in a let-expression definitions can be inherited::

  let x = 123; in
  { x = x;
    y = 456;
  }

  is equivalent to

  let x = 123; in
  { inherit x;
    y = 456;
  }

``inherit x`` implies ``x = x``
``inherit x y z`` can be used to inherit multiple attrs at the same time.
``inherit (pkgs) zlib`` implies ``zlib = pkgs.zlib``
``inherit (pkgs) zlib coreutils`` can be used to inherit multiple attrs from
pkgs.

with:

``with set; expr``: introduces the set ``set`` into the lexical scope of
``the expression expr``::

  let as = { x = "foo"; y = "bar"; };
  in with as; x + y

Note that ``with a b c`` is equivalent to ``with (a b c)``. Therefore, we can
write::

  with import <nixpkgs> {};

Set overlay::

  set1 // set2

The resulting set consists of attributes from both set1 and set2. If an
attribute is present in both then set2 overrides set1.

Set operations::

    set.attrpath
    set.attrpath or defaultValue
    set ? "attrpath" (does set contain attrpath or not: true/false)
                     (same as: builtins.hasAttr "attrpath" set)
                     (also see "?" in optional args for functions)
    set1 // set2

Set as Function
~~~~~~~~~~~~~~~

A set that has a ``__functor`` attribute whose value is callable (i.e. is
itself a function or a set with a __functor attribute whose value is
callable) can be applied as if it were a function, with the set itself
passed in first::

  let add = { __functor = self: x: x + self.x; };
      inc = add // { x = 1; };
  in inc 1

Functions
~~~~~~~~~

Anonymous functions are defined as ``pattern: expr``::

    # single argument function
    x: !x # negation function

    # set argument
    { x, y, z }: x + y + z

    # optional arguments with default values
    { x, y ? "foo", z ? "bar" }: x + y + z

    # @pattern, in the following examples "args" variable holds the
    # whole argument set
    args@{ x, y, z, ... }: x + y + z + args.a
    { x, y, z, ... } @ args: x + y + z + args.a

If you want to define a function without an argument then pass an empty set as
argument::

  {}: "hello"

Named functions are just let bindings for anonymous functions::

    let f = x: !x
        g = {x , y, z}: x + y + z

Calling a function. Whitespace is function application operator::

    # single argument
    f "foo"

    # set argument
    f {x = "foo"; y = "bar"; z = "baz";}

Functions are first class, i.e. they can return functions::

    let concat = x: y: x + y; # function returning a function
    in builtins.map (concat "foo") [ "bar" "bla" "abc" ] # Currying

Debugging
~~~~~~~~~

Assertions::

    assert e1; e2

Dynamic
~~~~~~~

Make the set attributes to be accessed, dynamically ::

    let attr = "lib"
    builtins.getAttr attr nixpkgs

Evaluating expressions
----------------------

Using ``nix eval``
~~~~~~~~~~~~~~~~~~

You can either use ``nix repl`` to interactively evaluate the expressions or
use ``nix eval`` to evaluate an expression::

  $ nix eval '("hello")'
  "hello"
  $ nix eval '(1 + 2)'
  3

Note the parenthesis to evaluate an expression otherwise it evaluates
an attribute from ``NIX_PATH``.

Do not end the expression with a semicolon.

Using ``nix-instantiate``
~~~~~~~~~~~~~~~~~~~~~~~~~

We can evaluate a file using ``nix-instantiate``::

  $ cat expr.nix
  "hello"
  $ nix-instantiate --eval expr.nix
  "hello"

Built-in functions
------------------

The easiest way to find top level functions is to use tab in ``nix repl``::

    nix-repl> <tab>

builtins.*
~~~~~~~~~~

Nix provides `a library of built-in functions
<https://nixos.org/nix/manual/#ssec-builtins>`_. All built-in functions
are available through the ``builtins.`` namespace prefix. To see a list of all
builtins::

    nix-repl> builtins.<tab>

Printing to stdout
------------------

You can use ``builtins.trace``. For example::

  $ nix eval '(builtins.trace "hello world" "expr")'
  trace: hello world
  "expr"

or ``nixpkgs.lib.debug.trace*``::

  let nixpkgs = import <nixpkgs> {};
  in lib.debug.traceSeq (builtins.attrNames nixpkgs.lib) ""

Nix Modules: importing a file
-----------------------------

How do we use nix code from a file or organize code over multiple files?
We can store a nix expression (any nix expression) in a file.  For
example, ``expr.nix``::

  $ cat expr.nix
  [1 2 3]

The builtin function ``import`` can be used to load the expression from
the file to use it in another expression::

  $ nix eval '(import ./expr.nix)'
  [ 1 2 3 ]

The effect of ``import <path>`` is to replace the import statement with the
contents of the file.

::

  $ cat expr.nix
  {}: [1 2 3]
  $ nix eval '(import ./expr.nix {})'
  [ 1 2 3 ]

Its common to define a set in a file and use it like this::

  $ cat expr.nix
  {}: {x = "hello";}
  $ nix eval '(with import ./expr.nix {}; x)'
  "hello"

If you want to store multiple definitions in a file you can store them as an
expression evaluating to a set.

Common mistakes: note that the argument to import must be path type
string. ``expr.nix`` is not a path type, ``./expr.nix`` is.  A path must
have a "/" in it::

  $ nix eval '(import expr.nix)'
  error: undefined variable 'expr' at (string):1:9

Nix Modules: importing a directory
----------------------------------

If the imported path is a directory, the file ``default.nix`` in that
directory is loaded::

  $ cat ./default.nix
  {}: {x = "hello";}
  $ nix eval '(with import ./. {}; x)'
  "hello"

Module search path
~~~~~~~~~~~~~~~~~~

A name in angle brackets is treated as a directory name to be imported
and is searched in ``NIX_PATH``. For example, ``import <nixpkgs> {};``
searches for the ``nixpkgs`` directory in ``NIX_PATH`` and imports it.

Several nix programs provide a command line flag e.g. ``nix eval -I`` to
provide search path on command line.

Note that the syntax is like the ``include`` syntax in C.

Further Reading
---------------

With the knowledge of the fundamentals of the language you can now
proceed to the nix derivations guide.

Quick References
----------------

* https://nixos.wiki/wiki/Nix_Expression_Language
* https://nixcloud.io/tour/ A tour of Nix (language)
* https://medium.com/@MrJamesFisher/nix-by-example-a0063a1a4c55 Nix by example
* https://nix.dev/anti-patterns/language.html
