## Basic check that jqmd handles jq and json blocks, JQ_OPTS, post-run RUN_JQ, etc.:

    $ cat <<'-' >test.md
    > ```jq
    > def jqmd_data($x): . + $x; .
    > ```
    > ```json
    > { "x": "y" }
    > ```
    > ```yaml
    > a: b
    > ```
    > ```shell
    > JQ_OPTS -n
    > ```
    > -
    $ $TESTDIR/../jqmd.md test.md
    {
      "x": "y",
      "a": "b"
    }
