# jdep

### A tool to help `javac` and `make` play nicely together

## General Information

This is `jdep` version 1.4, dated 19-July-2016.

Jdep is a tool for analyzing Java `.class` file dependencies, so that the
peculiar compilation behavior of many Java compilers can be tamed to be
compatible with conventional Unix style Makefiles.

The official source and documentation for jdep is maintained at:

https://github.com/FUDCo/jdep (i.e., here)

At the moment, I'm only distributing this tool in source form, open sourced
under the MIT license.

Bug reports, flames, questions and other feedback should be directed to
jdep-feedback@fudco.com

## Installation

1. Clone the `jdep` repo from https://github.com/FUDCo/jdep

2. Type `make` to compile the package.  It will produce two executable files in
the `bin` subdirectory: `jdep` and `touchp` (the latter is actually just a
simple shell script).

3. Copy the executables to wherever you put your installed executables.

## Supported Platforms

This tool has been tested on and is known to work on (and indeed is used
regularly on) every version of SPARC Solaris, Linux, FreeBSD, and Mac OSX I've
encountered as of this writing.  It will work with all Sun Java JDKs up through
JDK 8 (which is the most current version as of this writing).

More generally, I see no reason why `jdep` shouldn't be able to build and run
without modification on any sane Unix platform or system that supports the Unix
file API, as it doesn't do anything exotic at the OS level.  But I don't make
any promises, and the usual disclaimers (which are more verbosely and
legalistically explicated in the license terms included in the source
distribution) apply.

You are welcome to make whatever use of this code you like, but if you fix any
bugs or tweak it to work on a wider range of platforms, please do let me know
and I'll be happy to roll your changes back into the main distribution.

## What The Heck Is This?

(You don't need to read this section if all you want to know is how to use the
tool. You can skip ahead to the next section. On the other hand, it's not much
use knowing how to use the tool unless you understand the problem it's trying
to solve, so you might want to read this section anyway, even though it's a bit
long and rambling.)

This is my answer to the problem that Sun's Java compiler, `javac` does not
play nicely with `make`. This is also a problem with any other Java compiler
which attempts to emulate the behavior of `javac`, such as that provided with
OpenJDK. This is unspeakably irritating to those of us who would like to use
the Unix command line tools as our principal development environment to develop
Java code the same way we have always developed C or C++ or FORTRAN code or
indeed code in just about any compiled language *except* Java.

Unlike, for example, C, Java does not have header files. This is good, in that
it defines out of existence an entire class of C development bugs, wherein a
header and the source file(s) it corresponds to get out of sync with each
other. Instead, Java classes simply import the other classes they depend
on. However, this means that to compile a Java class you need to have compiled
the classes that it imports. But this introduces a couple of problems.

The first problem is that classes may be mutually dependent. That is, class X
may import class Y while at the same time class Y imports class X. The `javac`
compiler copes with this by requiring you to compile such classes together,
i.e., in a single invocation of the compiler. This effectively precludes
separate compilation in the sense that we have traditionally come to know it,
where we compile each source file individually and then link the resulting `.o`
files when done. This joint compilation requirement is the first step to making
`javac` incompatible with `make`.

The second problem is more subtle and happens when class A imports class B but
class B does *not* import class A. The authors of `javac` apparently recognized
that Java's importation rules hindered the operation of `make` to a degree, so
they incorporated a bunch of make-like behavior into `javac` itself.  Thus,
if you attempt to compile class A and it requires the importation of class B,
`javac` will go looking for `B.class`. If it fails to find it, it will then
look for `B.java` and implicitly (and silently) include it in the compilation.
This helpful behavior means that classes can get mysteriously recompiled when
you are not expecting it, resulting in all manner of surprising mayhem. The
mayhem ensues because although it understands to recompile B if necessary when
compiling A, it doesn't know to recompile A when B changes (though knowing when
to do the latter is the fundamental mission of `make` in the first place).
Attempting to reconcile this implicit compilation behavior with a set of file
dependencies that one can put into a makefile eventually reduces one to hair
pulling and ultimately to gibbering idiocy.

The most straightforward way out of this situation, the one which I took (until
I wrote `jdep`), and the approach taken by essentially everyone I know doing
Java development work using the conventional Unix toolchain, is to
simply always recompile everything whenever recompiling anything.  This has the
virtue of being extremely easy to do in a makefile, as well as being absolutely
foolproof from a dependency analysis standpoint.  Unfortunately, it is rather
awkward to manage in a project environment where different people are
responsible for different pieces of the source tree. Also, even with fast
computers it is painfully time consuming once a project develops into having a
large number of source files (which, in a non-trivial project, it eventually
will because Java wants you to put each class into its own source file).

After suffering with this situation for about 5 years, cursing at `javac` all
the while, I finally had a classic "Aha!" experience: from the perspective of
`make`, `javac` does not behave like a compiler but more like a linker. It's
just that it does its job using the pre-compilation (`.java`) files rather than
the post-compilation (`.class`) ones. This perspective leads to the following
`make` strategy: "compile" a `.java` file by `touch`ing the corresponding
`.class` file (this updates the last-modified timestamp on the `.class` file,
just as actually compiling it would), then "link" by running `javac` on all the
`.java` files whose `.class` files now need to be "linked" according to
`make`'s dependency analysis. This strategy relies on the assumption that it is
possible to have your makefile synthesize a command line by mapping a list of
`.class` files into a corresponding list of `.java` files. Fortunately, GNU
`make` readily does this sort of thing (I don't know about Sun's `make` or
others, but given that we do have GNU `make` there's no real reason to use
anything else anyway).

The final piece of the puzzle concerns file dependencies.  When making a C
program, the `.c` files will depend on the `.h` files, which you can either
keep track of by hand (which is tedious and error prone), or you can have an
automated tool keep track of the dependencies for you. In the case of C this is
very easy, since all you need to do is have the C compiler produce a list of
which `.h` files it included in the process of compiling a given `.c` file. The
GNU C compiler, `gcc`, does this in a very convenient way with the `-MD`
command line option (or, in practice, the `-MMD` option), which not only
produces a list of file dependencies but outputs it in the form of an actual
`make` dependency line (in a `.d` file) that can be included in your makefile
directly.

The case of Java is a bit more complex.  Remember, as we said, that Java
doesn't use header files. Instead, a given `.java` file depends, in effect, on
other `.java` files. There is no obvious inclusion hierarchy to follow as there
is in C.  Ideally, `javac` would spit out the same kind of dependency
information that `gcc` does, but it doesn't (and is unlikely ever to, in my
estimation, since this mode of use is really not what its creators intended).
Due to the several ways in which one can implicitly import a class without ever
naming it explicitly in an `import` declaration, writing a tool to extract the
dependency information from the Java source files yourself is not really
practical (or rather, if you succeed in doing it you will have done a large
fraction of the work towards writing your own Java compiler). However, once
`javac` has run, the dependency information we need is present in the resulting
`.class` files, which are very easy to read (indeed, were designed to be in a
machine-friendly format). This was even easier for me, as I already had a
program to read and dump `.class` files laying around that I had written years
ago when I was messing with Java compilers; it was a simple matter to modify it
into the program now called `jdep`.

Here is what `jdep` does: it reads a batch of `.class` files and produces a
corresponding set of `.d` files suitable for inclusion into a GNU makefile.
These `.d` files define the dependencies used to drive the "compilation" phase
of the make process described above, in which compilation is simulated using
`touch` (actually, using `touchp`, but that's a minor detail we'll get to in a
moment).

## Using `jdep` to enable using `javac` with `make`

By convention, the source for a Java class `Foo` is placed in a file named
`Foo.java`.  Moreover, this file is generally located according the class's
package in a file directory tree whose hierarchical structure matches that of
the overall package hierarchy. These conventions are not so much enforced by
the compiler as they are expected by it, in the sense that if you deviate from
them it will get confused and not always do the right thing. Consequently, we
assume you will continue to follow these conventions when using `jdep` and
`make`.  (None of this applies, of course, to IDEs, which keep track of all
this stuff in a database or project file of some kind. But if I wanted to use
an IDE I wouldn't be here. The problem with Integrated Development Environments
is they're so darned Integrated. But I digress.)

The discussion that follows describes setting up a makefile. For expository
purposes, the `make` variable `JAVA_DIR` is assumed to be set to the pathname
of the directory root of the file tree where Java source files live, while the
variable `CLASS_DIR` fulfills the same role with respect to compiled class
files, and `DEP_DIR` similarly with respect to `.d` files.  Thus for example,
the class `bar.baz.Foo` would have its source file be taken from
`$(JAVA_DIR)/bar/baz/Foo.java`, while its compiled class file would end up in
`$(CLASS_DIR)/bar/baz/Foo.class` and its dependencies would be described in
`$(DEP_DIR)/bar/baz/Foo.d`.

As described in more detail in the preceding section, the key trick to making
this work is to think of compilation with `javac` as the "link" phase of the
build process, and to "compile" by using `touch`.

An example of how you set all this up is included in the subdirectory `example`
of the `jdep` package.

In the example, we define the list of source files:

```
EXAMPLE_SRC = $(shell cd $(JAVA_DIR); find com -name '*.java')
```

I used `find` to enumerate the source files; however, you can get your list of
source files however you like, including by just manually entering them
directly.

We then define the list of class files by doing a pattern transformation on the
list of source files:

```
EXAMPLE_CLA = $(EXAMPLE_SRC:%.java=$(CLASS_DIR)/%.class)
```

The list of make dependency files is defined similarly:

```
EXAMPLE_DEP = $(EXAMPLE_SRC:%.java=$(DEP_DIR)/%.d)
```

"Compile" using `touchp`. I use the following implicit `make` rule:

```
$(CLASS_DIR)/%.class: $(JAVA_DIR)/%.java
        touchp $@
```

Note that I use `touchp` rather than `touch`. This is a simple shell script
that is included as part of the `jdep` package. The analogy is to `mkdir`:
`touchp` is to `touch` as `mkdir -p` is to `mkdir`. Normally, `touch` will
create a zero-length file if the file being touched does not yet exist.
However, if the directory the resulting file would placed in does not already
exist, `touch` will fail. In contrast, `touchp` will create the directory path
down to the point needed for the `touch` to succeed, just as `mkdir -p` will
create an entire directory path.

"Link" using `javac`, using a `make` rule where the ultimate target depends on
the list of class files:

```
$(MODULE_NAME_TARGET): $(EXAMPLE_CLA)
```

in this rule, execute a `javac` command line like:

```
        javac -d $(CLASS_DIR) -classpath $(CLASS_DIR) $(?:$(CLASS_DIR)/%.class=$(JAVA_DIR)/%.java)
```

`make` will bind the variable `$?` to the list of class files that were newer
than the target file, i.e., the ones that got touched in the "compilation"
phase.  This list of class files is converted back into a list of source files
using another pattern transformation.

Next, as a further part of the "link" rule, run `jdep` to update the `.d` files
for any classes that got compiled:

```
        jdep -c $(CLASS_DIR) -j $(JAVA_DIR) -d $(DEP_DIR) $?
```

Finally, produce an updated version of the ultimate target file. I create a
`.jar` file of all the `.class` files:

```
        cd $(CLASS_DIR); jar cf ../$@ `find com -name '*.class'`
```

but if you prefer to work with a loose collection of .`class` files you could
instead just have your ultimate target be a marker file that you `touch`:

```
        touch $@
```

Finally, be sure to include the dependency files that `jdep` generated in
earlier runs of `make`:

```
-include $(EXAMPLE_DEP)
```

## One limitation of this approach

The scheme described here absolutely depends on the 1-to-1 correspondence
between source files and class files. However, the compilation rules for Java
permit you in some cases to violate this principle: you are allowed to place
more than one non-public (i.e., package scoped) class in a source file.  Though
it's arguably a bad practice anyway, if you use `jdep` you simply can't do
that at all.  You *can* use inner classes, however, since there is enough
information in a class file to track an inner class back to its outer class,
and inner classes are nearly always preferrable to embedded package classes.

## `jdep` Command Reference

#### Synopsis:

`jdep` *option*... *file*...

Each *file* should be a Java `.class` file, which may be specified either with
or without the trailing `.class` portion of the name.

The program accepts the following options:

##### `-a`

Include all packages in the dependency information generated. By
default, `jdep` will omit packages whose package names begin
with `java.`, `javax.`, or `com.sun.` since these
contain library classes that your makefile won't know how to build anyway.

##### `-e` *package*

Exclude the package *package* from the dependency information
generated. This option may be specified as many times as you wish to exclude as
many packages as you wish. Use this to exclude library packages in addition to
the defaults mentioned in the description of the `-a` flag.

##### `-i` *package*

Include the package *package* in the dependency information generated.
This option may be specified as many times as you wish to include as many
packages as you wish. If this option is not used, `jdep` will include
all packages not explicitly excluded with the `-e` option. However, if
at least one `-i` option is specified, then `jdep` will only
include those packages it was specifically told to include.

##### `-h`

Print a summary of the command options and then exit.

##### `-c` *cpath*

Use *cpath* as the base directory for `.class` files. This path
will be used to locate class files for inner classes.

##### `-j` *jpath*

Use *jpath* as the base directory for `.java` files. This path
will be prepended to source file paths in the resultant dependency
files.

##### `-d` *dpath*

Use *dpath* as the base directory for `.d` files. This path will
be used to generate the output file pathnames for the various
dependency files which `jdep` produces.


## Change history

#### Version 1.1

Added the `-e' command line option to explicitly specify which packages will be
excluded from analysis.

#### Version 1.2

Added the `-i' command line option to explicitly specify which packages will be
included in the analysis.

Cleaned up the text of the usage information output by the `-h' option.

#### Version 1.3:

Changed the license from the GPL to the MIT license.

Documentation refresh.

#### Version 1.4

Migrate onto GitHub.

## Todo

There should be a proper man page for `jdep`.

`touchp` is kind of a hack.  It would be great if it could be dispensed with,
perhaps if somebody were to add the `-p` flag to `touch` (hint, hint).
