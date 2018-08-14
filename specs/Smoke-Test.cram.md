## Basic check that jqmd handles jq and json blocks, const, JQ_OPTS, post-run RUN_JQ, etc.:

    $ cat <<'-' >test.md
    > ```json
    > { "x": "y" }
    > ```
    > ```yaml ! const foo
    > a: \(env.FOO) - b
    > ```
    > ```yaml @func mksite SITE="$1" WP_HOME="$2"
    > services:
    >   \($SITE):
    >     environment:
    >       WP_HOME: \($WP_HOME)
    > ```
    > ```shell
    > JQ_OPTS -n '. + foo'
    > mksite dev https://bashup.github.io/whatever
    > ```
    > -
    $ FOO=bar $TESTDIR/../jqmd.md test.md
    {
      "x": "y",
      "services": {
        "dev": {
          "environment": {
            "WP_HOME": "https://bashup.github.io/whatever"
          }
        }
      },
      "a": "bar - b"
    }
