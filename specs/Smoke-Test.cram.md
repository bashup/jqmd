## Basic check that jqmd handles jq and json blocks, JQ_OPTS, post-run RUN_JQ, etc.:

    $ cat <<'-' >test.md
    > ```jq
    > def jqmd_data($x): $x; .
    > ```
    > ```json
    > { "x": "y" }
    > ```
    > ```jq
    > .x
    > ```
    > ```shell
    > JQ_OPTS -n
    > ```
    > -
    $ $TESTDIR/../jqmd test.md
    "y"
