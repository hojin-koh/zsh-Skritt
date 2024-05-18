# zsh-Skritt

A heavily-opinionated and colorful zsh scripting framework. This library is geared toward simple scripts that does only one thing per script.

## Dependencies

- zsh: obviously.
- GNU coreutils
- zstd: for compression of log files. (I did mention this is heavily-opinionated, right?)
- util-linux(optional): for printing help message with `column`
- pv(optional): only if you need the progess-bar functionality

## Installation

Just clone this repository down into some subdirectory in your project. I personally prefer to use [git-subrepo](https://github.com/ingydotnet/git-subrepo):

```zsh
git subrepo clone https://github.com/hojin-koh/zsh-Skritt deps/zsh-Skritt
```

Or you can also use submodule:

```zsh
git submodule add https://github.com/hojin-koh/zsh-Skritt deps/zsh-Skritt
```

## Usage

The basic usage of this framework looks like this, which basically does nothing:

```zsh
#!/usr/bin/env zsh
description="Test if a simple script can be run"

main() {
  echo The main script goes here
}

source "${0:a:h}/deps/zsh-Skritt/skritt" # where this library is stored
```

`description` variable is reserved for the description of the script, which is used in the auto-generated help message.

`main` function is the things to be actually run.

The output looks like this:

```
[I-0000.0] > Begin ./scriptname.zsh
[I-0000.0] 2021-09-20 14:09:59 (SHLVL=5)
The main script goes here
[I-0000.0] < End ./scriptname.zsh
```

The main added functionalities of this library are:
- Opinionated colorful messages
- Command-line parsing
- Need-to-run check
- Pre-/post-script hooks
- Temp directory management
- Logging

### Colorful messages

`debug` command puts text into the log file only, not to the screen. All other commands in this section will output both to the screen (in color) and to the log file (no color).

`info` command outputs bright white info messages with a timestamp, just like the example in the **Basics** section.

`warn` command outputs yellow warning text with a timestamp.

`err` command outputs red error message with a timestamp and exits. If there's second parameter after the actual message, it will be used as the return value when exiting.

`prompt` command asks user a question, and stores the reply in `$REPLY` variable.

`promptyn` command asks user a yes/no question, and stores the reply in `$REPLY` variable.

### Command-line

Typing `./scriptname.zsh --help` will display an auto-generated help message, showing available options. If wrong options are given at the command line, the help message will also be displayed along with the error message.

`opt` command declares an option, the usage is: `opt [-r] [-<group>] <opt-name> <default-value> <description>`.

`-r` means this option is mandatory, and it will be an error if this value is empty. If a required argument is empty, but some positional arguments are given, these positional arguments will be used to fill in the required arguments in order.

`-<group>` specifies the which group this option belongs to. It is mainly used in help messages to group options for easier reading. If not specified, the option will belong to a group with empty name, and will be listed on the top of the help message. You cannot specify "r" as the group name.

`<opt-name>` is the name of the option. This also decides the corresponding variable name. For example, if the option name is `cm-threshold`, then you can specify `--cm-threshold=0.5` or `cm-threshold=0.5` on the command line, and the value will be stored inside a variable named `cm_threshold`. The option-name to variable conversion is as follows (Yes, there might be conflicts if you use consecutive dashes and dots—don't do that):
  - A single dash is converted to one underscore `_`.
  - A single dot is converted to three understores `___`.

`<default-value>` is the default value.

`<description>` is the help string regarding this option. It will be automatically added to the help message.

`opt` commands must be used inside a function named `setupArgs()`, which should be defined in the script. An example:

```zsh
#!/usr/bin/env zsh
description="Example of argument parsing"

setupArgs() {
  opt -r opt-1 '' "First Option"
}

main() {
  info "opt-1 = $opt_1"
}

source "${0:a:h}/deps/zsh-Skritt/skritt" # where this library is stored
```

`./scriptname.zsh` will give an error, but `./scriptname.zsh --opt-1=5`, `./scriptname.zsh opt-1=5`, or `./scriptname.zsh 5` will assign 5 to `$opt_1`.

### Need-to-run Check

You can optionally implement a `check()` function to indicate whether this script really need to run. Returning 0 from this function means what it's designed to do is already done / in the desired state, and there's no need to run anything. Returning 1 from this function means it's not done yet, and there's need to run this script. If there's no such function, it's always assumed that there's need to run this script.

If there's no need to run the script, it will exit without actually running the main function, unless `--force` is specified on the command line.

When a special option `--check` is specified, the script will return the return value of `check()` (or return 1 if there no such function) without running anything. `--force` has no effect here.

### Pre-/Post-script Hooks

The overall flow of the script looks like this:

```
skrittLibraryInit()

setupArgs()

"preparse" hooks
option parsing
check required arguments
"postparse" hooks

check()
"prescript" hooks
main()
"exit" hooks
```

These mysterious hooks already contain several built-in functions, but users can add new things into them through the `addHook()` function. Usage: `addHook <hook-name> <function-name> [begin]`

`<hook-name>` is basically the quoted string in the above flow.

`<function-name>` is a zsh function.

If the optional third parameter is "begin", the function will be inserted at the beginning of the hook sequence, instead of at the end (the default behaviour).

### Temp Directory Management

`putTemp <variable-name>` generates a temp directory and store the directory path into the desinated variable. At the end of the script (specifically, inside the "exit" hook) the temp directory will be cleaned up automatically.

### Logging

If built-in option `--logfile` was specified, log text will be written into this file, with automatic log-rotate mechanisms. Note that logging is set up inside the "prescript" phase, so the only proper way to modify this value would be from inside the "postparse" phase.

By default, 3 old log files will be kept, which are named `logfile.1.zst`, `logfile.2.zst`, and `logfile.3.zst`. This value can be changed with `--logrotate` option.
