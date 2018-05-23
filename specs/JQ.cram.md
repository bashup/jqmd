## Running JQ and Managing Filters+Options

### Filter/Option State

Initially, there are no filters or options set:

````sh
    $  source "$TESTDIR/../jqmd.md"; set +e

    $ HAVE_FILTERS || echo nope
    nope

    $ echo "${JQ_OPTS[@]}"
    jq
````

But they can be added using `FILTER`, `JQ_OPTS`, `ARG`, and `ARGJSON`:

````sh
    $ FILTER '.'
    $ HAVE_FILTERS && echo yep
    yep
    $ ARG x "foo"
    $ ARGJSON y '{}'
    $ JQ_OPTS --slurpfile bar baz
    $ echo "${JQ_OPTS[@]}"
    jq --arg x foo --argjson y {} --slurpfile bar baz
````

And then reset with `CLEAR_FILTERS`:

````sh
    $ CLEAR_FILTERS
    $ HAVE_FILTERS || echo nope
    nope
    $ echo "${JQ_OPTS[@]}"
    jq
````

You can generate args with `ARGQUOTE`:

~~~shell
    $ ARGQUOTE 'foo"bar'; echo $REPLY
    $JQMD_QA_1
    $ ARGQUOTE spammity; echo $REPLY
    $JQMD_QA_4
~~~

and also via extra args to `FILTER` or `JSON`:

~~~shell
    $ FILTER 'foo(%s; %s)' bar 'baz"spam'; echo $jqmd_filters
    foo($JQMD_QA_7; $JQMD_QA_10)

    $ CLEAR_FILTERS
    $ JSON '{%s: %s}' foo bar; echo "${JQ_OPTS[@]}"
    jq --arg JQMD_QA_1 foo --arg JQMD_QA_4 bar
~~~

### Invoking JQ

The `JQ_CMD` function adds the supplied args to `$JQOPTS` and combines them with the current imports, defines, and filters to generate a command line in `${REPLY[@]}`.  It also resets the current filters and options.

````sh
    $ JQ() { JQ_CMD "$@" || return; printf ' %q' "${REPLY[@]}"; echo; }
    $ jqmd_defines=
    $ CLEAR_FILTERS

# No filters specified, default to '.'

    $ JQ
     jq .
    $ JQ -c
     jq -c .
    $ JQ --slurpfile bar baz -L foo -- spim spam
     jq --slurpfile bar baz -L foo . spim spam

# Filters given, use those:

    $ JQ .foo /bar  # data file(s) after filter
     jq .foo /bar
    $ JQ --argfile x y --indent 3 .bar
     jq --argfile x y --indent 3 .bar
    $ JQ -c .[].baz
     jq -c .\[\].baz

    $ FILTER '.spam'; JQ_OPTS -L foo
    $ JQ
     jq -L foo .spam
    $ JQ -c   # filter and opts are reset:
     jq -c .

# Imports and defines go before filters, newline-separated:

    $ FILTER '.spam'
    $ DEFINE 'def x: 1;'
    $ IMPORTS 'import foo;'
    $ JQ -c
     jq -c $'import foo;\ndef x: 1;\n.spam'
    $ jqmd_defines=
    $ jqmd_imports=

# Filters given by `-f` or `--fromfile` get read into the command line as part of the filter:

    $ echo '.filter' >foo.jq
    $ echo '.thing' >bar.jq
    $ JQ -f foo.jq -c --fromfile bar.jq
     jq -c .filter\|.thing

# Unless the file doesn't exist or can't be read:
    $ JQ -f nosuch.file
    */jqmd.md: line *: nosuch.file: No such file or directory (glob)
    [69]
````

### Capturing JQ Output

The `CALL_JQ` command is just like `RUN_JQ`, except that it captures the output in `REPLY`, while still clearing filters and such.  It also returns jq's exit status.

````sh
# No filter

    $ CALL_JQ -- <(echo '{}')
    $ echo "$REPLY"
    {}

# With filter

    $ CALL_JQ .foo-2 <(echo '{"foo":42}')
    $ echo "$REPLY"
    40

# Exit levels
    $ CALL_JQ -n -e '[] | .[]'  # force exit level 4
    [4]
````

