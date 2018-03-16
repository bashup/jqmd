## YAML Interpolation

jqmd supports jq interpolation in YAML blocks, which requires changing the number of backslashes found before an open parenthesis.  Since YAML escapes *all* backslashes in the input (to create valid JSON), this requires dealing with everything as doubled backslashes.  In the case where the original number of backslashes was odd, a single backslash is removed, making the JSON invalid (and therefore suitable for a jq expression).  In the case where the original number was even, *two* backslashes are removed, which removes one of the original backslashes (that was escaping the interpolation).

So this YAML:

```yaml
ultimate answer: \(40+2)
not \\(a key: \\\\(or value
```

Should produce this output:

~~~shell
    $ "$TESTDIR/../jqmd.md" "$TESTDIR/$TESTFILE" </dev/null
    {
      "not \\(a key": "\\\\\\(or value",
      "ultimate answer": "42"
    }
~~~

Assuming that jq is run with null input and the result keys are sorted:

```jq defs
def jqmd_data($x): . + $x;
```

```shell
RUN_JQ -n -S
```
