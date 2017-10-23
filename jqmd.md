#!/usr/bin/env bash
: '
<!-- ex: set syntax=markdown : '; eval "$(mdsh -E "$BASH_SOURCE")"; # -->

# jqmd - literate jq programming

jqmd is an mdsh extension written as a literate program using mdsh.  Within this source file, `shell` code blocks are the main program, while `bash runtime` blocks are captured as data to be used as part of the runtime compiled into jqmd programs.

### Contents

<!-- toc -->

- [File Header](#file-header)
- [Runtime](#runtime)
  * [jq imports, filters, and defines](#jq-imports-filters-and-defines)
  * [jq options and arguments](#jq-options-and-arguments)
  * [Invoking jq](#invoking-jq)
  * [YAML and JSON data](#yaml-and-json-data)
- [Main Program](#main-program)

<!-- tocstop -->

## File Header

The main program begins with a `#!` line and edit warning:

```shell
#!/usr/bin/env bash
# ---
# This file was automatically generated from jqmd.md - DO NOT EDIT!
# ---
```

Followed by its license text:

```shell mdsh
# incorporate the LICENSE file as bash comments
source realpaths; realpath.location "$MDSH_SOURCE"
echo; sed -e '1,2d; s/^\(.\)/# \1/; s/^$/#/;' "$REPLY/LICENSE"; echo
```

## Runtime

The runtime also begins with a `#!` line, so that compiled scripts will begin with one too:

```bash runtime
#!/usr/bin/env bash

# --- BEGIN jqmd runtime ---
```

### jq imports, filters, and defines

```bash runtime
jqmd_imports=
jqmd_filters=
jqmd_defines=

HAVE_FILTERS() { [[ ${jqmd_filters-} ]]; }
CLEAR_FILTERS() { unset jqmd_filters; }

IMPORTS() { jqmd_imports+="${jqmd_imports:+$'\n'}$1"; }
DEFINE()  { jqmd_defines+="${jqmd_defines:+$'\n'}$1"; }
FILTER()  { jqmd_filters+="${jqmd_filters:+|}$1"; }
```

### jq options and arguments

```bash runtime
JQOPTS=(jq)
JQ_OPTS() { JQOPTS+=("$@"); }
ARG()     { JQ_OPTS --arg     "$1" "$2"; }
ARGJSON() { JQ_OPTS --argjson "$1" "$2"; }
```

### Invoking jq

```bash runtime
RUN_JQ() {
    local opt nargs cmd=(jq) opts=("${JQOPTS[@]:1}" "$@")

    while (("${#opts[@]}")); do
        opt="${opts[0]}"
        case "$opt" in
        -{f,-fromfile})                     nargs=2 ; FILTER <"${opts[1]}" ;;
        -{L,-indent})                       nargs=2 ;;
        --{arg,arjgson,slurpfile,argfile})  nargs=3 ;;
        --)  break   ;; # rest of args are data files
        -*)  nargs=1 ;;
        *)   FILTER "$opt"; break ;; # jq program: data files follow
        esac
        cmd+=("${opts[@]:0:$nargs}")    # add $nargs args to cmd
        opts=("${opts[@]:$nargs}")      # shift $nargs args off opts
    done

    HAVE_FILTERS || FILTER .    # jq needs at least one filter expression

    opts[0]=n/a; "${cmd[@]}" -f <(
        printf "%s\n" "${jqmd_imports-}" "${jqmd_defines-}" "${jqmd_filters-}"
    ) "${opts[@]:1}"

    CLEAR_FILTERS   # cleanup for any re-runs
}
```

### YAML and JSON data

```bash runtime
YAML()    { JSON "$(echo "$1" | yaml2json -)"; }
JSON()    { FILTER "jqmd_data($1)"; }

command -v yaml2json >/dev/null || yaml2json() {
    python -c 'import sys, yaml, json; json.dump(yaml.safe_load(sys.stdin), sys.stdout, indent=4)';
}

# --- END jqmd runtime ---
```

## Main Program

The non-runtime part of the program defines hooks for mdsh to be able to compile jq, yaml, and json blocks:

```shell
# Language Support
mdsh-compile-jq()   { printf 'FILTER %q\n' "$1"; }
mdsh-compile-yml()  { printf 'JSON %q\n' "$(echo "$1" | yaml2json -)"; }
mdsh-compile-yaml() { printf 'JSON %q\n' "$(echo "$1" | yaml2json -)"; }
mdsh-compile-json() { printf 'JSON %q\n' "$1"; }
```

It also evals the runtime, and defines header/footer hooks to embed the runtime in compiled scripts, and to ensure jq is run at the script's end if there are any leftover filters:

```shell
# Load the runtime so it's usable by mdsh
printf -v REPLY '%s\n' "${mdsh_raw_bash_runtime[@]}"; eval "$REPLY"

# Add runtime to the top of compiled scripts
printf -v REPLY 'mdsh:file-header() { echo -n %q; }' "$REPLY"; eval "$REPLY"

# Ensure scripts process any leftover filters at end
mdsh:file-footer() { echo 'if [[ $0 == $BASH_SOURCE ]] && HAVE_FILTERS; then RUN_JQ; fi'; }
```

Finally, we include the source of mdsh directly, so our compiled version won't need it installed:

```mdsh
sed -e '1,4d; /^# Copyright (c) 2017 PJ Eby/,/^$/d'  "$(command -v mdsh)"
```
