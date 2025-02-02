---
title: "First Steps with Nix - Building emacs"
date: 2021-08-08
author: Heinrich Hartmann
location: Stemwede
style: markdown
tags: nix, post
Draft: false
---
<style>
.markdown-container {
 width: 60em;
}
@media only screen and (max-width: 60em) {
    .markdown-container {
        text-align: left;
        width: 100%;
        margin: 0;
        margin-top: 3em;
        padding: 0.1em;
    }
}
.center {
    display: block;
    margin:auto
}
</style>

<img src="./nix.png" style="float:left; width:180px; margin:10px;"/>

[nix](https://nixos.org/) is a package manager that does things a little differently. I have
fascinated by nix ever since [Christine Koppelt](https://twitter.com/ckoppelt) told me about it a
few years back when we were working together in Munich.

The basic idea behind nix, is that all sofware is kept in a content addressable* store, and only the
exact dependencies needed are made available in a build, dev or runtime environment. The design has
some similarity with git and aims to solve similar problems as docker. From an abstract
perspective, nix presents the cleanest approach to building software that is currently available.

Here are a few of the selling points, that I hope to realize with nix:

1. 100% conflict free dependency management
2. (Almost) reproducible builds
3. Declaratively defined "cheap" run-time environments for applications, development and whole
   systems.
4. Create minimal VM or Docker images containing those environments

Unfortunately, nix still has a rather steep learning curve. I have been banging my head against the
table quite a few times over the past week, and this is AFTER I printed the manual and read most of
it :) In this note, I'll take you along for the journey of a first more serios experiment: building
the latest emacs, with a few extras enabled.

In the following sections we will actually create **four** different build variant that realize our
goal following slightly different approaches:

1. [Emacs From Scratch](#v1)
2. [Refactoring nixpkgs' Emacs](#v2)
3. [Overriding nixpkgs' emacs](#v3) 
4. [The emacs overlay](#v4)

If you are only interested in using the results, you can direcly skip to the [GitHub
repository](https://github.com/HeinrichHartmann/emacs-head.nix).

WARNING: This post assumes some basic familiarity with nix, to follow the examples in detail.
Nevertheless, I hope that this post will allow you to get some idea about what is possible,
and how things are done in the nix world.

*) The nix storage address is the hash of the reciept ("derivation") that was used to build the
artifact, and not the hash of the artifact itself. This is mainly since nix has to deal with builds
that are non-reproducible. We may see a content addressable nix variant in the near future.

## Available emacs versions

The usual way to install emacs with nix is to use the command:

```shell
; nix-env -iA nixpkgs.emacs
```

At the time of this writing, this will install
[emacs-27.2](https://search.nixos.org/packages?channel=unstable&from=0&size=50&sort=relevance&query=emacs).
This is the latest released version, but 4 months old by now and [5000+
commits](https://github.com/emacs-mirror/emacs/compare/emacs-27.2...master) behind master. We want
to run something a little bit more recent, ideally with [native
compilation](https://www.emacswiki.org/emacs/GccEmacs) (`--with-native-compilation`) and native JSON
support (`--with-json`) enabled.

# Variant 1 -- Emacs From scratch {#v1}

The first emacs package we are going to build is starting from scratch. I am using a .nix template
that is derived from this [nix
pill](https://nixos.org/guides/nix-pills/fundamentals-of-stdenv.html#idm140737319559808):

```nix
; cat emacs-from-scratch.nix
with {
  pkgs = import <nixpkgs> {}; # make nixpkgs available 
};

pkgs.stdenv.mkDerivation {
  name = "emacs-head";
  # ... here goes the receipt ...
}
```

This file already compiles, and gives us a (pretty useless) derivation, which we can view with:
```
nix show-derivation $( nix-instantiate -f emacs-from-scratch.nix )
```

## Step 1.1 -- Pull emacs sources from GitHub

To pull the sources from GitHub, we add the following statement to our .nix file:

```nix
  src = pkgs.fetchFromGitHub {
      owner = "emacs-mirror";
      repo = "emacs";
      rev = "4d439744685b6b2492685124994120ebd1fa4abb"; # git-SHA of curret master HEAD
      sha256 = "00vxb83571r39r0dbzkr9agjfmqs929lhq9rwf8akvqghc412apf";
  };
```

The sha256 content is a nix-specific hash of the downloaded tar file. The easiest way to get it is
to build the derivation without sha256 specified, and extract the expected sha from the error
message.

We build the package with
```
; nix-build emacs-from-scratch.nix
...
configuring
no configure script, doing nothing
building
build flags: SHELL=/nix/store/x0dcb2rxlzf32g0ddfkqqz1sfcyx4yay-bash-4.4-p23/bin/bash
There seems to be no "configure" file in this directory.
Running ./autogen.sh ...
./autogen.sh
...
Your system seems to be missing the following tool(s):
autoconf (missing)
```
and get an build-time (!) error that autoconf is missing.

To resolve the situation we make the following changes:

1. Add `preConfigure = "./autogen.sh"`, to generate a suitable configure file.
   (The necessity of this step took me WAY TOO LONG to figure out 😅).
   
2. Add `buildInputs = [ pkgs.autoconf ]` to make autoconf available in the build environment.

While at it we also add `configureFlags = ["--without-all"]` as to simplify our lives for the time being.

Hint: Use `nix-shell --pure emacs-from-scratch.nix` to drop into an interactive build environment.
The [build phases](https://nixos.org/manual/nixpkgs/stable/#sec-stdenv-phases) can be manually
executed with commands like `unpackPhase`. I used this to run `./configure --help` to get
information about the available configure flags.

## Step 1.2 - A first try ...

After resolving a first set of dependecny errors, we arrive a the following file:

```nix
with rec {
  pkgs = import <nixpkgs> {};
};

pkgs.stdenv.mkDerivation {
   name = "emacs-head";
   src = pkgs.fetchFromGitHub {
       owner = "emacs-mirror";
       repo = "emacs";
       rev = "4d439744685b6b2492685124994120ebd1fa4abb";
       sha256 = "00vxb83571r39r0dbzkr9agjfmqs929lhq9rwf8akvqghc412apf";
  };
  preConfigure = "./autogen.sh";
  configureFlags = ["--without-all"];
  buildInputs = [ pkgs.autoconf pkgs.texinfo pkgs.ncurses ];
}
```

The build proceeds through the configure phase, and fails after a few minutes with the following error:
```
/nix/store/x0dcb2rxlzf32g0ddfkqqz1sfcyx4yay-bash-4.4-p23/bin/bash: /bin/pwd: No such file or directory
```

`pwd(1)` is part of the GNU coreutils, which is available in the nix build environment by default.
It's however, not at its standard location `/bin/pwd` but in the nix store, in my case at:
`/nix/store/937f5738d2frws07ixcpg5ip176pfss1-coreutils-8.32/bin/pwd`

## Step 1.3 -- Patching the Makefiles

We have to patch the Makefiles in order to fix the issue. Taking some inspiration from the [nikpgs
emacs](https://github.com/NixOS/nixpkgs/blob/master/pkgs/applications/editors/emacs/generic.nix)
derivation, we add the following patches to our .nix file:

```nix
postPatch = pkgs.lib.concatStringsSep "\n" [
  ''
  for makefile_in in $(find . -name Makefile.in -print); do
    substituteInPlace $makefile_in --replace /bin/pwd pwd
  done
  substituteInPlace src/Makefile.in --replace 'RUN_TEMACS = ./temacs' 'RUN_TEMACS = env -i ./temacs'
  substituteInPlace lisp/international/mule-cmds.el --replace /usr/share/locale ${pkgs.gettext}/share/locale
  ''
];
```

With these patches the build succeeds and we have our first emacs.
```
; nix-build emacs-from-scrach.nix
...
/nix/store/qh9v8rnp8bjm18r7wz9cq7khhqf63wvw-emacs-head

; /nix/store/qh9v8rnp8bjm18r7wz9cq7khhqf63wvw-emacs-head/bin/emacs --version
GNU Emacs 28.0.50
...
```

## Step 1.4 -- Adding configure flags

We are now removing the `--without-all` configure flag, to arrive at a more featureful emacs variant.
Looking at the build errors, we need to add `pkgs.gnutls` and `pkgs.pkg-config` to the buildInputs, and get 
a working version. 

- For `--with-json` we need `pkgs.janson` as well.
- For `--with-native-compilation` we add `pkgs.libgccjit` and `pkgs.zlib`, and set the environment variables:
  ```nix
  NATIVE_FULL_AOT = "1";
  LIBRARY_PATH = "${pkgs.stdenv.cc.libc}/lib";
  ```

In all those cases, adding the entry to `buildInputs` was sufficient for the build process to pick
up the dependency. This means that `LD_FLAGS` and `CFLAGS` have been set accordingly without any
need for additional configuration. 

The final emacs-from-scratch.nix file looks as follows:

```nix
with rec {
  pkgs = import <nixpkgs> {};
};

pkgs.stdenv.mkDerivation {
  name = "emacs-head";
  src = pkgs.fetchFromGitHub {
      owner = "emacs-mirror";
      repo = "emacs";
      rev = "4d439744685b6b2492685124994120ebd1fa4abb";
      sha256 = "00vxb83571r39r0dbzkr9agjfmqs929lhq9rwf8akvqghc412apf";
  };
  # Environtment for --native-compilation
  NATIVE_FULL_AOT = "1";
  LIBRARY_PATH = "${pkgs.stdenv.cc.libc}/lib";

  # Dependencies
  buildInputs = [
    pkgs.autoconf pkgs.texinfo pkgs.ncurses # --without-all
    pkgs.gnutls pkgs.pkg-config # normal build
    pkgs.jansson # --with-json
    pkgs.zlib pkgs.libgccjit # --with-native-compilation
  ];

  postPatch = pkgs.lib.concatStringsSep "\n" [
    ''
    for makefile_in in $(find . -name Makefile.in -print); do
      substituteInPlace $makefile_in --replace /bin/pwd pwd
    done
    substituteInPlace src/Makefile.in --replace 'RUN_TEMACS = ./temacs' 'RUN_TEMACS = env -i ./temacs'
    substituteInPlace lisp/international/mule-cmds.el --replace /usr/share/locale ${pkgs.gettext}/share/locale
    ''
  ];
  preConfigure = "./autogen.sh";
  configureFlags = [ "--with-json" "--with-native-compilation" ];
}
```

To be fair, I don't expect `--with-native-compilation` to fully work at this point, since there are
a few [suspicious lisp
patches](https://github.com/NixOS/nixpkgs/blob/master/pkgs/applications/editors/emacs/generic.nix#L81)
in the upstream repository that we did not apply yet.

However, emacs compiles and starts-up fine: 
```
$( nix-build emacs-from-scratch.nix )/bin/emacs 
```

# Variant 2 - Refactoring nixpkgs' Emacs {#v2} 

To get a fully functional emacs version we want to leverage the community know-how, and avoid
discovering all the pit-falls the hard way.

The upstream emacs derivation available in nixpkgs is available here [here](https://github.com/NixOS/nixpkgs/tree/2129356a6405cc342e860d87623d3823e8ee3b8c/pkgs/applications/editors/emacs).
The following 3 files are relevant for the emacs build:

- [27.nix](https://github.com/NixOS/nixpkgs/blob/2129356a6405cc342e860d87623d3823e8ee3b8c/pkgs/applications/editors/emacs/27.nix) calls generic.nix and specifies some patches
- [generic.nix](https://github.com/NixOS/nixpkgs/blob/2129356a6405cc342e860d87623d3823e8ee3b8c/pkgs/applications/editors/emacs/generic.nix) specifies the source tar-ball, dependency and configure options
- [all-packages.nix](https://github.com/NixOS/nixpkgs/blob/2129356a6405cc342e860d87623d3823e8ee3b8c/pkgs/top-level/all-packages.nix) calls generic.nix with some overrides and registers the derivation in the `<nixpkgs>` name-space

This setup is quite complex, and involves a lot of indirection that is not needed for our purposes.
In the next steps, we will refactor these three files, and produce a single-file version that is
functionally identical to the above for the cases of graphical OSX builds and non-graphical Linux
builds.

## Step 2.1 - Consolidate nix source files

We will refrain from explaining the refactoring steps full detail. The interested reader
can alwas refer to the [commit log](https://github.com/HeinrichHartmann/emacs-head.nix/commits/master).
On a high level, we proceeded as follows:

1. Remove the all-package.nix dependency by providing an independent entry point emacs-refactored.nix of the form:
   ```nix
   with import <nipkgs> {}; callPackage ./27.nix { ... }
   ````
2. Merge 27.nix and generic.nix into emacs-refactored.nix by substituting file contents for the `import` calls.
3. Cleanup by inlining calls and specializing to the OSX/Linux configuration described above. 

After each step we check the file defines an identical build by using `nix-instantiate -f <filename>`.
If the retured hash value is identical, we have not changed the build.

## Step 2.2 - Pull latest sources

Next we change src to pull from the latest GitHub master, as we did above.
We fix the build by:

- Remove the patches as they no-longer seem to apply 
- Add `autoconf` and `texinfo` dependencies to buildInputs
- Run `./autogen.sh` from the pre-configure hook

## Step 2.3 Enable Native Compilation and native JSON

The upstream emacs package thankfully has already built-in nativeComp parameter!
I am pretty sure we should be able to set this with a command-line switch (`--args`), but I could not get this to work.
(What seems to work is to use the following nix expression: `nix-build -E 'with import <nixpkgs> {}; emacs.override { nativeComp = true; }'`)
We set this parameter to `true` to enable nativeComp. 

Native JSON support is activated by adding "--with-json" as configure flag. The jansson dependency is already speficified.

With this changes, `nix-build` goes right through an we should have a fully functional emacs `--with-json` and `--with--native-compilation` running!
To be sure, we check this by printing the lisp variable `system-configuration-options` which contains a copy of the build flags:

```
; $( nix-build emacs-refactored.nix )/bin/emacs -nw -q --batch --eval '(message system-configuration-options)'
--prefix=/nix/store/ ... --with-native-compilation --with-json
```

The complete build definition can be found
[here](https://github.com/HeinrichHartmann/emacs-head.nix/blob/master/emacs-refactored.nix).

# Variant 3 -- Overriding nixpkgs' emacs  {#v3}

In going through the above steps, one notes, that there were actually very few modifications to the
stock emacs package necessary to build the latest `emacs@head` version: We changed `src`, added a
`preConfigure` entry, and specified two more dependencies.

It is possible to apply those changes to upstream derivation, using the [package
overrides](https://nixos.org/manual/nixpkgs/stable/#chap-overrides) mechanism.
The resulting file looks like this:

```nix
; cat emacs-override.nix
with (import <nixpkgs> {});

(emacs-nox.override {
     nativeComp = true;
}).overrideAttrs (old : {
     pname = "emacs";
     version = "head";
     src = fetchFromGitHub {
        owner = "emacs-mirror";
        repo = "emacs";
        rev = "4d439744685b6b2492685124994120ebd1fa4abb";
        sha256 = "00vxb83571r39r0dbzkr9agjfmqs929lhq9rwf8akvqghc412apf";
     };
     patches = [];
     configureFlags = old.configureFlags ++ ["--with-json"];
     preConfigure = "./autogen.sh";
     buildInputs = old.buildInputs ++ [ autoconf texinfo ];
})
```

Most of the changes shoudl be self-explanatory. Note, that we use `.override` to override the
nativeComp arguments to `callPackage`, and `.overrideAttrs` to manipulate the internal of the
derivation.

This is the emacs version that I am currently using on my Linux machine. It allows me to leverage
the upstream maintenance efforts and only maintain a minimal set of changes that is necessary to
build the latest master, with my desired configuration.

# Variant 4 -- The emacs overlay  {#v4}

An even more powerful mechanism to "override" nixpkg's packages is the
[Overlay](https://nixos.org/manual/nixpkgs/stable/#chap-overlays) mechanism.


There is a community maintained [emacs-overlay](https://github.com/nix-community/emacs-overlay)
repository available, which seems like it does the job of building the latest emacs with the desired
configuration. Initially I discatded this overlay since it looked quite involved (requireing use of
NixOs or Home Manager), and does a fair bit more than we need for our set-goal. After having
worked through the above versions, I think I understand now how to navigate this complexity,
and produce a stand-alone nix-file ([GH](https://github.com/HeinrichHartmann/emacs-head.nix/blob/16ee7df0cf2092ffeb117d79d8bd89f1716834c7/emacs-overlay.nix)) 
that leverages this overlay:

```nix
; cat nix-overlay.nix
with rec {
  emacs-overlay = import (builtins.fetchTarball { url = https://github.com/nix-community/emacs-overlay/archive/master.tar.gz; });
  pkgs = import <nixpkgs> { overlays = [ emacs-overlay ]; };
};

pkgs.emacsGcc
```

The following command will now build the community maintained emacsGcc:
```
nix-build nix-overlay.nix
```

To install this emacs version, you could use:
```
nix-env -f nix-overlay.nix -i ".*"
```

Note that this version does not include the "--with-json" configuration, we have used before. In order
to add this, we would need to specify overrides as we did above.

# Conclusion

We have seen four different methods to install the latest-gratest emacs with a custom configuration.
This exercise has been a good learning experience for me. I hope that this writeup is somewhat helpful
for you as well in your journey to find yourself around with nix.

While working on this project, I have asked the nix community multiple times for support over the following channels:

- [Chat/Matrix](https://matrix.to/#/#community:nixos.org)
- [StackOverflow](https://stackoverflow.com/search?q=user%3A1209380+%5Bnix%5D)
- [Forum/Discourse](https://discourse.nixos.org/t/how-to-build-and-distribute-binary-packages-with-nix/14439)

In all cases, I got very helpful replies within a few minutes. This has been an outstanding experience!

What I took away from this exercise is that nix is a very powerful tool, that has the potential to
change how we build sofware and manage runtime environments. A lot of heavy lifting has been taken
place behind the scenes to make this possible. At this point in time, the user-facing APIs are still
very complicated, and sometimes feel rough. To get productive you have to dig into the nix
expression language, which is not all that complicated, but requires a certain time investment.

If you are a beginner and want to dig deeper, I would recommend to:

1. Watch some of the intro videos from [Burke Libbey's](https://twitter.com/burkelibbey) [YouTube Channel](https://www.youtube.com/channel/UCSW5DqTyfOI9sUvnFoCjBlQ/featured)
2. Work through the first few [Nix pills](https://nixos.org/guides/nix-pills/)
3. Spent some [quality time](https://www.heinrichhartmann.com/archive/quality-time.html) with the [Nix manual](https://nixos.org/manual/nix/stable/) and the [NixPkgs manual](https://nixos.org/manual/nixpkgs/stable/).

Also don't hesitate to get in touch with community via chat!

<hr/>
<script src="https://utteranc.es/client.js"
        repo="HeinrichHartmann/comments"
        issue-term="title"
        label="Comment"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
