# Nix

## The ecosystem

* `Nix Language`
* `Nix` Package Manager
* `Nixpkgs` the package repository
* `NixOS` a linux distribution based on nixpkgs

* `HomeManager` user configuration manager
* `Nix-darwin` nix-like configuration for macOS


## Termonology

### Derivation

A derivation is the result of aggregating a information required to build software.
It contains information about how and where to pull the source code from, as well as steps on how to compile or package the software.
Also dependencies and configuration options are included.

Derivations are distinguished by hashing all information to an unique identifier.
Nix does not rely on the host system for it's assumptions so it should work across systems.

Derivations can reference other derivations and form a directed acyclic graph.

Dependencies of a derivation are referred to as `build closure`.


### Fixed Output Derivations (FODs)

Derivations that are leaves in the DAG and are defined by their content.
As a leave they do not refer to other derivations.
FODs contain hashes.
A hash ensures the reproduceability, as FODs can have access to the internet while fetching content.
`fetch*` of nix builtins are are used to create FODs.


### Input-Addressed Derivations (IA Derivations)

Regularly referenced as just `Derivations`. 
Built using `stdenv.mkDerivation` and `build*`


### Content-Addressable Derivations (CA Derivations)

A hybrid of FODs and IA derivations that address the rebuild problem of derivations.
Any patch in an IAD will cause cascading affect to all derivations using it to be rebuild.
Instead of rebuilding CA derivations allow to just update the reference to the patched dependency.


## Creating Derivations

The creation of derivations `instantiatiation` or `evaluation`.
Every package of nixpkgs refers to a derivation.

In nix `< 2.4`
```bash 
nix-instantiate '<nixpkgs>' -A hello
```

In nix `>= 2.4`
```bash 
nix eval nixpkgs#hello.drvPath
```
Requires `experimental-features = nix-command flakes` in `~/.config/nix/nix.conf`.

To inspect the content of the derivation use
```bash 
nix derivation show /nix/store/vg2kr2hqr1b95s0wjz6nlaakh6vdsfaz-hello-2.12.1.drv
```

Derivations having the same hash indicate the equal derivation.

A `.drv` marks a derivation.
This is not the built derivation only a build plan.


## Realise a derivation

Building of a derivation is referred to as `realisation`.
Derivation are a construction blueprint for an artifact.
Realisation is the execution of that blueprint.
```bash 
nix-store --realise /nix/store/vg2kr2hqr1b95s0wjz6nlaakh6vdsfaz-hello-2.12.1.drv

# using flakes
nix build nixpkgs#hello
```

The `nix-build` and `nix build` perform instantiation and realization.


## Entering temporary shell

Start a temp shell that has git, neovim and nodejs with
```bash
nix-shell -p git neovim nodejs
```

Downsides of this approach are that the version are not pinned and no further customization can be made here.

The better approach is to use a `shell.nix` file.
```nix 
let
  nixpkgs = fetchTarball "https://github.com/NixOS/nixpkgs/tarball/nixos-23.11";
  pkgs = import nixpkgs { config = { }; overlays = [ ]; };
in

pkgs.mkShell {
  packages = with pkgs; [
    git
    neovim
    nodejs
  ];
}
```

When running 
```bash 
nix-shell
```

Nix will automatically look for the `shell.nix` and use it to configure the temp shell.

To export shell environment variables use 
```nix 
let
  nixpkgs = fetchTarball "https://github.com/NixOS/nixpkgs/tarball/nixos-23.11";
  pkgs = import nixpkgs { config = { }; overlays = [ ]; };
in

pkgs.mkShell {
  packages = with pkgs; [
    git
    neovim
    nodejs
  ];

  GIT_EDITOR = "${pkgs.neovim}/bin/nvim"; # to export this env variable
}
```

There are protected shell variables e.g. `PS1` that can only be overridden by using `shellHook`.
The `shellHook` allows to run start-up commands in the shell.
```nix 
#...
  shellHook = ''
    git status
  '';
#...
```

## Nix Language

Eval a nix file with
```shell
nix-instantiate --eval file.nix
```

Attribute set
```nix
{
  string = "hello";
  integer = 1;
  float = 3.141;
  bool = true;
  null = null;
  list = [ 1 "two" false ];
  attribute-set = {
    a = "hello";
    b = 2;
    c = 2.718;
    d = false;
  }; # comments are supported
}
```

Recursive declaration of a attribute set
```nix
rec {
  one = 1;
  two = one + 1;
  three = two + 1;
}
```

Let expression aka let binding
```nix
let
  a = 1;
in
a + a
```

Attribute access
```nix
let
  attrset = { x = 1; };
in
attrset.x
```

Nested attribute access
```nix
let
  attrset = { a = { b = { c = 1; }; }; };
in
attrset.a.b.c
```

Nested attribute assignment
```nix 
{ a.b.c = 1; }

{ a = { b = { c = 1; }; }; }
```

With expression, scope receiver
```nix 
let
  a = {
    x = 1;
    y = 2;
    z = 3;
  };
in
with a; [ x y z ];
```

Shorthand assignment with inherit
```nix 
let
  x = 1;
  y = 2;
in
{
  inherit x y;
}

{ x = 1; y = 2; }
```

Inherit with scope
```nix 
let
  a = { x = 1; y = 2; };
in
{
  inherit (a) x y;
}

{ x = 1; y = 2; }
```

Inherit in let expressions
```nix 
let
  inherit ({ x = 1; y = 2; }) x y;
in [ x y ]

[1 2]
```

String interpolation, only works with character strings
```nix 
let
  name = "Nix";
in
"hello ${name}"

"hello Nix"
```

If an attribute set has an `outPath` attribute and the set is used in a string interpolation the outpath will be used.
```nix 
a = { outPath = "foo"; }
"${a} bar"
# "foo bar"
```

File system paths
```nix 
/absolute/path
```

Relative file system paths
```nix 
./relative

# or

relative/path
```

Lookup path
```nix 
<nixpkgs>

/nix/var/nix/profiles/per-user/root/channels/nixpkgs
```

Lookup path sub directory
```nix 
<nixpkgs/lib>
```

Multi-line strings
```nix 
''
multi
line
string
''
```

Functions only take a single argument, arguments and body are separated by `:`
```nix 
x: x + 1
```

Multi-args are archive by currying
```nix 
x: y: x + y
```

Using argument sets
```nix 
{ a, b }: a + b
```

Optional arguments with default values
```nix 
{ a, b ? 0 }: a + b
```

Allow additional arguments
```nix 
{ a, b, ...}: a + b
```

Name attributes
```nix 
args@{ a, b, ... }: a + b + args.c

# or

{ a, b, ... }@args: a + b + args.c
```

Assigning a name to a function
```nix 
let
  f = x: x + 1;
in f
```

Function application
```nix 
let
  f = x: x + 1;
in f 1

# or

let
  f = x: x.a;
in
f { a = 1; }
```

Pass arguments by name
```nix 
let
  f = x: x.a;
  v = { a = 1; };
in
f v
```

Immediate application
```nix 
(x: x + 1) 1
```

Destructuring arguments
```nix 
{a, b}: a + b
```

Application and destructuring
```nix 
let
  f = {a, b}: a + b;
in
f { a = 1; b = 2; }
```

Default values and application
```nix 
let
  f = {a, b ? 0}: a + b;
in
f { a = 1; }
```

@-Pattern
```nix 
let
  f = {a, b, ...}@args: a + b + args.c;
in
f { a = 1; b = 2; c = 3; }
```

Function libraries

* `builtins` - is the std
```nix 
builtins.toString
```

Apply right on import
```nix 
import ./file.nix 1
```

`lib` attribute set of `nixpkgs`

`pkgs.lib` implemented in nix language
```nix 
let
  pkgs = import <nixpkgs> {};
in
pkgs.lib.strings.toUpper "lookup paths considered harmful"
```

Impure function to fetch data

* `builtins.fetchurl`
* `builtins.fetchTarball`
* `builtins.fetchGit`
* `builtins.fetchClosure`

Some unpack automatically
```nix 
builtins.fetchTarball "https://github.com/NixOS/nix/archive/7c3ab5751568a0bc63430b33a5169c5e4784a0ff.tar.gz"
```

Derivations are the core concept of nix.
The nix language is used to describe derivations.
Derivations are ran to produce build results.
Build results can be passed to other derivations.
`derivations` and `stdenv.mkDerivation` are used to create derivations.
Derivations denote something that will be eventually built.
It returns an attribute set, that can be used in string interpolations.

Derivations are just attributes sets with special attributes.
To turn an attribute set into a derivation the `builtins.derivation` function can be used.
Directly calling this function is discouraged and `stdenv.mkDerivation` should be used instead.
There are also other `builders` with sane defaults for `builtins.derivation`.

Nix language support `if/else` conditionals
```nix 
extension = if stdenv.isDarwin then ".dylib" else ".so";
```

The `callPackage` function calls a function with it's appropriate dependencies.


## Worked Examples

### Shell Environments

```nix 
{ pkgs ? import <nixpkgs> {} }:       # a function, with default lookup import
let
  message = "hello world";            # finding of the value to the name `message`
in
pkgs.mkShell {
  buildInputs = with pkgs; [ cowsay ];# add cowsay to build inputs
  shellHook = ''
    cowsay ${message}                 # use cowsay in shell hook with message
  '';
}
```

### NixOS configuration

```nix 
{ config, pkgs, ... }: {                          # a function, requiring at least config, pkgs and more
  imports = [ ./hardware-configuration.nix ];     # just a string list, not `import`
  environment.systemPackages = with pkgs; [ git ];# sets attribute to list with git
  # ...
}                                                 # returns an attribute set
```

### Package

```nix 
{ lib, stdenv, fetchurl }: # a function requiring lib, stdenv and fetchurl
stdenv.mkDerivation rec { # use rec to allow reference to pname and version
  pname = "hello";
  version = "2.12";
  src = fetchurl { # fetchurl to fetch artifact
    url = "mirror://gnu/${pname}/${pname}-${version}.tar.gz"; # self reference
    sha256 = "1ayhp9v4m4rdhjmnl2bq3cibrbqqkgjbl3s7yk2nhlh8vj3ay16g";
  };
  meta = with lib; {
    license = licenses.gpl3Plus; # set pgl3plus as license
  };
} # returns result of mkDerivation
```


## Packaging existing software with Nix

> A package is an informally defined Nixpkgs concept referring to a Nix derivation representing an installation of some project. Packages have mostly standardised attributes and output layouts, allowing them to be discovered in searches and installed into environments alongside other packages.

Use `mkDerivation` and `nix-build hello.nix`.
Setup a `default.nix` to declare a default environment for `nix-build`.
Now `nix-build -A hello` can be used, and `lib` will be passed to `hello.nix` via `callPackage`.

Until we know the actual hash of a `fechzip`-call use `lib.fakeSha256` after the initial call the actual hash will be known and can be used.

Hashed can be pre-fatched using 
```bash
nix-prefetch-url \
 --unpack https://github.com/atextor/icat/archive/refs/tags/v0.5.tar.gz \
 --type sha256
```

If additional installation is required before build use `installPhase`.
The output directory of the derivation is stored in the `$out` and can be referenced. There are a bunch of [phases](https://nixos.org/manual/nixpkgs/stable/#sec-stdenv-phases).


## Working with local files

To build a local derivation, the files required for the build needs to be accessible by the `builder`.
The default builder executable can only read from the Nix store.

A `file set` represents a collection of local files.
Various  functions allow to create, compose and manipulate files.

Setup a repl with nix 23.11 using
```bash 
nix repl -f https://github.com/NixOS/nixpkgs/tarball/master
```
Then
```nix 
fs = lib.fileset
```

To print all files included in the current file set
```nix
fs.trace ./. null
```

`null` is applied to an optional path here.

Use `lib.fileset.toSource` to store files into the Nix store.
These files can then be used in `mkDerivation` e.g. as the `src` argument.
Storing the files in the store first is required as the builder can only access the store.

When using `./.` to include the current directory, the `result` symlink will cause a new artifact hash each build, which is not desired.
To prevent this use the `difference` function to subtract files.
For potentially missing files use the `maybeMissing` function.

Any change to an included file will trigger a rebuild of the derivation.
Use `unions` and `difference` to exclude a set of files from the fileset.
```nix 
fs.difference ./. (fs.unions [(fs.maybeMissing ./result) ./default.nix ./build.nix ./nix]);
```

Another approach is to use `fileFilter` to exclude files of a certain type.
```nix 
fs.fileFilter (file: lib.hasSuffix ".nix" file.name) ./.
```

`uninons` is also usable to explicitly include files e.g.
```nix 
fs.unions [
    ./hello.txt
    ./world.txt
    ./build.sh
    (fs.fileFilter
      (file:
        lib.hasSuffix ".c" file.name
        || lib.hasSuffix ".h" file.name
      )
      ./src
    )
  ];
```

To match files tracked by git use `gitTracked`.
To not use to many files use `intersection` to only include files that are in both filesets.


## Building and running Docker images

Nixpkgs provide `dockerTools` to create docker images.
Using `pkgs.dockerTools.buildImage`.
Does not work well on apple silicon atm.


## Modules

See the [Nix module spec](https://nixos.wiki/wiki/NixOS_modules) for reference.

A module has the following syntax
```nix 
{
  imports = [
    # Paths to other modules.
    # Compose this module out of smaller ones.
  ];

  options = {
    # Option declarations.
    # Declare what settings a user of this module module can set.
    # Usually this includes an "enable" option to let a user of this module choose.
  };

  config = {
    # Option definitions.
    # Define what other settings, services and resources should be active.
    # Usually these are depend on whether a user of this module chose to "enable" it
    # using the "option" above. 
    # You also set options here for modules that you imported in "imports".
  };
}
```

A standard module as a function accepting an attribute set
```nix 
# default.nix
{ ... }:
{
}
```

Options can be declared using `mkOption`
```nix 
{ lib, ... }: {
 options = {
   scripts.output = lib.mkOption {
     type = lib.types.lines;
   };
 };
}
```

Use `lib.types` as argument to the `type` parameter of `mkOption` to type the option.
To perform type checking on the `default.nix` create an `eval.nix` like
```nix 
let
  nixpkgs = fetchTarball "https://github.com/NixOS/nixpkgs/tarball/nixos-23.11";
  pkgs = import nixpkgs { config = {}; overlays = []; };
in
pkgs.lib.evalModules {
  modules = [
    ./default.nix
  ];
}
```

This also evaluate the module and merges values into the final attribute set.

Evaluate using
```bash 
nix-instantiate --eval eval.nix -A config.scripts.output
```
This this parses and evaluates `default.nix` using `eval.nix`.



## Nix Pills


### Nix Environment

To install something into the nix environment use 
```bash 
nix-env -i hello
```

This installs hello for the nix user and creates a new version of the nix profile.
To see the profile generations run 
```bash 
nix-env --list-generations
```

To list installed derivations run
```bash 
nix-env -q
```

To rollback a generation use 
```bash 
nix-env --rollback
```

To switch to a specific generation in any direction use
```bash 
nix-env -G 3
```


### Querying the store

Show the dependencies of `hello`
```bash 
nix-store -q --references `which hello`
```

Show dependencies requiring `hello`
```bash 
nix-store -q --referrers `which hello`
```

The closures of a derivation is all it's dependencies.
View the closure of `man` with
```bash 
nix-store -qR `which man`
```

To improve the visualization use
```bash 
nix-store -q --tree `which man`
```

To view the dependency tree of the current profile use
```bash 
nix-store -q --tree ~/.nix-profile
```


### Derivation

In the repl
```nix 
d = derivation { name = "some-name"; builder = "some-builder"; system = "some-system"; }
d
<<derivation .../...-some-name.drv>>
```

Nothing was build yet only a `.drv` file was created.
The `.drv` file contains information about how to `realise` the derivation.
To look at the derivation use
```bash 
nix derivation show /nix/store/.../...-something.drv
```

The attribute set returned by `derivation` can be introspected in the REPL with
```nix 
builtins.attrNames d

[ "all" "builder" "drvAttrs" "drvPath" "name" "out" "outPath" "outputName" "system" "type" ]
```

The `type` attribute is special and marks this attribute set as a derivation.

Attributes added to the derivation attribute set will be available in the environment variables 
```nix 
simple = derivation { name = "simple"; builder = "${bash}/bin/bash"; args = [ ./simple_builder.sh ]; gcc = gcc; coreutils = coreutils; src = ./simple.c; system = builtins.currentSystem; }
```
Here `gcc` and `coreutils` can be referenced in the builder script via `$gcc` and `$coreutils` the same is true for `$src`.

The equivalent using `pkgs.stdenv.mkDerivation` would look like
```nix 
let
  pkgs = import <nixpkgs> {};
in
  pkgs.stdenv.mkDerivation {
    name = "simple";
    builder = "${pkgs.bash}/bin/bash";
    args = [ ./simple_builder.sh ];
    gcc = pkgs.gcc;
    coreutils = pkgs.coreutils;
    src = ./simple.c;
    system = builtins.currentSystem;
}
```

Then use 
```bash 
nix-build simple.nix
```

This performs `nix-instantiate` and `nix-store -r`.
So first the derivation is created by parsing and evaluation the `simple.nix` file.
The second step is performing the actual build using the derivation (`.drv`) file.

The assignments can be simplified using `inherit`
```nix 
let
  pkgs = import <nixpkgs> {};
in
  pkgs.stdenv.mkDerivation {
    name = "simple";
    builder = "${pkgs.bash}/bin/bash";
    args = [ ./simple_builder.sh ];
    inherit (pkgs) gcc coreutils;
    src = ./simple.c;
    system = builtins.currentSystem;
}
```

The `buildInputs` variable is a space separated list of all derivations out path required by the current derivation.

The `//` operator is a union-operator for attribute sets.

Nix ARchive (`NAR`) files is a deterministic archive format.
NAR-files can be created using `nix-store --dump` and `nix-store --restore`.

Derivations mentioned in the `out`-path of a derivation will become runtime dependencies.
This is determined in 3 steps
1. Dump the derivations as NAR. Works for single files and directories.
1. For each build dependency `.drv` and its relative out path, search contents of the NAR for this out path.
1. Any match will be a runtime dependency

If nix finds out path in the [`ld rpath`](https://en.wikipedia.org/wiki/Rpath) it will make the derivation a runtime dependency.
This is sometimes wrong and can be resole using [`patchelf`](https://github.com/NixOS/patchelf).
In case of the `hello.nix` derivation it will still depend on `gcc` due to debug symbols.
To remove the debug symbols [`strip`](https://linux.die.net/man/1/strip) can be utilized.


## Garbage Collections

Run the garbage collection with
```bash 
nix-collect-garbage
```

To print out all GC-roots of the nix-store run
```bash 
nix-store --gc --print-roots
```

Anything that is not referenced by a GC-root will be garbage collected.

An indirect root will be create by any `result` symlink.
These can be cleaned up by removing the `result` link or by removing it from `/nix/var/nix/gcroots/auto`.

To update every thing and remove everything old run
```bash 
nix-channel --update
nix-env -u --always
rm /nix/var/nix/gcroots/auto/*
nix-collect-garbage -d
```

## Flakes

Flakes are unit of packaging Nix code in reproducible and discoverable way.
A flake is a filesystem tree that contains a `flake.nix` at root.
The `flake.nix` contains meta data about the flake and the dependencies (`inputs`) and `outputs`.

An example `flake.nix` looks as follows
```nix 
{
  description = "A flake for building Hello World";

  inputs.nixpkgs.url = "github:NixOS/nixpkgs/nixos-20.03";

  outputs = { self, nixpkgs }: {

    packages.x86_64-linux.default =
      # Notice the reference to nixpkgs here.
      with import nixpkgs { system = "x86_64-linux"; };
      stdenv.mkDerivation {
        name = "hello";
        src = self;
        buildPhase = "gcc -o hello ./hello.c";
        installPhase = "mkdir -p $out/bin; install -t $out/bin hello";
      };

  };
}
```

## Home Manager

The [Home Manager](https://nix-community.github.io/home-manager/index.html) is a nix derivation/flake that can be use to manage linux/unix/macos dotfiles.

## Nix Darwin

The [nix-darwin](https://github.com/LnL7/nix-darwin) as a wrapper around nixpkgs to make is compatible for macos.
