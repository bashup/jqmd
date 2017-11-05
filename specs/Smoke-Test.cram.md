## Basic check that jqmd handles jq and json blocks, const, JQ_OPTS, post-run RUN_JQ, etc.:

    $ cat <<'-' >test.md
    > ```jq
    > def jqmd_data($x): . + $x; .
    > ```
    > ```json
    > { "x": "y" }
    > ```
    > ```yaml ! const foo
    > a: b
    > ```
    > ```shell
    > JQ_OPTS -n '. + foo'
    > ```
    > -
    $ $TESTDIR/../jqmd.md test.md
    {
      "x": "y",
      "a": "b"
    }
