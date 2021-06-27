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

You can generate args with `ARGSTR` and `ARGVAL`:

~~~shell
    $ ARGSTR 'foo"bar'; echo $REPLY
    $JQMD_QA_1
    $ ARGSTR spammity; echo $REPLY
    $JQMD_QA_4
    $ ARGVAL true; echo $REPLY
    $JQMD_JA_7
    $ printf ' %q' "${JQ_OPTS[@]}"; echo
     jq --arg JQMD_QA_1 foo\"bar --arg JQMD_QA_4 spammity --argjson JQMD_JA_7 true
    $ RUN_JQ -n '$JQMD_JA_7'
    true
~~~

JSON quoting can be done with `JSON-QUOTE`, `JSON-LIST`, and `JSON-KV`, or via extra args to `FILTER` or `JSON`:

~~~shell
    $ JSON-QUOTE $'\n\r\t' $'\x01\x1f' "\\"; echo "${REPLY[@]}"
    "\n\r\t" "\u0001\u001f" "\\"

    $ JSON-LIST; echo "${REPLY[@]}"
    []

    $ JSON-LIST foo bar baz; echo "${REPLY[@]}"
    ["foo", "bar", "baz"]

    $ JSON-KV; echo "${REPLY[@]}"
    {}

    $ JSON-KV foo=blue bar=baz spam; echo "${REPLY[@]}"
    {"foo": "blue", "bar": "baz", "spam": "spam"}

    $ FILTER 'foo(%s; %s)' bar 'baz"spam'; echo "$jqmd_filters"
    foo("bar"; "baz\"spam")

    $ CLEAR_FILTERS
    $ JSON '{%s: %s}' foo bar; echo "$jqmd_filters"
    jqmd_data({"foo": "bar"})
~~~

If you're on bash 4, you can also `JSON-MAP` an associative array to a JSON object:

~~~shell
    $ if (( BASH_VERSINFO > 3 )); then
    >     declare -A empty_map mymap=([foo]=42 [bar]=baz [bing]=boom)
    >     JSON-MAP mymap;    echo "${REPLY[@]}"
    >     JSON-MAP emptymap; echo "${REPLY[@]}"
    > else # fake it for bash 3
    >     echo '{"bing": "boom", "bar": "baz", "foo": "42"}'
    >     echo '{}'
    > fi
    {"bing": "boom", "bar": "baz", "foo": "42"}
    {}
~~~

You can also `APPLY` a jq expression with jq variables bound to shell variables or values, either as strings or raw jq expressions:

~~~shell
    $ CLEAR_FILTERS
    $ foo='bar"baz' bar='27'

    $ APPLY . foo; echo "$jqmd_filters"; CLEAR_FILTERS
    "bar\"baz" as $foo

    $ APPLY 'bang($bar)'; echo "$jqmd_filters"; CLEAR_FILTERS
    ( bang($bar) )

    $ APPLY 'bang($bar)' @bar; echo "$jqmd_filters"; CLEAR_FILTERS
    ( ("27"|fromjson) as $bar | bang($bar) )

    $ APPLY . foo @bar baz=thingy @spam=54; echo "$jqmd_filters"; CLEAR_FILTERS
    "bar\"baz" as $foo | ("27"|fromjson) as $bar | "thingy" as $baz | ("54"|fromjson) as $spam

# Longer values get passed as arguments instead of quoted:

    $ APPLY . foo="This is a relatively long string. It should be passed as an argument."
    $ APPLY . @bar='"This is a rather long JSON value. It should be passed as an argument."'
    $ echo "$jqmd_filters"
    $JQMD_QA_1 as $foo
    | $JQMD_JA_4 as $bar

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
     jq -c $'.filter\n| .thing'

# Unless the file doesn't exist or can't be read:
    $ JQ -f nosuch.file
    */jqmd.md: line *: nosuch.file: No such file or directory (glob)
    [69]

# But if the total filter size is too long, it will use an inline file:
    $ JQ_SIZE_LIMIT=10 JQ -f foo.jq -c  --fromfile bar.jq
     JQ_WITH_FILE 2 jq -c $'.filter\n| .thing'
    $ (function jq(){ echo jq "$@"; cat "$3"; }; JQ_WITH_FILE 2 jq -c $'.filter\n| .thing' x y z; )
    jq -c -f /dev/fd/63 x y z
    .filter
    | .thing
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

