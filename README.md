## Literate jq+shell Programming with `jqmd`

`jqmd` is a tool for writing well-documented, complex manipulations of YAML or JSON data structures using bash scripting and `jq`.  It allows you to mix both kinds of code -- plus snippets of YAML or JSON data! -- within one or more markdown documents, making it easier to write scripts that do complex things like generate `docker-compose` configurations or manipulate serialized Wordpress options.

`jqmd` is implemented as an extension of [`mdsh`](https://github.com/bashup/mdsh), which means you can extend it to process additional kinds of code blocks by defining functions inside your `mdsh` blocks.  But you do not need to install mdsh, and you can use `jqmd --compile` to make distributable scripts that don't require jqmd *or* mdsh.

**Contents**

<!-- toc -->

- [Installation](#installation)
- [Usage](#usage)
  * [Data Merging](#data-merging)
- [Programming Models](#programming-models)
  * [Filters](#filters)
  * [Scripts](#scripts)
  * [Extensions](#extensions)
- [Available Functions](#available-functions)
  * [Adding jq Code and Data](#adding-jq-code-and-data)
  * [Adding jq Options and Arguments](#adding-jq-options-and-arguments)
  * [Controlling jq Execution](#controlling-jq-execution)
  * [Command-line Arguments](#command-line-arguments)
- [Supporting Additional Languages](#supporting-additional-languages)
  * [Postprocessing Code or Data Blocks](#postprocessing-code-or-data-blocks)
  * [Advanced Compilation Techniques](#advanced-compilation-techniques)

<!-- tocstop -->

### Installation

If you have [`basher`](https://github.com/basherpm/basher) on your system, you can install jqmd with `basher install bashup/jqmd`; otherwise, just download the [jqmd executable](bin/jqmd), `chmod +x` it, and put it in a directory on your `PATH`.

### Usage

Running `jqmd some-document.md args...` will read and interpret unindented, triple-backquote fenced code blocks from `some-document.md`, according to the language listed on the block:

* `shell` -- interpreted as bash code, executed immediately.  Shell blocks can invoke various jqmd functions as described later in this document.

* `jq` -- jq code, which is added to a jq filter pipeline for execution at the end of the file, or to be run explicitly with the `RUN_JQ` function.

* `jq defs` -- jq function definitions, which are accumulated over the course of the program run, and included at the start of any executed filter pipelines

* `jq imports` -- jq module includes or imports, which are accumulated over the course of the program run, and included at the start of any executed filter pipelines (before the current set of `jq defs`).

* `yaml`, `json` -- YAML data or JSON expressions, which are added to the jq filter pipeline as `jqmd::data(data)`.  (Which turns the given data into a jq filter to modify an existing data structure; see [Data Merging](#data-merging), below for more details).  Data blocks can also be tagged as "named constants": a code block starting with e.g. `````yaml !const foo`` will have its contents defined as a zero-argument jq function named `foo`.

  That is, the following two code blocks do the exact same thing:

  ~~~markdown
  ​```jq defs
  def pi: jqmd::data(3.14159);
  ​```
  ​```json !const pi
  3.14159
  ​```
  ~~~

  (Note: YAML data can only be processed if there is a `yaml2json` executable on `PATH`, the system `python` interpreter has PyYAML installed, or [yaml2json.php](https://packagist.org/packages/dirtsimple/yaml2json) is installed; otherwise an error will occur.  (For best performance, we recommend installing a tool like this [yaml2json written in Go](https://github.com/bronze1man/yaml2json), as its process startup time alone is considerably smaller than that of Python or PHP.)

  Both YAML and JSON blocks can contain **jq string interpolation expressions**, denoted by ``\( )``.  For example, a JSON block containing ``{ "foo": "\(env.BAR)"}`` will cause jq to insert the contents of the environment variable `BAR` into the data structure at the appropriate point.  (Note that this means that if you have a backslash before a `(` in your YAML blocks and you *don't* want it to be treated as interpolation, you will need to add an extra backslash in front of it.)

  (In addition, `json` blocks do not have to be valid JSON: they can actually contain arbitrary jq expressions.  The only real difference between a `json` block and a `jq` block is that it's automatically wrapped in a call to `jqmd::data()`.)

(As with `mdsh`, you can extend the above list by defining appropriate hook functions in `mdsh` blocks; see the section below on "Supporting Additional Languages" for more info.)

Once all blocks have been executed or added to the filter pipeline, jq is run on standard input with the built-up filter pipeline, if any.  (If the filtering pipeline is empty, jq is not run.)  Filter pipeline elements are automatically separated with `|`,  so you should not include a `|` at the beginning or end of your `jq` blocks or `FILTER` code.

As with `mdsh`, you can optionally make a markdown file directly executable by giving it a shebang line such as `#!/usr/bin/env jqmd`, or use a [shelldown header](https://github.com/bashup/mdsh#making-sourceable-scripts-and-handling-0) to make it executable, sourceable, and pretty.  :)  A sample shelldown header for jqmd might look like:

~~~markdown
#!/usr/bin/env bash
: '
<!-- ex: set ft=markdown : '; eval "$(jqmd --eval "$BASH_SOURCE")" # -->

# My Awesome Script

...markdown and code start here...
~~~

Also as with `mdsh`, you can run `jqmd --compile` to output a bash version of your script, with no external dependencies (other than jq and maybe  `yaml2json` or PyYAML).  `jqmd --compile` and `jqmd --eval` both inject the necessary jqmd runtime functions into the script so that it will work on systems without jqmd installed.  (Note that unless your script uses the `YAML` or `yaml2json` functions at *runtime*, your script's users will not need it installed.)

(If you'd like more information on compiling, sourcing, and shelldown headers, feel free to have a look at the [mdsh docs](https://github.com/bashup/mdsh)!)

#### Data Merging

In a jqmd program, one is often incrementally defining some sort of data structure (such as, e.g. a docker-compose project specification, or a set of Wordpress options).  While jq expressions can be used directly to manipulate such a data structure, a more intuitive way to express such data structures is as a series of JSON or YAML blocks that are combined in some way.  For this reason, jqmd defines an intuitive data structure merging function to apply such data blocks to an existing data structure.  This merging function is exposed to jqmd programs as  `jqmd::data($data)`, and a call to it is wrapped around JSON and YAML blocks by default.  The merge algorithm is as follows:

* If `.` is an array, add `$data` to it (concatenating if `$data` is also an array, otherwise appending)
* If `.` and `$data` are both objects, recursively merge their values using this same algorithm
* In all other cases, return `$data`

For most programs, this algorithm is sufficient to do most incremental data structure creation.  If you have different needs, however, you can use the `@data` directive in a `mdsh` (or `shell mdsh`) block, to specify a different jq function that will wrap `JSON` and `YAML` data.  (For example `@data foo` will use `foo(...data)` instead of `jqmd::data(...data)`.)

### Programming Models

`jqmd` supports developing three types of programs: filters, scripts, and extensions.  The main differences are that:

* Filters typically run jq once, implicitly, at the end of the document, sending the output to stdout,
* Scripts explicitly run jq multiple times or not at all, and
* Extensions are shell scripts written using `jqmd` functions to create different markdown processing and/or jq support tools.

#### Filters
Filters are programs that build up a single giant jq pipeline, and then act as a filter, typically taking JSON input from stdin and sending the result to stdout.  If your markdown document defines at least one filter, and doesn't use `RUN_JQ` or `CLEAR_FILTERS` to reset the pipeline, it's a filter.  `jqmd` will automatically run `jq` to do the filtering from stdin to stdout, after the *entire markdown document* has been processed.  If you don't want jq to read from stdin, you can use `JQ_OPTS -n` within your script to start the filter pipeline without any file input.  (Similarly, you can use `JQ_OPTS -- somefile` to force jq to read input from a specific file instead of stdin.)

#### Scripts

If your program isn't a filter, it's probably a script.  Scripts can run jq with shared imports, functions, and arguments, using the `RUN_JQ` function.  (They must not add anything to the filter pipeline after the last `RUN_JQ` or `CLEAR_FILTERS` call, though, or `jqmd` will think the program's a filter!)

You'll generally use this approach if your script needs to run jq multiple times with different inputs and filters.  Each time a script uses the `CLEAR_FILTERS` or `RUN_JQ`  functions, the filter pipeline is reset to empty and can then be built up again to run different operations.

(Note: unlike the filter pipeline, jq options, arguments, imports, and defintions are *cumulative*.  They can only be added to as the program executes, and cannot be reset.  Thus, they are shared across all invocations of `RUN_JQ`.  So anything specific to a given run of jq should be specified as a filter, or passed as an explicit command-line argument to `RUN_JQ`.)

#### Extensions

`jqmd` itself can be extended by other shell scripts, to make more-specialized tools or custom interpreters.  Sourcing `jqmd` from a bash script will define all its functions, but not actually run a program.  In this way, you can use all of the available functions described below (plus any of `mdsh`'s underlying API) in a shell script, rather than a markdown file.  (You can also use or redefine jqmd and mdsh's internal functions, but those not documented here or in the mdsh documentation are subject to change without notice!)

If you are sourcing `jqmd` (whether it's to write an extension or reuse its functions), you should also read  the  [mdsh docs](https://github.com/bashup/mdsh), since jqmd is an extension of mdsh.

### Available Functions

Within `shell` blocks, many functions are available for your use.  When passing `jq` code to them, it's best to use single quotes to avoid unwanted interpretation of $ variables or other quoting issues, e.g.:

```shell
DEFINE '
def recursive_add($other): . as $original |
    reduce paths(type=="array") as $path (
        (. // {}) * $other; setpath( $path; ($original | getpath($path)) + ($other | getpath($path)) )
    );
'
DEFINE 'def jqmd::data($arg): recursive_add($arg);'
```

#### Adding jq Code and Data

* `IMPORTS` *arg* -- add the given jq `import` or `include` statements to a block that will appear at the very beginning of the jq "program".  (Each statement must be terminated with `;`, as is standard for jq.)  Imports are accumulated in the order they are processed, but *all* imports active as of a given jq run will be placed at the beginning of the overall program, as required by jq syntax.

  (This function is the programmatic equivalent of including a `jq imports` code block at the current point of execution.)

* `DEFINE` *arg* -- add the given jq `def` statements to a block that will appear after the `IMPORTS`, but *before* any filters.  (Each statement must be terminated with `;`, as is standard for jq.)

  This function is the programmatic equivalent of including a `jq defs` code block at the current point of execution.

  Note: you do **not** have to define all your functions this way.  Functions can also be defined at the beginning of `FILTER` blocks or `jq`-tagged code blocks.  The main benefits of using `DEFINE` or `jq defs` blocks are that:

  - They can be done "out of order" within a document: you can use a function in a `jq` or `FILTER` block *before* its `DEFINE` block appears, as long as the `DEFINE` happens before jq is actually run.

  - In a script that runs jq more than once, `IMPORTS` and `DEFINE` blocks persist across jq runs, while `jq` and `FILTER` blocks reset after every `RUN_JQ`.

  - While a `jq` or `FILTER` block *has* to include a filter expression of some kind (even if it's just `.`), `DEFINE` blocks can **only** contain definitions and comments.

    (Well, technically, you *can* include filtering expressions in a `DEFINE` block, but it's not recommended, and you would then have to end the block with a `|` to get a syntactically-correct jq program.)

* `FILTER` *expr [args...]* -- add the given jq expression to the jq filter pipeline.  The expression is automatically prefixed with `|` if any filter expressions have already been added to the pipeline.  (This function is the programmatic equivalent of including a `jq` code block at the current point of execution.)

  If any arguments are supplied after *expr*, they are inserted wherever `%s` appears in it.  (So `FILTER "foo(%s; %s)" bar baz` will expand to something like `foo($JQMD_QA_1, $JQMD_QA_4)`, with `--arg JQMD_QA_1 bar --arg JQMD_QA_4 baz` added to the jq command line.  In this way, you can insert arbitrary strings into a jq expression, even if they contain characters that must be escaped in JSON.

  If you are using arguments, please note that you must 1) only put `%s` in parts of the expression where a jq *variable* can appear, 2) escape all other uses of `%` by doubling them (`%%`), and 3) make sure you have the same number of `%s`s in *expr* as you have additional arguments.  (None of these rules apply if you only supply *expr* with no *args*.)

  Every `jq`-tagged code block or `FILTER` argument **must** contain a jq expression.  Since jq expressions can begin with function definitions, this means that you can begin a filter with function definitions.  This can be useful for redefining `jqmd::data` or other functions at various points within your filter pipeline, or to define functions that will only be used for one `RUN_JQ` pipeline.

  Bear in mind, however, that because a filter block *must* contain a valid jq expression, you may need to terminate your filter with a `.` if it contains only functions.  For example, this bit of `jq` code is a valid filter, because it ends with a `.`:

  ```jq
  # Add as many functions as you like
  def f1($other): something;
  def f2: another(thing);

  # but finish with a '.' to create a no-op filtering expression
  .
  ```

  This "end function-only filters with a ." rule applies whether you're using `jq`-tagged code blocks or the `FILTER` function.

* `JSON`  *data [args...]* -- a shortcut for  `FILTER "jqmd::data("`*data*`")"` *args...*.  This function is the programmatic equivalent of including a `json` code block at the current point of execution, but it can also include interpolated args, as with `FILTER` (and the same rules for `%s` and escaping `%` apply if you supply any *args*).

* `YAML` *data* -- a shortcut for  `FILTER "jqmd::data("`*data-converted-to-json*`")"`.  This function is the programmatic equivalent of including a `yaml` code block at the current point of execution, and only works if there is a `yaml2json` converter on `PATH`, the system default `python` has PyYAML installed, or [yaml2json.php](https://packagist.org/packages/dirtsimple/yaml2json) is on the system `PATH`.)

* `@data` *jq-funcname* -- change the default data wrapper function from `jqmd::data` to *jq-funcname*.  If used in a `mdsh` or `shell mdsh` block, it changes this for `yaml` and `json` blocks; if used in a regular `shell` block, it changes the data wrapper used by the `YAML` and `JSON` functions.

* `yaml2json` -- a filter that takes YAML or JSON input, and produces JSON output.  The actual implementation is system-dependent, using either a `yam2json` command line tool, Python, or PHP, depending on what's available.  This can be used to convert data, validate it, or to remove jq expressions from untrusted input.

Notice that JSON and YAML blocks are always filtered through a `jqmd::data()` function, or the function specified by the last `@data` directive.  (Just remember that within each filter block, you can begin with function definitions but *must* end with an expression, even if it's only a `.`.)

Also note that data passed to the `JSON` and `YAML` functions *can contain jq interpolation expressions*, which means that you **must not pass untrusted data to them**.  You can validate and/or neutralize such data by piping it through the `yaml2json` function before calling `JSON` on it.

#### Adding jq Options and Arguments

* `JQ_OPTS` *opts...* -- add *opts* to the jq command line being built up.  Whenever jq is run (either explicitly using `RUN_JQ` or `CALL_JQ`, or implicitly at the end of the document), the given options will be part of the command line.
* `ARG` *name value* -- define a jq variable named `$`*name*, with the supplied string value.  (Shortcut for  `JQ_OPTS --arg name value`.)
* `ARGQUOTE` *value* -- like `ARG`, but instead of passing in an argument name, a unique argument name is generated, and returned in `$REPLY`.  The returned string will expand as *value* in any jq expressions.
* `ARGJSON` *name json-value* -- define a jq variable named `$`*name*, with the supplied JSON value.  (Shortcut for `JQ_OPTS --argjson name json`.)  This is especially useful for passing the output of other programs or data files as arguments to your jq code, e.g. `ARGJSON something "$(wp option get something --format=json)"`.


(Note: the added options will reset to empty again after `RUN_JQ`, `CALL_JQ`, or `CLEAR_FILTERS`.)

#### Controlling jq Execution

* `RUN_JQ` *args...* -- invoke `jq` with the current `JQ_OPTS` and given *args*.  If a "program" is given in `JQ_OPTS` (i.e., a non-option argument other than `--`), it's added to the filter pipeline, after any `IMPORTS` and `DEFINE` blocks established so far.  Any `-f` or `--fromfile` options are similarly added to the filter pipeline, and multiple such files are allowed.  (Unlike plain jq, which doesn't work properly with multiple `-f` options.)

  After jq is run, the filter pipeline is emptied with `CLEAR_FILTERS`

* `CALL_JQ` *args...* -- exactly like `RUN_JQ`, except that the output of `jq` is captured into `$REPLY`.  You should use this instead of shell substitution to capture jq's output.

* `CLEAR_FILTERS` -- reset the current filter pipeline and `JQ_OPTS` to empty.  This can be used at the end of a script to keep `jqmd` from running jq on stdin/stdout.

* `HAVE_FILTERS` -- succeeds if there is anything in the filter pipeline at the time of excution, fails otherwise. (i.e., you can use `if HAVE_FILTERS; then ...` to take action in a script based on the current filter state.

Note: piping into `RUN_JQ` or `CALL_JQ`, or invoking them in a subshell or shell substituion will *not* reset the current filter pipeline.  To capture jq's output, use `CALL_JQ` instead of shell substitution.  To pipe input into jq, pass it as a post-`--` argument to `RUN_JQ` or `CALL_JQ`, e.g.:

~~~sh
$ echo '"something"' | RUN_JQ .       # WRONG: CLEAR_FILTERS won't run
$ RUN_JQ . -- <(echo '"something"')   # RIGHT: use process substitution instead of piping

$ foo bar "$(RUN_JQ)"        # WRONG: CLEAR_FILTERS won't run
$ CALL_JQ; foo bar "$REPLY"  # RIGHT
~~~

#### Command-line Arguments

You can pass additional arguments to `jqmd`, after the path to the markdown file.  These additional arguments are available as `$1`, `$2`, etc. within any top-level `shell` code in the markdown file.

### Supporting Additional Languages

By default, `jqmd` only interprets markdown blocks tagged as `shell`, `jq`, `yaml`, `yml`, or `json`.  As with `mdsh`, however, you can define interpreters for other block types by defining `mdsh-lang-X` functions in`mdsh` blocks, via a wrapper script, or as exported functions in your bash environment.  (You can also override these functions to change jqmd's default interpretation of jq, YAML, or JSON blocks.)

When a code block of type `X` is encountered, the corresponding `mdsh-lang-X` function will be called with the contents of the block on the function's standard input.  (Note: the function name is **case sensitive**, so a block tagged `C` (uppercase) will not use a processor defined for `c` (lowercase) or vice-versa.)

If no corresponding language processor is defined when a block of type `X` is encountered, the contents of the block are appended to a bash array named `mdsh_raw_X`, where `X` is the language tag on the code block, with non-identifier characters changed to `_`.

So, if you have code blocks with a language tag of  `foo`, then `$mdsh_raw_foo` or `${mdsh_raw_foo[0]}` will return the contents of the first such block, `${mdsh_raw_foo[1]}` is the second, and so on.  Normal bash array rules apply, and you are free to modify the array in any way you like: mdsh merely adds new elements to the array as blocks are encountered.

#### Postprocessing Code or Data Blocks

Functions named `mdsh-after-X` will be called *after* a code block of  type `X` is processed.  The function does *not* receive the block contents, and is called whether or not a corresponding  `mdsh-lang-X` function exists.

If there is no `mdsh-lang-X`, the `mdsh-after-X` function can read the most recent block's contents from `${mdsh_raw_X[-1]}`.  (This can be useful when running language interpreters that can't execute code via stdin, or when you want the code to run with the main script's stdin.)  If you don't unset the array, it will keep growing as more blocks of that language are encountered.

Another trick you can do with postprocessing hooks, is to automatically run jq after each `jq` block.  For example:

> ~~~markdown
> ​```mdsh
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

(Note that `mdsh` post-processing hooks are **only** applied to blocks found in the current markdown file.  They do **not** run when `FILTER` ,  `YAML` , `JSON`, etc. are invoked programmatically!)

#### Advanced Compilation Techniques

`mdsh` actually allows for far more sophisticated code generation and metaprogramming than we've covered here: please consult its [docs](https://github.com/bashup/mdsh) for more details!



