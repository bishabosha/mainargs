# mainargs

MainArgs is a small, dependency-free library for command line argument parsing.

- [Usage](#usage)
  - [Parsing Main Method Parameters](#parsing-main-method-parameters)
  - [Multiple Main Methods](#multiple-main-methods)
  - [Parsing Case Class Paramters](#parsing-case-class-parameters)
  - [Re-using Argument Sets](#re-using-argument-sets)
  - [Annotations](#annotations)
  - [Customization](#customization)
  - [Custom Argument Parsers](#custom-argument-parsers)

# Usage

## Parsing Main Method Parameters

You can parse command line arguments and use them to call a main method via
`ParserForMethods(...)`:

```scala
package testhello
import mainargs.{main, arg, ParserForMethods}

object Main{
  @main
  def run(@arg(short = 'f', doc = "String to print repeatedly")
          foo: String,
          @arg(name = "my-num", doc = "How many times to print string")
          myNum: Int = 2,
          @arg(flag = true, doc = "Example flag")
          bool: Boolean) = {
    println(foo * myNum + " " + bool)
  }
  def main(args: Array[String]): Unit = ParserForMethods(this).runOrExit(args)
}
```

```bash
$ ./mill testhello -f hello
hellohello false

$ ./mill testhello -f hello --my-num 3
hellohellohello false

$ ./mill testhello -f hello --my-num 3 --bool
hellohellohello true

$ ./mill testhello --wrong-flag
Missing argument: --foo <str>
Unknown argument: "--wrong-flag"
Expected Signature: run
  -f --foo <str>  String to print repeatedly
  --my-num <int>  How many times to print string
  --bool          Example flag
```

Setting default values for the method arguments makes them optional, with the
default value being used if an explicit value was not passed in from the
command-line arguments list.

After calling `ParserForMethods(...)` on the `object` containing your `@main`
methods, you can call the following methods to perform the argument parsing and
dispatch:

### runOrExit

Runs the given main method if argument parsing succeeds, otherwise prints out
the help text to standard error and calls `System.exit(1)` to exit the proess

### runOrThrow

Runs the given main method if argument parsing succeeds, otherwise throws an
exception with the help text

### runEither

Runs the given main method if argument parsing succeeds, returning `Right(v:
Any)` containing the return value of the main method if it succeeds, or `Left(s:
String)` containing the error message if it fails.

### runRaw

Runs the given main method if argument parsing succeeds, returning
`mainargs.Result.Success(v: Any)` containing the return value of the main method
if it succeeds, or `mainargs.Result.Error` if it fails. This gives you the
greatest flexibility to handle the error cases with custom logic, e.g. if you do
not like the default CLI error reporting and would like to write your own.

## Multiple Main Methods

Programs with multiple entrypoints are supported by annotating multiple `def`s
with `@main`. Each entrypoint can have their own set of arguments:

```scala
package testhello2
import mainargs.{main, arg, ParserForMethods}

object Main{
  @main
  def foo(@arg(short = 'f', doc = "String to print repeatedly")
          foo: String,
          @arg(name = "my-num", doc = "How many times to print string")
          myNum: Int = 2,
          @arg(flag = true, doc = "Example flag")
          bool: Boolean) = {
    println(foo * myNum + " " + bool)
  }
  @main
  def bar(i: Int,
          @arg(doc = "Pass in a custom `s` to override it")
          s: String  = "lols") = {
    println(s * i)
  }
  def main(args: Array[String]): Unit = ParserForMethods(this).runOrExit(args)
}
```

```bash
$ ./mill testhello2
Need to specify a sub command: foo, bar

$ ./mill testhello2 foo -f hello
hellohello false

$ ./mill testhello2 bar -i 10
lolslolslolslolslolslolslolslolslolslols
```

## Parsing Case Class Parameters

If you want to construct a configuration object instead of directly calling a
method, you can do so via `ParserForClass[T]` and `constructOrExit:

```scala
package testclass
import mainargs.{main, arg, ParserForClass}

object Main{
  @main
  case class Config(@arg(short = 'f', doc = "String to print repeatedly")
                    foo: String,
                    @arg(name = "my-num", doc = "How many times to print string")
                    myNum: Int = 2,
                    @arg(flag = true, doc = "Example flag")
                    bool: Boolean)
  def main(args: Array[String]): Unit = {
    val config = ParserForClass[Config].constructOrExit(args)
    println(config)
  }
}
```
```bash
$ ./mill testclass --foo "hello"
Config(hello,2,false)

$ ./mill testclass
Missing argument: --foo <str>
Expected Signature: apply
  -f --foo <str>  String to print repeatedly
  --my-num <int>  How many times to print string
  --bool          Example flag
```

`ParserForClass[T]` also provides corresponding `constructOrThrow`,
`constructEither`, or `constructRaw` methods for you to handle the error cases
in whichever style you prefer.

## Re-using Argument Sets

You can share arguments between different `@main` methods by defining them in a
`@main case class` configuration object with an implicit `ParserForClass[T]`
defined:

```scala
package testclassarg
import mainargs.{main, arg, ParserForMethods, ParserForClass}

object Main{
  @main
  case class Config(@arg(short = 'f', doc = "String to print repeatedly")
                    foo: String,
                    @arg(name = "my-num", doc = "How many times to print string")
                    myNum: Int = 2,
                    @arg(flag = true, doc = "Example flag")
                    bool: Boolean)
  implicit def configParser = ParserForClass[Config]

  @main
  def bar(config: Config,
          @arg(name = "extra-message")
          extraMessage: String) = {
    println(config.foo * config.myNum + " " + config.bool + " " + extraMessage)
  }
  @main
  def qux(config: Config,
          n: Int) = {
    println((config.foo * config.myNum + " " + config.bool + "\n") * n)
  }

  def main(args: Array[String]): Unit = ParserForMethods(this).runOrExit(args)
}
```

```bash

$ ./mill testclassarg bar --foo cow --extra-message "hello world"
cowcow false hello world

$ ./mill testclassarg qux --foo cow --n 5
cowcow false
cowcow false
cowcow false
cowcow false
cowcow false
```

This allows you to re-use common command-line parsing configuration without
needing to duplicate it in every `@main` method in which it is needed. A `@main
def` can make use of multiple `@main case class`es, and `@main case class`es can
be nested arbitrarily deeply.

## Option or Sequence parameters

`@main` method parameters can be `Option[T]` or `Seq[T]` types, representing
optional parameters without defaults or repeatable parameters

```scala
package testoptseq
import mainargs.{main, arg, ParserForMethods, ArgReader}

object Main{
  @main
  def runOpt(opt: Option[Int]) = println(opt)

  @main
  def runSeq(seq: Seq[Int]) = println(seq)

  @main
  def runVec(seq: Vector[Int]) = println(seq)

  def main(args: Array[String]): Unit = ParserForMethods(this).runOrExit(args)
}
```
```bash
$ ./mill testoptseq runOpt
None

$ ./mill testoptseq runOpt --opt 123
Some(123)

$ ./mill testoptseq runSeq --seq 123 --seq 456 --seq 789
List(123, 456, 789)
```

## Annotations

The library's annotations and methods support the following parameters to
customize your usage:

### @main

- `name: String`: lets you specify the top-level name of `@main` method you are
  defining. If multiple `@main` methods are provided, this name controls the
  sub-command name in the CLI

- `doc: String`: a documentation string used to provide additional information
  about the command. Normally printed below the command name in the help message

### @arg

- `name: String`: lets you specify the long name of a CLI parameter, e.g.
  `--foo`. Defaults to the name of the function parameter if not given

- `short: Char`: lets you specify the short name of a CLI parameter, e.g. `-f`.
  If not given, theargument can only be provided via its long name

- `doc: String`: a documentation string used to provide additional information
  about the command

- `flag: Boolean`: turns a boolean argument into a flag, such that it can be
  provided via `--foo` rather than `--foo true`, with the absence of the flag
  being interpreted as `false`.

## Customization

Apart from taking the name of the main `object` or config `case class`,
`ParserForMethods` and `ParserForClass` both have methods that support a number
of useful configuration values:

- `allowPositional: Boolean`: allows you to pass CLI arguments "positionally"
  without the `--name` of the parameter being provided, e.g. `./mill testhello
  -f hello --my-num 3 --bool` could be called via `./mill testhello hello 3
  --bool`. Defaults to `false`

- `allowRepeats: Boolean`: allows you to pass in a flag multiple times, and
  using the last provided value rather than raising an error. Defaults to
  `false`

- `totalWidth: Int`: how wide to re-format the `doc` strings to when printing
  the help text. Defaults to `95`

- `printHelpOnExit: Boolean`: whether or not to print the full help text when
  argument parsing fails. This can be convenient, but potentially very verbose
  if the list of arguments is long. Defaults to `true`

- `docsOnNewLine: Boolean`: whether to print argument doc-strings on a new line
  below the name of the argument; this may make things easier to read, but at a
  cost of taking up much more vertical space. Defaults to `false`

## Custom Argument Parsers

If you want to parse arguments into types that are not provided by the library,
you can do so by defining an implicit `ArgReader[T]` for that type:

```scala
package testcustom
import mainargs.{main, arg, ParserForMethods, ArgReader}

object Main{
  implicit object PathRead extends ArgReader[os.Path](
    "path",
    strs => Right(os.Path(strs.head, os.pwd))
  )
  @main
  def run(from: os.Path, to: os.Path) = {
    println("from: " + from)
    println("to:   " + to)
  }

  def main(args: Array[String]): Unit = ParserForMethods(this).runOrExit(args)
}
```
```bash
$ ./mill testcustom --from mainargs --to out
from: /Users/lihaoyi/Github/mainargs/mainargs
to:   /Users/lihaoyi/Github/mainargs/out
```

In this example, we define an implicit `PathRead` to teach MainArgs how to parse
`os.Path`s from the [OS-Lib](https://github.com/lihaoyi/os-lib) library.
`ArgReader` requires the following fields:

```scala
class ArgReader[T](val shortName: String, // what to print in <...> in the help text
                   val read: Seq[String] => Either[String, T],
                   val alwaysRepeatable: Boolean = false, // used to allow Seq[T]-like parsers
                   val allowEmpty: Boolean = false) // used to allow Option[T]-like parsers
```

Note that `read` takes all tokens that were passed to a particular parameter.
Normally this is a `Seq` of length `1`, but if `allowEmpty` is `true` it could
be an empty `Seq`, and if `alwaysRepeatable` is `true` then it could be
arbitrarily long.

The `allowRepeats` parameter can also result in multiple tokens being passed to
your `ArgReader`; for `ArgReader`s that do not expect that, the convention is to
simply pick the last token in the list. There is no need to raise an error on
duplicates, as you can simply disable `allowRepeats` if you want the parser to
raise an error when a parameter is provided more than once.


