---
author: "Alfero Chingono"
title: "Dotnet Spawn Global Tool"
date: 2022-04-10T14:31:58Z
draft: false
description: "Introducing dotnet-spawn code generation global tool"
slug: dotnet-spawn-global-tool
tags: [
"dotnet",
"code-generation",
"roslyn"
]
categories: [
"development"
]
image: "cover.jpg"
---
Back in 2013, I created a nice little sample project demonstrating code generation among other things. [Scott Hanselman](https://twitter.com/shanselman) had this to say about it:

<!--{ { < tweet user="shanselman" id="288902839005220865" > } }-->

> Really col http://t.co/dzTs3fnv "Northwind" demo app by @achingono on GitHub. Clever use of T4 and more. Starred! https://t.co/1GTE055f
>
> — Scott Hanselman (@shanselman) January 9, 2013

I really enjoyed code generation with T4 in Visual Studio that I used it in all my projects. Four years later, I asked the following on Twitter:

{{< tweet user="achingono" id="936598758682451969" >}}

[Glenn von Breadmeister](https://twitter.com/condrong) was kind enough to respond with:

{{< tweet user="condrong" id="936636161933574144" >}}

And [Andrew Stanton-Nurse](https://twitter.com/anurse) took it further by saying:

{{< tweet user="anurse" id="938069127700156416" >}}

I somewhat forgot about these conversations until recently when I started building another sample project and needed to generate some code for rapid development. Looking around, I couldn't find something that suited my needs and besides, I had a lingering question: "Is razor really that bad at code generation?". I figured maybe it's worth a try.

So, I created a dotnet global tool for generating project files.

[![Nuget](https://img.shields.io/nuget/v/dotnet-spawn.svg)](https://www.nuget.org/packages/dotnet-spawn)

[dotnet-spawn](https://github.com/achingono/dotnet-spawn/) is a roslyn-based code generator for `dotnet` that adds files to a project or solution. Code generation parameters can be supplied inline or from a [response file](https://docs.microsoft.com/en-us/dotnet/standard/commandline/syntax#response-files). At this time [dotnet-spawn](https://github.com/achingono/dotnet-spawn/) is able to create a single file or multiple files of any extension based on a Razor file template.

### How To Use

Invoking the `dotnet spawn` command will generate project files based on the supplied parameters or [response file](https://docs.microsoft.com/en-us/dotnet/standard/commandline/syntax#response-files). You can control how verbose the output will be by using the `--verbosity` option.

### Command Options

- `--project`: The path to the project or solution file to analyze with Roslyn
- `--template`: The path to the template file used for code generation
- `--output`: The path to a file/folder. If the tool is generating a single file, then the path should be a file, otherwise, it should be a folder.
- `--namespace`: The namespace used for generated files, if applicable. This parameter is available in the Razor template and can be customized further.
- `--generator`: One of `SingleFile` or `MultipleFile`. As the names suggest, the `SingleFile` generator creates a single file and the `MultipleFile` generator creates multiple files.
- `--match`: The lambda expression representing the Roslyn symbols that should be used for code generation.
- `--pattern`: The lambda expression used by the `MultipleFile` generator to generate file names.
- `--verbosity`: Set the verbosity level. Allowed values are `Debug`, `Info`, and `Quiet`.

### The Help Option

This tool provides an option to display a brief description of the available commands, options, and arguments. `System.CommandLine` automatically generates help output. For example:

```console
dotnet spawn --help
```

produces the following output:

```console
Description:
  Roslyn based code generator.

Usage:
  dotnet-spawn [options]

Options:
  -p, --project <project> (REQUIRED)      Project file, Solution file or directory
  -t, --template <template> (REQUIRED)    Specify the template used for the generated code file(s)
  -g, --generator <generator> (REQUIRED)  Specify the generator used for the generated code file(s)
  -n, --namespace <namespace> (REQUIRED)  Specify the namespace for the generated code file(s)
  -m, --match <match>                     Specify the lambda expression used for the nanes of the generated code file(s)
  -p, --pattern <pattern>                 Specify the lambda expression used for the nanes of the generated code file(s)
  -o, --output <output> (REQUIRED)        Specify the file or folder used for the generated code file(s) [default: .]
  -v, --verbosity <Debug|Info|Quiet>      Output verbosity [default: Info]
  -f, --force                             Force execution of command
  --version                               Show version information
  -?, -h, --help                          Show help and usage information

```

### Response File

A _response file_ is a file that contains a set of tokens for a command-line app. Response files are a feature of `System.CommandLine` that is useful in two scenarios:

- To invoke a command-line app by specifying input that is longer than the character limit of the terminal.
To invoke the same command repeatedly without retyping the whole line.
- To use a response file, enter the file name prefixed by an `@` sign wherever in the line you want to insert commands, options, and arguments. The following lines are equivalent:

```console
dotnet spawn @controllers.rsp
```

Contents of _controllers.rsp_:

```
--project
./src/Sample.sln
--template
./src/Sample/Templates/Controller.cshtml
--output
./src/Sample/Controllers
--namespace
Sample.Controllers
--generator
MultipleFile
--match
symbol => symbol.IsReferenceType && !(symbol.IsAbstract || symbol.IsNamespace || symbol.IsVirtual)
--pattern
model => $"{model.Name}.cs"
--verbosity
Info
```

### How To Install

You can install the latest build of the tool using the following command.

```console
dotnet tool install -g dotnet-spawn 
```

### How To Uninstall

You can uninstall the tool using the following command.

```console
dotnet tool uninstall -g dotnet-spawn
```

