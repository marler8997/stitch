An exploration on a scripting language.  The basic syntax is:

```
program args...
program args...
```


* how to interpret the program string
    - if it contains slashes, it is a file path
    - otherwise, it is a program in one of the PATH directories (NOTE: need builtin to resolve this, like $findprog)
* if a command argument looks like one argument, it should be

Special characters:

Char | Description
-----|-------------
`$`  | starts an expression, escape with `$$`
`(`  | start a command substitution, escape with `$(`
`)`  | ends a command substitution, escape with `$)`

> NOTE: instead of using `#` as a comment, the sequence `$#` can be used.  This reduces the number of special characters for which different modes would need to accomodate.  One thing to note is `$#` is processed earlier than other kinds of `$` escaping, that may be confusing.  Not sure if this is worth it or not.

> NOTE: Say a line starts with `$foo`, if foo is a string, then it's expanded and it's interpreted as a program. If it's function (like $echo), then it's interpreted as that builtin function.

> IDEA: maybe a script or a line should be able to change their special symbol from `$` to something else?  For example, you could have something like this `$->! !echo your balance is $1.25`.

## Sample

```
$# this is a comment

$echo hello

$# varibles
$set name Fred
$echo Hello $name

$# arrays
$setarray names args Fred Lisa Joey
$echo Hello $expand.names

$# command substitution
$set arch ($oneline uname -m)
```

> NOTE: $echolines is just like $echo except it prints a newline after each argument
```sh
> $echolines arg1 arg2 arg3
arg1
arg2
arg3

> $echolines arg 1 arg 2 arg 3
arg
1
arg
2
arg
3

> $echolines arg$space$1 arg$space$2 arg$space$3
arg 1
arg 2
arg 3

$# NOTE: maybe I could make space be something like $_ or $-, whatever I pick, note that no other symbol could start with it
$#       $- might be better because some would probably expect $_foo to work
> $echolines arg$-1 arg$-2 arg$-3
> $echolines arg$_1 arg$_2 arg$_3
> $echolines arg$,1 arg$,2 arg$,3

$# NOTE: I can use command substituion to include spaces in a string like this
> $echolines ($echo arg 1) ($echo arg 2) ($echo arg 3)
arg 1
arg 2
arg 3


$# NOTE: even with $echo, there may be cases where we want to include `$` characters without escaping them.  In this case, maybe I should include a HERDOC style?  (an example of this is calling awk)  I could support something a number of delimiters such as
> $echolines $(arg 1) $[arg 2] $"arg 3" $'arg 4' $<arg 5> ${arg 6} $|arg 7|

> awk $"{print $1 $2}"

$# NOTE: I suppose these modes would disable `$` expansion?  Or maybe we want some modes to enable it?  Something to ponder on.
$# Not sure if I should support this many styles?  I will probably also want to have some sort of multiline
$# style of HERDOC, this one can take more characters.

$echo $delimited EOF
This is a raw multi-line
string that does not end
until it sees...
EOF
```

> NOTE: I don't really like using "$ " to escape a space, because then the argument doesn't look like one argument.  This could also make the implementation a bit more complicated because the parser wouldn't be able to split arguments using whitespace before expansion.
> TODO: should I support quoted arguments?  I'll leave them out for now since they aren't necessary for scripts to be funcationally complete.

$# Variables

```
$#
$# $set [SCOPE.]VARNAME VALUE
$#
$set msg Hello, my name is Fred
$echo $msg
$# prints "Hello, my name is Fred"

$echo $g.msg
$# still prints "Hello, my name is Fred", "g" is a special scope through which user variables can be accessed.  This should be used for variables that may conflict with predefined variables (i.e. $g.echo instead of $echo). (Note: `$g.g` is not the same as `$g`, it would be a user script variable named `g`).

$set cpu_count ($oneline nproc)

$echo You have $cpu_count cpus
$echo Expand cpu count with a suffix is $cpu_count$cpus
```



Builtin variables

Name   | C Equivalent
-------|---------
space  | The string " "
lf     | The string "\n"
cr     | The string "\r"
crlf   | The string "\r\n"
newline| Either "\n" or "\r\n" depending on platform
tab    | The string "\t"
g      | The script's global scope.
env    | Environment variable scope.
expand | Scope to expand arrays through.

> NOTE: maybe support $XX$ to insert arbitrary hex characters?  So a space could be $20$?

## Arrays

Arrays are important because they provide a way to represent one or more strings without requiring them to be delimited.  Not having an array type and requiring a delimiter instead is the source of many pitfalls with other scripting languages.

```
$#
$# $setarray [SCOPE.]VARNAME splitlines VALUE
$# $setarray [SCOPE.]VARNAME args ARGS...
$#
$# TODO: splitwhitespace?
$#
$# Example:
$setarray mounts splitlines (cat /proc/mounts)

$# now the "mounts" variable is an array of all the mounts

$# How to use the array in a command?
$args-to-lines $expand.mounts

$# Not using $expand.VAR will cause an error
$args-to-lines $mounts
$# Error: mounts is an array, you must expand it with $expand.mounts
```

Note that arrays cannot be used within another string.  They must be expanded on their own.  The reason for using the `expand` scope to expand arrays, is so that it's immediately apparent that an array is being expanded into 0 or more arguments.

So we have 2 types, strings and arrays of strings. Is that all we need?
We also have scopes, but so far string/arrays/scopes haven't had any intermixing in the grammar.

### Environment variables:

Environment variables are accessed in the `env` scope.

```
$env.VAR

$set env.VAR VALUE
```

## Command Substitution

```
$# run command and return it's output as a string
(PROG ARGS...)

$set myfile_content (cat myfile)

$# you can use the $oneline builtin to enforce that the command only prints one line and it will also trim the trailing newline
$set lsfile ($oneline which ls)

$# TODO: maybe have a $firstline/$lastline?

$# Example
make -j(nproc)
```

# Examples

### Hello World

```
$echo Hello World
```

### Read input and print it

```
$set input (read)
$echo You entered: $input
```

### Random

```
$set pwd ($oneline pwd)
$set arch ($oneline uname -m)
$set target $arch$-linux-musl
$set prefix $env.HOME/$target
$set jobs -j(nproc)

wget example.come/$target$.tar.xz
tar xf $target$.tar.xz

$set env.PATH $pwd/$target/bin:$env.PATH
```

## Builtin Functions/Programs

#### $oneline PROG ARGS...

Runs the given program, enforces it only outputs one line and trims the trailing newline if it exists.

# Grammar

What I've got so far
```
Script ::= Command*

Command ::= Arg*

Arg ::= (Char | '$' DollarExpr)*

DollarExpr ::= '(' Command ')' | [a-zA-Z_.]* '$'?
```

# Idea

How to handle return code and stdout/stderr?  I could have a CommandResult type of object.
```
$#
$# $captureouts PROG ARGS...
$# $captureall     PROG ARGS...
$#
$# the $capture* builtins will run the program/args that follow, and instead of just returning stdout, it will return
$# an object.  $capturestreams will capture stdout and stderr via the "out" and "err" fields.  $captureall will also capture
$ the exit code in the field `code`, note that this overrides the default behavior of exiting on a non-zero exit code.

$set some_program_result ($captureouts some-program)
$echo the stdout of some_program_result is: $some_program_result.out
$echo the stderr of some_program_result is: $some_program_result.err

$set another_program_result ($captureouts another-program)
$echo the stdout of another_program_result is: $another_program_result.out
$echo the stderr of another_program_result is: $another_program_result.err
$echo the exit code of another_program_result is: $another_program_result.code
```

# Idea: piping

Piping is a pretty useful feature.  You could emulate piping by setting output to variables, however, piping directly from one process to another could remove extra copies of the output.

A running process has 2 objects, the stdout and stderr file stream.  Once the process has finished, it returns 3 objects, a stdout string, a stderr string and an exit code.

A script should be able to forward stdout and/or stderr from one process to the stdin of another.  Note that to forward stdout/stderr to a named file that can be read by another program, pipe files can be used.  This means the scripting language would need a way to redirect stdout and/or stderr to a named file as well.  Note that this could be done using a pipe operator that goes to a builtin command that forwards it to a file (means we don't need a redirect feature).

Syntax
```

foo $out bar
foo $err bar
foo $outerr bar

$# How to pipe stdout and stderr to different places?

$# the following won't work because $err could belong to the previous command
foo $out COMMAND... $err COMMAND...


$# to redirect the output of "foo" to a file "myfile", use $in2file
foo $out $in2file myfile

$# to forward out/err to different places, could do multiple lines like this
$set foo $stage foo args...
$# the $stage COMMAND... builtin will create create a process that has not been started yet
$# now $myprogram.out and $myprogram.err are open file handles
$attach $foo.out bar
$attach $foo.err baz
$run $foo
```
