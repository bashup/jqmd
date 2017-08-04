## Literate jq+shell Programming with `jqmd`

`jqmd` is a tool for writing well-documented, complex manipulations of YAML or JSON data structures using bash scripting and `jq`.  It allows you to mix both kinds of code -- plus snippets of YAML or JSON data! -- within one or more markdown documents, making it easier to write scripts that do complex things like generate `docker-compose` configurations or manipulate serialized Wordpress options.  (It can even read `.env` files and make the variables accessible from jq expressions!)

`jqmd` is implemented as an [`mdsh`](https://github.com/bashup/mdsh) extension, which means you can extend it to process additional kinds of code blocks by defining functions inside your `shell` blocks.  (It also means you need `mdsh` on your path!)

If you have [`basher`](https://github.com/basherpm/basher) on your system, it's the easiest way to install `jqmd`, as it will automatically install `mdsh` as part of the installation.  Just run:

```shell
basher install bashup/jqmd
```

If you don't have or want `basher`, though, you'll need to manually install `mdsh` first, then copy the `jqmd` script from this repository onto your path.

### Usage

Running `jqmd some-document.md args...` will read and interpret triple-quoted code blocks from `some-document.md`, according to the language listed on the block:

* `shell` -- interpreted as bash code, executed immediately.  Shell blocks can invoke various jqmd functions as described later in this document.  They can also `INCLUDE` other files, such as other markdown files, `.env` files, `.jq` files, `.yml` or `.json` files -- see the docs for the `INCLUDE` function near the end of this document.)
* `jq` -- jq code, which is added to a jq filter pipeline for execution at the end of the file, or to be run explicitly with the `RUN_JQ` function.
* `yaml`, `json` -- constant data, which is added to the jq filter pipeline as `jqmd_data(data)`; the effect of this depends on your definition of a `jqmd_data`  function as of the current point in the filter pipeline.  (Note: yaml data can only be processed if the system `python` interpreter has PyYAML installed; otherwise an error will occur.)

(As with `mdsh`, you can extend the above list by defining appropriate hook functions; see the section below on "Supporting Additional Languages" for more info.)

Once all blocks have been executed or added to the filter pipeline, jq is run on standard input with the built-up filter pipeline, if any.  (If the filtering pipeline is empty, jq is not run.)  Filter pipeline elements are automatically separated with `|`,  so you should not include a `|` at the beginning or end of your `jq` blocks or `FILTER` code.

As with `mdsh`, you can optionally make a markdown file executable by giving it a shebang line such as `#!/usr/bin/env jqmd`, or if you prefer not to make it a level one heading, you can put something like this on the first line instead:

```sh
``exec jqmd "$0" "$@"``
```

Most markdown processors will treat the above as a standalone paragraph containing a bit of code, while most shells will interpret it as a POSIX `sh` command to run `jqmd` on the file, passing along any extra arguments.  (Unlike a `#!`  line, this won't work with all shells, nor will the file be executable *without* the use of a shell, so use this trick at your own risk!)

### Programming Models

`jqmd` supports developing three types of programs: filters, scripts, and extensions.  The main differences are that:

* Filters typically run jq once, implicitly, at the end of the document, sending the output to stdout,
* Scripts explicitly run jq multiple times or not at all, and
* Extensions are shell scripts written using `jqmd` functions to create different markdown processing and/or jq support tools.

#### Filters
Filters are programs that build up a single giant jq pipeline, and then act as a filter, typically taking JSON input from stdin and sending the result to stdout.  If your markdown document defines at least one filter, and doesn't use `RUN_JQ` or `CLEAR_FILTERS` to reset the pipeline, it's a filter.  `jqmd` will automatically run `jq` to do the filtering from stdin to stdout, after the *entire markdown document* (and anything it `INCLUDE`s) have been processed.  If you don't want jq to read from stdin, you can use `JQ_OPTS -n` within your script to start the filter pipeline without any file input.  (Similarly, you can use `JQ_OPTS -- somefile` to force jq to read input from a specific file instead of stdin.)

#### **Scripts**

If your program isn't a filter, it's probably a script.  Scripts can run jq with shared imports, functions, and arguments, using the `RUN_JQ` function.  (They must not add anything to the filter pipeline after the last `RUN_JQ` or `CLEAR_FILTERS` call, though, or `jqmd` will think the program's a filter!)

You'll generally use this approach if your script needs to run jq multiple times with different inputs and filters.  Each time a script uses the `CLEAR_FILTERS` or `RUN_JQ`  functions, the filter pipeline is reset to empty and can then be built up again to run different operations.

(Note: unlike the filter pipeline, jq options, arguments, imports, and defintions are *cumulative*.  They can only be added to as the program executes, and cannot be reset.  Thus, they are shared across all invocations of `RUN_JQ`.  So anything specific to a given run of jq should be specified as a filter, or passed as an explicit command-line argument to `RUN_JQ`.)

#### Extensions

`jqmd` itself can be extended by other shell scripts, to make more-specialized tools or custom interpreters.  Sourcing `jqmd` from a bash script will define all its functions, but not actually run a program.  In this way, you can use all of the available functions described below (plus any of `mdsh`'s underlying API) in a shell script, rather than a markdown file.  (You can also use or redefine jqmd and mdsh's internal functions, but those not documented here or in the mdsh documentation are subject to change without notice!)

If you are sourcing `jqmd` (whether it's to write an extension or reuse its functions), you should also read  the  [mdsh docs](https://github.com/bashup/mdsh), so that you'll know how to deal with the required `EXIT` trap and bash options.

(Note: because of the limits of how `jq` syntax works, `jqmd` has to accumulate imports, definitions, and filters separately.  And because the maximum amount of data that can be stored in a variable or passed on a command-line varies between platforms (and is sometimes quite small), these accumulations are done in temporary files.  So a temporary directory is created on demand, and deleted on script completion by an `EXIT` trap.  Therefore, if you define an `EXIT` trap in a script that sources `jqmd`, be aware that the trap **must** invoke `mdsh-teardown` to ensure the temporary directory gets wiped.)

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
```

(Whether using arguments or heredocs, keep in mind that shell quoting and jq code don't mix well: single quotes are usually best!)

#### Adding jq Code and Data

These functions can all read input from their stdin, so you can use pipelines, redirection, or heredocs (e.g. `<<'EOF'`) to supply their input.  (Most also support passing a single explicit argument, so you can do e.g. `FILTER '.x'` *without* needing a heredoc or pipeline.)

* `IMPORTS` *arg-or-stdin* -- add the given jq `import` or `include` statements to a block that will appear at the very beginning of the jq "program".  (Each statement must be terminated with `;`, as is standard for jq.)  Imports are accumulated in the order they are processed, but *all* imports active as of a given jq run will be placed at the beginning of the overall program, as required by jq syntax.

* `DEFINE` *arg-or-stdin* -- add the given jq `def` statements to a block that will appear after the `IMPORTS`, but *before* any filters.  (Each statement must be terminated with `;`, as is standard for jq.)

  Note: you do **not** have to define all your functions this way.  Functions can also be defined at the beginning of `FILTER` blocks or `jq`-tagged code blocks.  The main benefits of using `DEFINE` blocks are that:

  - They can be done "out of order" within a document: you can use a function in a `jq` or `FILTER` block *before* its `DEFINE` block appears, as long as the `DEFINE` happens before jq is actually run.

  - In a script that runs jq more than once, `IMPORTS` and `DEFINE` blocks persist across jq runs, while `jq` and `FILTER` blocks reset after every `RUN_JQ`.

  - While a `jq` or `FILTER` block *has* to include a filter expression of some kind (even if it's just `.`), `DEFINE` blocks can **only** contain definitions and comments.

    (Well, technically, you *can* include filtering expressions in a `DEFINE` block, but it's not recommended, and you would then have to end the block with a `|` to get a syntactically-correct jq program.)

* `FILTER` *arg-or-stdin* -- add the given jq expression to the jq filter pipeline.  The expression is automatically prefixed with `|` if any filter expressions have already been added to the pipeline.  (This function is the programmatic equivalent of including a `jq` code block at the current point of execution.)

  Every `jq`-tagged code block or `FILTER` argument **must** contain a jq expression.  Since jq expressions can begin with function definitions, this means that you can begin a filter with function definitions.  This can be useful for redefining `jqmd_data` or other functions at various points within your pipeline, or to define functions that will only be used for one `RUN_JQ` pipeline.

  Bear in mind, however, that because a filter block *must* contain a valid jq expression, you may need to terminate your filter with a `.` if it contains only functions.  For example, this bit of `jq` code is a valid filter, because it ends with a `.`:

  ```jq
  # Add as many functions as you like
  def f1($other): something;
  def f2: another(thing);

  # but finish with a '.' to create a no-op filtering expression
  .
  ```

  This "end function-only filters with a ." rule applies whether you're using `jq`-tagged code blocks or the `FILTER` function.

* `JSON`  -- reads JSON data from stdin, wraps it in a call to `jqmd_data()`, and adds it to the filter pipeline.   (This function is the programmatic equivalent of including a `json` code block at the current point of execution.)

* `YAML` -- reads YAML data from stdin, converts it to JSON, then passes it to `JSON`.  (This function is the programmatic equivalent of including a `yaml` code block at the current point of execution, and only works if the system default `python` has PyYAML installed.)

Notice that data is always filtered through a `jqmd_data()` function, which your script must define.  You are not required to keep this function the same, however.  You can redefine it at various points in your pipeline if you need to handle it.  (Just remember that within each filter block, you can begin with function definitions but *must* end with an expression, even if it's only a `.`.)

#### Adding jq Options and Arguments

* `JQ_OPTS` *opts...* -- add *opts* to the jq command line being built up.  Whenever jq is run (either explicitly using `RUN_JQ`, or implicitly at the end of the document), the given options will be part of the command line.
* `ARG` *name value-or-stdin* -- define a jq variable named `$`*name*, with the supplied string value.  (Basically equivalent to `JQ_OPTS --arg name value`, except that the value can come from stdin.)
* `ARGJSON` *name json-or-stdin* -- define a jq variable named `$`*name*, with the supplied JSON value.  (Basically equivalent to `JQ_OPTS --argjson name json`, except that the data can come from stdin.)  This is especially useful for passing the output of other programs or data files as arguments to your jq code, e.g. `wp option get something --format=json | ARGJSON something`.


#### Controlling jq Execution

* `RUN_JQ` *args...* -- invoke `jq` with the current `JQ_OPTS` and given *args*.  If a "program" is given in `JQ_OPTS` (i.e., a non-option argument other than `--`), it's added to the filter pipeline, after any `IMPORTS` and `DEFINE` blocks established so far.  Any `-f` or `--fromfile` options are similarly added to the filter pipeline, and multiple such files are allowed.  (Unlike plain jq, which doesn't work properly with multiple `-f` options.)

  After jq is run, the filter pipeline is emptied with `CLEAR_FILTERS`

* `CLEAR_FILTERS` -- reset the current filter pipeline to empty.  This can be used at the end of a script to keep `jqmd` from running jq on stdin/stdout.

* `HAVE_FILTERS` -- succeeds if there is anything in the filter pipeline at the time of excution, fails otherwise. (i.e., you can use `if HAVE_FILTERS; then ...` to take action in a script based on the current filter state.

#### Command-line Arguments and Includes

You can pass additional arguments to `jqmd`, after the path to the markdown file.  These additional arguments are available as `$1`, `$2`, etc. within any top-level `shell` code in the markdown file.

For convenient access inside of shell functions, there is also a bash array, `$ARGV`, that contains the full command line given to jqmd, including the path to the markdown file itself.  (In other words, `${ARGV[0]}` contains the path to the markdown file, `${ARGV[1]}` is the first argument after it, and so on.)

The `INCLUDE` *filename args...* function executes another file as if it were literally included in the current file at the point of execution.  And within the included file, the positional arguments and `$ARGV`  array reflect the arguments given to `INCLUDE`.  If you want the included file to see the same arguments, use `INCLUDE filename "$@"` (in top-level code) or `INCLUDE filename "${ARGV[@]:1}"` (inside a function).

You aren't limited to including other markdown files, either.  The extension of the supplied  *filename* determines how the file's contents will be interpreted:

* Filenames ending in `.md`, `.mdown`, or `.markdown` are processed as markdown files, with code block interpretation as described at the start of this document.  Additional arguments passed to `INCLUDE` are available via `$1`, `$2`, etc. and `$ARGV` as described above.
* Filenames ending in `.env` are intepreted as shell scripts, and any variables they define will be **exported**, making them available for reading via jq's `env` function (jq 1.5 and up).  Additional arguments passed to `INCLUDE` are available via `$1`, `$2`, etc. and `$ARGV` as described above.
* Files ending in `.jq`, `.yaml`, `.yml`, or  `.json` are processed the same as an equivalent-language block in a markdown file, with appropriate wrapping and/or translation before being added to the current filter pipeline.  Additional arguments passed to `INCLUDE` are ignored, so there is no difference between using `INCLUDE` on these file types and simply doing `FILTER <file.jq`, `YAML <file.yml`, or `JSON <file.json`.  (Indeed, `INCLUDE` is actually *less* flexible, because a valid filename is required!  The main usefulness of INCLUDE for these file types is when you don't know in advance what extension the file will have.)

As with `mdsh`, you can define file extension handlers for extensions not listed above.  Handlers for a specific extension can be defined by creating bash functions named `mdsh-include-ext`, where `ext` is the file extension.  The longest-matching extension will be used, so `INCLUDE somefile.tar.gz` will call`mdsh-include-tar.gz` if it exists, falling back to `mdsh-include-gz` , and finally an error if no matching handler is found.

If the file doesn't exist, lacks an extension, or no matching handler is found for that extension, an error message is written to stderr and the `INCLUDE` command fails, causing the overall script to terminate.

### Supporting Additional Languages

By default, `jqmd` only interprets markdown blocks tagged as `shell`, `jq`, `yaml`, `yml`, or `json`.  As with `mdsh`, however, you can define interpreters for other block types by defining `mdsh-lang-X` functions in your `shell` blocks, a wrapper script, or as exported functions in your bash environment.  (You can also override these functions to change jqmd's default interpretation of jq, YAML, or JSON blocks.)

When a code block of type `X` is encountered, the corresponding `mdsh-lang-X` function will be called with the contents of the block on the function's standard input.  (Note: the function name is case sensitive, so a block tagged `C` (uppercase) will not use a processor defined for `c` (lowercase) or vice-versa.)

If no corresponding language processor is defined when a block of type `X` is encountered, the contents of the block are appended to `$MDSH_TMP/unprocessed.X`, where `X` is the language tag on the code block, and `$MDSH_TMP` is an automatically-created temporary directory.  (You can then use the accumulated contents of these files at the end of your script, or read/write/delete the files as needed.)

#### Postprocessing Code or Data Blocks

Functions named `mdsh-after-X` will be called *after* a code block of  type `X` is processed.  The function does *not* receive the block contents, and is called whether or not a corresponding  `mdsh-lang-X` function exists.  If there is no `mdsh-lang-X`, the `mdsh-after-X` function can read the block from `$MDSH_TMP/unprocessed.X`
and then delete the file.  (This can be useful when running language interpreters that can't execute code via stdin, or when you want the code to run with the main script's stdin.)  If you don't delete the file, it will keep growing as more blocks of that language are encountered.

Another trick you can do with postprocessing hooks, is to automatically run jq after each `jq` block.  For example:

> ~~~markdown
> ​```shell
> mdsh-after-jq() { RUN_JQ -- somefile.json ; }
> ​```
>
> ​```jq
> .x.y
> ​```
>
> ​```jq
> .z
> ​```
> ~~~

The above script will run `jq` twice on `somefile.json` with two different filters, instead of once with the results of the first filter being passed through the second.  (Because each run of `RUN_JQ` resets the filter pipeline.)  As always, however,`IMPORTS`, `DEFINES`, `JQ_OPTS`, `ARG` defs, etc. are retained from one jq run to the next.

(Note that `mdsh` post-processing hooks are **only** applied to blocks found in the current markdown file or any `INCLUDE`d markdown files.  They do **not** run when `FILTER` ,  `YAML` , `JSON`, etc. are invoked programmatically, or upon`INCLUDE` of non-markdown files!)

## LICENSE

`jqmd` is copyright 2017 PJ Eby, and MIT-licensed as follows:

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT OWNERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

