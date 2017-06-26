## Literate jq+shell Programming with `jqmd`

`jqmd` is a tool for writing well-documented, complex manipulations of YAML or JSON data structures using bash scripting and `jq`.  It allows you to mix both kinds of code -- plus snippets of YAML or JSON data! -- within one or more markdown documents, making it easier to write scripts to do complex things like generate `docker-compose` configurations or manipulate serialized Wordpress options.  (It can even read `.env` files and make the variables accessible from jq expressions!) 

### Usage

Running `jqmd some-document.md args...` will read and interpret triple-quoted code blocks from `some-document.md`, according to the language listed on the block:

* `shell` -- interpreted as bash code, executed immediately.  Shell blocks can invoke various jqmd functions as described later in this document.  They can also `INCLUDE` other files, such as other markdown files, `.env` files, `.jq` files, `.yml` or `.json` files -- see the docs for the `INCLUDE` function near the end of this document.)
* `jq` -- jq code, which is added to a jq filter pipeline for execution at the end of the file, or to be run explicitly with the `RUN_JQ` function.
* `yaml`, `json` -- constant data, which is added to the jq filter pipeline as `jqmd_data(data)`; the effect of this depends on your definition of a `jqmd_data`  function as of the current point in the filter pipeline.  (Note: yaml data can only be processed if the system `python` interpreter has PyYAML installed; otherwise an error will occur.)

Once all blocks have been executed or added to the filter pipeline, jq is run on standard input with the built-up filter pipeline, if any.  (If the filtering pipeline is empty, jq is not run.)  Filter pipeline elements are automatically separated with `|`,  so you should not include a `|` at the beginning or end of your `jq` blocks or `FILTER` code.

### Installation and Setup

If you have [basher](https://github.com/basherpm/basher), you can install via `basher install pjeby/jqmd`.  Otherwise just clone this repo or copy the file and place it on your `PATH`.

Invoke `jqmd` *mdfile args...* to run a jqmd script or filter.  You can optionally make a markdown file executable by giving it a shebang line such as `#!/usr/bin/env jqmd`, or if you prefer not to make it a level one heading, you can put something like this on the first line instead:

```sh
``exec jqmd "$0" "$@"``
```

Most markdown processors will treat the above as a standalone paragraph containing a bit of code, while most shells will interpret it as a POSIX `sh` command to run `jqmd` on the file, passing along any extra arguments.  (Unlike a `#!`  line, this won't work with all shells, nor will the file be executable *without* the use of a shell, so use this trick at your own risk!)

### Programming Models

`jqmd` supports developing three types of program: filters, scripts, and extensions.

#### Filters
Filters are programs that build up a single giant jq pipeline, and then act as a filter, taking JSON input from stdin and sending the result to stdout.  If your markdown document defines at least one filter, and doesn't use `RUN_JQ` or `CLEAR_FILTERS`, it's a filter.  `jqmd` will automatically run `jq` to do the filtering from stdin to stdout, after the *entire markdown document* (and anything it `INCLUDE`s) have been processed.

#### **Scripts**

If your program isn't a filter, it's probably a script.  Scripts can run jq with shared imports, functions, and arguments, using the `RUN_JQ` function.  (They must not add anything to the filter pipeline after the last `RUN_JQ` or `CLEAR_FILTERS` call, though, or `jqmd` will think the program's a filter!)

You'll generally use this approach if your script needs to run jq multiple times with different inputs and filters.  Each time a script uses the `CLEAR_FILTERS` or `RUN_JQ`  functions, the filter pipeline is reset to empty and can then be built up again to run different operations.

(Note: unlike the filter pipeline, jq options, arguments, imports, and defintions are *cumulative*.  They can only be added to as the program executes, and cannot be reset.  Thus, they are shared across all invocations of `RUN_JQ`.  So anything specific to a given run of jq should be specified as a filter, or passed as an explicit command-line argument to `RUN_JQ`.)

#### Extensions

`jqmd` itself can be extended by other shell scripts, to make more-specialized tools or custom interpreters.  Sourcing `jqmd` from a bash script will define all its functions, but not actually run a program.  In this way, you can use all of the available functions described below in a shell script, rather than a markdown file.  (You can also use or redefine jqmd's internal functions, but those not documented here are subject to change without notice!)

If you are writing an extension (or reusing jqmd functions in another script), you should also read the section below on "How jqmd Works Internally".  (Spoiler alert: use bash's `-e` and `-u` options in your script, *or else*.)

### Available Functions

Within `shell` blocks, many functions are available for your use.  Many of them accept either a positional argument or input from stdin, so that you can use pipelines or heredocs.  e.g.:

```shell
# multi-line define w/heredoc
DEFINE <<'EOF'
def recursive_add($other): . as $original |
    reduce paths(type=="array") as $path (
        (. // {}) * $other; setpath( $path; ($original | getpath($path)) + ($other | getpath($path)) )
    );
EOF

# one-liner define w/argument
DEFINE 'def jqmd_data($arg): recursive_add($arg);'

echo $0; for f in "$@"; do echo "$f"; done
```

(Whether using arguments or heredocs, keep in mind that shell quoting and jq code don't mix well: single quotes are usually best!)

#### Adding jq Code and Data

These functions can all read input from stdin, so you can use pipelines or heredocs (e.g. `<<EOF`) to supply their input.  (Most also support passing a single explicit argument,, so you can do e.g. `FILTER '.x'` *without* needing a heredoc or pipeline.)

* `IMPORTS` *arg-or-stdin* -- add the given jq `import` or `include` statements to a block that will appear at the very beginning of the jq "program".  (Each statement must be terminated with `;`, as is standard for jq.)
* `DEFINE` *arg-or-stdin* -- add the given jq `def` statements to a block that will appear after the `IMPORTS`, but before any filters.  (Each statement must be terminated with `;`, as is standard for jq.)
* `FILTER` *arg-or-stdin* -- add the given jq expression to the jq filter pipeline.  The expression is automatically prefixed with `|` if any filter expressions have already been added to the pipeline.  (This function is the programmatic equivalent of including a `jq` code block at the current point of execution.)
* `JSON`  -- reads JSON data from stdin, wraps it in a call to `jqmd_data()`, and adds it to the filter pipeline.   (This function is the programmatic equivalent of including a `json` code block at the current point of execution.)
* `YAML` -- reads YAML data from stdin, converts it to JSON, then passes it to `JSON`.  (This function is the programmatic equivalent of including a `yaml` code block at the current point of execution, and only works if the system default `python` has PyYAML installed.)

#### Adding jq Options and Arguments

* `JQOPTS` *opts...* -- add *opts* to the jq command line being built up.  Whenever jq is run (either explicitly using `RUN_JQ`, or implicitly at the end of the document), the given options will be part of the command line.
* `ARG` *name value-or-stdin* -- define a jq variable named `$`*name*, with the supplied string value.  (Basically equivalent to `JQOPTS --arg name value`, except that the value can come from stdin.)
* `ARGJSON` *name json-or-stdin* -- define a jq variable named `$`*name*, with the supplied JSON value.  (Basically equivalent to `JQOPTS --argjson name json`, except that the data can come from stdin.)  This is especially useful for passing the output of other programs or data files as arguments to your jq code, e.g. `wp option get something --format=json | ARGJSON something`.


#### Controlling jq Execution

* `RUN_JQ` *args...* -- invoke `jq` with the current `JQOPTS` and given *args*.  If a "program" is given in `JQOPTS` (i.e., a non-option argument other than `--`), it's added to the filter pipeline, after any `IMPORTS` and `DEFINE` blocks established so far.  Any `-f` or `--fromfile` options are similarly added to the filter pipeline, and multiple such files are allowed.  (Unlike plain jq, which doesn't work properly with multiple `-f` options.)

  After jq is run, the filter pipeline is emptied with `CLEAR_FILTERS`

* `CLEAR_FILTERS` -- reset the current filter pipeline to empty.  This can be used at the end of a script to keep `jqmd` from running jq on stdin/stdout.

* `HAVE_FILTERS` -- succeeds if there is anything in the filter pipeline at the time of excution, fails otherwise. (i.e., you can use `if HAVE_FILTERS; then ...` to take action in a script based on the current filter state.

#### Command-line Arguments and Includes

You can pass additional arguments to `jqmd`, after the path to the markdown file.  These additional arguments are available as `$1`, `$2`, etc. within any top-level `shell` code in the markdown file.

For convenient access inside of shell functions, there is also a bash array, `$ARGV`, that contains the full command line given to jqmd, including the path to the markdown file itself.  (In other words, `${ARGV[0]}` contains the path to the markdown file, `${ARGV[1]}` is the first argument after it, and so on.)

The `INCLUDE` *filename args...* function executes another file as if it were literally included in the current file at the point of execution.  And within the included file, the positional arguments and `$ARGV`  array reflect the arguments given to `INCLUDE`.  If you want the included file to see the same arguments, use `INCLUDE filename "$@"` (in top-level code) or `INCLUDE filename "${ARGV[@]:1}"` (inside a function).

You aren't limited to including other markdown files, either.  The supplied  *filename* determines how the file's contents will be interpreted:

* Filenames ending in `.md`, `.mdown`, or `.markdown` are processed as markdown files, with code block interpretation as described at the start of this document.  Additional arguments passed to `INCLUDE` are available via `$1`, `$2`, etc. and `$ARGV` as described above.
* Filenames ending in `.env` are intepreted as shell scripts, and any variables they define will be **exported**, making them available for reading via jq's `env` function (jq 1.5 and up).  Additional arguments passed to `INCLUDE` are available via `$1`, `$2`, etc. and `$ARGV` as described above.
* Files ending in `.jq`, `.yaml`, `.yml`, or  `.json` are processed the same as an equivalent-language block in a markdown file, with appropriate wrapping and/or translation before being added to the current filter pipeline.  Additional arguments passed to `INCLUDE` are ignored, so there is no difference between using `INCLUDE` on these file types and simply doing `FILTER <file.jq`, `YAML <file.yml`, or `JSON <file.json`.  (Indeed, `INCLUDE` is actually *less* flexible, because a valid filename is required!  The main usefulness of INCLUDE for these file types is when you don't know in advance what extension the file will have.)

Any file extensions not listed above will result in an error, as `INCLUDE` cannot guess how such files should be interpreted.

#### Markdown Processing Functions

If you're writing an extension, or a markdown processing script of your own, you may find these functions useful also:

* `markdown-to-shell` *command languages...* -- read markdown from stdin, writing a sourceable shell script to stdout.  Triple-backquoted blocks tagged with any specified language are replaced with *command langname* and a heredoc for the the block body.  So for example, `markdown-to-shell process python perl` would output a `process python <<` command for each `python` code block, and and a `process perl <<` command for each `perl` code block.  It's up to you to supply a *command* that can take a language argument and process data from its stdin.  Typically, your full command using this would look something like `source <(markdown-to-shell cmd lang... <somefile.md)`, to parse `somefile.md` and execute the result
* `extract-markdown` *language_regex [firstline lastline]* -- read markdown from stdin, extracting and writing to stdout only the triple-backquoted code blocks whose language matches *language_regex*, optionally substituting *firstline* for the opening backquote line and *lastline* for the closing line.  (If omitted, the substituions are empty, causing those lines to be replaced with blank lines.)  `markdown-to-shell` actually works by building a language regex to pass to this function, along with some creative replacement patterns for *firstline* and *lastline* to implement the command and heredoc wrappers around each code block.

While `markdown-to-shell` is most likely to be useful for extensions or other tools, `extract-markdown` can sometimes be useful in scripts or filters, to extract code blocks from another file, or even *from the script itself*.   (For example, generating an `.ini` configuration file from `ini` blocks embedded in a script.)

### How jqmd Works Internally

Because of the limits on how `jq` syntax works, `jqmd` has to accumulate imports, definitions, and filters separately.  Because the maximum amount of data that can be stored in a variable or passed on a command-line varies between platforms (and is sometimes quite small), these accumulations are done in temporary files.  A temporary directory is created at the beginning of `jqmd` execution, and wiped by an `EXIT` trap.

Therefore, if you are extending jq or using its functions in another script, be aware that your `EXIT` trap, if any, must invoke `jqmd_teardown` to wipe the temporary directory.  Further, if you need to invoke any jqmd functions *after* teardown, you will need to call `jqmd_setup` again to create a new temporary directory.  (And then reset your `EXIT` trap, which will be overwritten by `jqmd_setup`!)

Extending scripts **must** use bash's `-e` and `-u` options.  These options prevent errors from cascading through a script.  For example, if a script calls a jqmd function after `jqmd_teardown` has run, the `-u` option will detect the error, and `-e` will keep the script from continuing.  But if you omit the `-u`, the functions will execute anyway, and try to write "temporary" files to the root directory of your machine!

(In general, it's a good idea to write bash scripts in ["unofficial strict mode"](http://redsymbol.net/articles/unofficial-bash-strict-mode/): i.e., with `set -euo pipefail`. With jqmd though, the `eu` bits are *mandatory*: don't bother opening issues about extensions that don't use them.)

