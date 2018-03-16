## Data merging

~~~shell
    $ source "$TESTDIR/../jqmd.md"; set +e

# Null + any = any

    $ JSON '1'; RUN_JQ -n
    1
    $ JSON '"x"'; RUN_JQ -n
    "x"
    $ JSON 'true'; RUN_JQ -n
    true
    $ JSON '{x:17}'; RUN_JQ -n -c
    {"x":17}
    $ JSON '[2]'; RUN_JQ -n -c
    [2]

# Simple mappings
    $ JSON '{a:42, b:5}'
    $ JSON '{c:43, b:10}'
    $ RUN_JQ -n
    {
      "a": 42,
      "b": 10,
      "c": 43
    }

# Nested mappings

    $ JSON '{x: {a: 42, b: 5}, y: "z"}'
    $ JSON '{q: {z: 19}, x: {b: 16, c: 27}}'
    $ RUN_JQ -n -c
    {"x":{"a":42,"b":16,"c":27},"y":"z","q":{"z":19}}

# Array + Array = concatenation

    $ JSON '[1,2]'
    $ JSON '[3,4]'
    $ RUN_JQ -n -c
    [1,2,3,4]

# Array + non-array = append
    $ JSON '[]'
    $ JSON '{}'
    $ RUN_JQ -n -c
    [{}]

# Arrays inside mappings
    $ JSON '{a: [1,2]}'
    $ JSON '{a: [3]}'
    $ RUN_JQ -n -c
    {"a":[1,2,3]}
~~~
