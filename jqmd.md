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
  * [JSON Data Structures](#json-data-structures)
  * [jq options and arguments](#jq-options-and-arguments)
  * [Invoking jq](#invoking-jq)
  * [YAML and JSON data](#yaml-and-json-data)
    + [jqmd::data](#jqmddata)
    + [YAML Interpolation](#yaml-interpolation)
    + [YAML to JSON Conversion (yaml2json)](#yaml-to-json-conversion-yaml2json)
- [Main Program](#main-program)

<!-- tocstop -->

## File Header

The main program begins with a `#!` line and edit warning, followed by its license text, and the source of mdsh:

```shell mdsh
@module jqmd.md
@main mdsh-main

@require pjeby/license @comment LICENSE
@require bashup/mdsh   mdsh-source "$BASHER_PACKAGES_PATH/bashup/mdsh/mdsh.md"
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
CLEAR_FILTERS() { unset jqmd_filters; JQ_OPTS=(jq); }

IMPORTS() { jqmd_imports+="${jqmd_imports:+$'\n'}$1"; }
DEFINE()  { jqmd_defines+="${jqmd_defines:+$'\n'}$1"; }
FILTER()  {
	case $# in
	1) jqmd_filters+="${jqmd_filters:+$'\n'| }$1"; return ;;
	0) return ;;
	esac
	local REPLY ARGS=(printf -v REPLY "$1"); shift
	JSON-QUOTE "$@"; ARGS+=("${REPLY[@]}"); "${ARGS[@]}"; FILTER "$REPLY"
}

APPLY() {
	local name filter='' REPLY expr=${1-} lf=$'\n'; shift
	while (($#)); do
		name=${1#@}; REPLY=${name#*=}
		[[ $name == *=* ]] || REPLY=${!name}
		if ((${#REPLY} > 32)); then
			if [[ $1 == @* ]]; then ARGVAL "$REPLY"; else ARGSTR "$REPLY"; fi
		else
			JSON-QUOTE "$REPLY"; [[ $1 != @* ]] || REPLY="($REPLY|fromjson)"
		fi
		filter+=" | $REPLY as \$${name%%=*}"; shift
	done
	filter=${filter:3}; [[ ! $expr || $expr == . ]] || filter="( ${filter:+$filter | }$expr )"
	${filter:+FILTER "$filter"}
}
```

### JSON Data Structures

```bash runtime
JSON-QUOTE() {
	local LC_ALL=C
	[[ $* != *\\* ]] || set -- "${@//\\/\\\\}"
	[[ $* != *'"'* ]] || set -- "${@//\"/\\\"}"
	set -- "${@/#/\"}"; set -- "${@/%/\"}"
	if [[ $* != *[$'\x01'-$'\x1F']* ]]; then REPLY=("$@"); else escape-ctrlchars "$@"; fi
}

JSON-LIST() {
	(($#)) || { REPLY=("[]"); return; }
	JSON-QUOTE "$@"; REPLY=${REPLY[*]/%/,}; REPLY=("[${REPLY%,}]")
}

JSON-MAP() {
	local -n v=$1; local LC_ALL=C n=${v[@]+${#v[@]}}; ((${n:-0})) || { REPLY=("{}"); return; }
	local i=1 p='\"${@:i:1}\": \"${@: '"$n"'+ i++:1}\",'; p="${v[*]/#*/$p}"
	set -- "${!v[@]}" "${v[@]}"
	[[ $* != *\\* ]] || set -- "${@//\\/\\\\}"
	[[ $* != *\"* ]] || set -- "${@//\"/\\\"}"
	eval "REPLY=(\"{${p%,}}\")"
	[[ $REPLY != *[$'\x01'-$'\x1F']* ]] || escape-ctrlchars "$REPLY"
}

JSON-KV() {
	(($#)) || { REPLY=("{}"); return; }
	local LC_ALL=C i=0 p='\"${REPLY[i]}\": \"${REPLY['"$#"'+i++]}\",'; p="${*/#*/$p}"
	[[ $* != *\\* ]] || set -- "${@//\\/\\\\}"
	[[ $* != *\"* ]] || set -- "${@//\"/\\\"}"
	REPLY=("${@%%=*}" "${@#*=}"); eval "REPLY=(\"{${p%,}}\")"
	[[ $REPLY != *[$'\x01'-$'\x1F']* ]] || escape-ctrlchars "$REPLY"
}

escape-ctrlchars() {
	local LC_ALL=C
	set -- "${@//$'\n'/\\n}"; set -- "${@//$'\r'/\\r}"; set -- "${@//$'\t'/\\t}"  # \n\r\t
	if [[ $* == *[$'\x01'-$'\x1F']* ]]; then
		local r s=$*; s=${s//[^$'\x01'-$'\x1F']}
		while [[ $s ]]; do
			printf -v r \\\\u%04x "'${s:0:1}"; set -- "${@//${s:0:1}/$r}"
			s=${s//${s:0:1}/}
		done
	fi
	REPLY=("$@")
}

```

### jq options and arguments

```bash runtime
JQ_CMD=(jq)
JQ_OPTS=(jq)
JQ_SIZE_LIMIT=16000
JQ_OPTS() { JQ_OPTS+=("$@"); }
ARG()     { JQ_OPTS --arg     "$1" "$2"; }
ARGJSON() { JQ_OPTS --argjson "$1" "$2"; }
ARGQUOTE() { ARGSTR "$1"; }  # deprecated
ARGSTR() { REPLY=JQMD_QA_${#JQ_OPTS[@]}; ARG     "$REPLY" "$1"; REPLY='$'$REPLY; }
ARGVAL() { REPLY=JQMD_JA_${#JQ_OPTS[@]}; ARGJSON "$REPLY" "$1"; REPLY='$'$REPLY; }
```

### Invoking jq

```bash runtime
JQ_CMD() {
	local f= opt nargs cmd=("${JQ_CMD[@]}"); set -- "${JQ_OPTS[@]:1}" "$@"

	while (($#)); do
		case "$1" in
		-f|--fromfile)
			opt=$(<"$2") || return 69
			FILTER "$opt"; shift 2; continue
			;;
		-L|--indent)                            nargs=2 ;;
		--arg|--argjson|--slurpfile|--argfile)  nargs=3 ;;
		--)  break   ;; # rest of args are data files
		-*)  nargs=1 ;;
		*)   FILTER "$1"; break ;;	# jq program: data files follow
		esac
		cmd+=("${@:1:$nargs}")	# add $nargs args to cmd
		shift $nargs
	done

	HAVE_FILTERS || FILTER .    # jq needs at least one filter expression
	for REPLY in "${jqmd_imports-}" "${jqmd_defines-}" "${jqmd_filters-}"; do
		[[ $REPLY ]] && f+=${f:+$'\n'}$REPLY
	done

	REPLY=("${cmd[@]}" "$f" "${@:2}")
	if ((${#f} > JQ_SIZE_LIMIT)); then REPLY=(JQ_WITH_FILE "${#cmd[@]}" "${REPLY[@]}"); fi
	CLEAR_FILTERS   # cleanup for any re-runs
}

JQ_WITH_FILE() { local n=$(($1++2)); "${@:2:$1}" -f <(echo "${!n}") "${@:n+1}"; }
RUN_JQ() { JQ_CMD "$@" && "${REPLY[@]}"; }
CALL_JQ() { JQ_CMD "$@" && REPLY=("$("${REPLY[@]}")"); }
```

### YAML and JSON data

YAML and JSON blocks are just jq filter expressions wrapped in a call to `jqmd::data()` (or an appropriate alternative, selected by a `@data` directive in a mdsh block).

```bash runtime
YAML()  { y2j "$1"; JSON "$REPLY"; }
JSON()  { FILTER "jqmd_data($1)" "${@:2}"; }
```

#### jqmd::data

The `jqmd::data` function provides a default wrapper to convert YAML or JSON data to a jq filter.  It recursively merges dictionaries and concatenates (or appends to) arrays.

```bash runtime
DEFINE '
def jqmd::blend($other; combine): . as $this | . *  $other | . as $combined | with_entries(
  if (.key | in($this)) and (.key | in($other)) then
    .this = $this[.key] | .other = $other[.key] | combine
  else . end
);

def jqmd::combine: (.this|type) as $this | (.other|type) as $other | .value =
  if $this == "array" then
    if $other == "array" then .this + .other else .this + [.other] end
  elif $this == "object" then
    if $other == "object" then
      .other as $o | (.this | jqmd::blend($o; jqmd::combine))
    else .other end
  else .other end;  # everything else just overrides

def jqmd::data($data): {this: ., other:$data} | jqmd::combine | .value ;
def jqmd_data($data): jqmd::data($data) ;
'
```
#### YAML Interpolation

YAML blocks are allowed to have jq string interpolation expressions.  But since such expressions are invalid JSON, and the yaml2json filter produces only valid JSON, it's necessary to selectively *invalidate* the resulting JSON before it becomes a JQ filter.  Specifically, either one or two backslashes must be removed before every open parenthesis, depending on whether the total number of preceding backslashes is divisible by four (i.e., the original number of backslashes was divisible by two.)

```bash runtime
y2j() {
	local p j; j="$(echo "$1" | yaml2json)" || return $?; REPLY=
	while [[ $j == *'\\('* ]]; do
		p=${j%%'\\('*}; j=${j#"$p"'\\('}
		if [[ $p =~ (^|[^\\])('\\\\')*$ ]]; then
			p="${p}"'\(' # odd, unbalance the backslash
		else
			p="${p}("   # even, remove one actual backslash
		fi
		REPLY+=$p
	done
	REPLY+=$j
}
```
#### YAML to JSON Conversion (yaml2json)

The `yaml2json` function is a wrapper that automatically selects one of `yaml2json:cmd`, `yaml2json:py`, or `yaml2json:php` to perform YAML to JSON conversions.  It does so by piping a small YAML input to each function and testing whether the result is a valid JSON conversion.

```bash runtime
yaml2json:cmd() { command yaml2json /dev/stdin; }

yaml2json:py() {
	python -c 'import sys, yaml, json; json.dump(yaml.safe_load(sys.stdin), sys.stdout)'
}

yaml2json:php() { command yaml2json.php; }

yaml2json() {
	local kind  # auto-select between available yaml2json implementations
	for kind in cmd py php; do
		REPLY=($(yaml2json:$kind < <(echo "a: {}") 2>/dev/null || true))
		printf -v REPLY %s ${REPLY+"${REPLY[@]}"}
		if [[ "$REPLY" == '{"a":{}}' ]]; then
			eval "yaml2json() { yaml2json:$kind; }"; yaml2json; return
		fi
	done
	exit 69 "To process YAML, must have one of: yaml2json, PyYAML, or yaml2json.php" # EX_UNAVAILABLE
}

# --- END jqmd runtime ---
```

## Main Program

The non-runtime part of the program defines hooks for mdsh to be able to compile jq, yaml, and json blocks and constants:

```shell
# Language Support
mdsh-compile-jq()         { printf 'FILTER %q\n' "$1"$'\n'; }
mdsh-compile-jq_defs()    { printf 'DEFINE %q\n' "$1"$'\n'; }
mdsh-compile-jq_imports() { printf 'IMPORTS %q\n' "$1"$'\n'; }

mdsh-compile-yml()  { y2j "$1"; mdsh-compile-json "$REPLY"; }
mdsh-compile-yaml() { y2j "$1"; mdsh-compile-json "$REPLY"; }
mdsh-compile-json() { mdsh-compile-jq "jqmd_data($1)"; }

mdsh-compile-func() {
	case ${tag_words-} in
		yml|yaml) y2j "$1"; REPLY="jqmd_data($REPLY)"$'\n' ;;
		json*) REPLY="jqmd_data($1)"$'\n' ;;
		jq|javascript|js) REPLY=$1 ;;
		*) mdsh-error "Invalid language for function: '%s'" "${tag_words-}"; return
	esac
	printf 'function %s() {\n\tAPPLY %q \\\n\t\t%s\n}\n' "${tag_words[2]}" "$REPLY" \
		"${2#*${tag_words}*${tag_words[1]}*${tag_words[2]}}"
}

const() {
	case "${mdsh_lang-}" in
	yaml|yml) y2j "$mdsh_block"; printf 'DEFINE %q\n' "def $1: $REPLY ;"$'\n' ;;
	json)     printf 'DEFINE %q\n' "def $1: $mdsh_block ;"$'\n' ;;
	*) mdsh-error "Invalid language for constant: '%s'" "${mdsh_lang-}"
	esac
}
```

It also evals the runtime, and defines header/footer hooks to embed the runtime in compiled scripts, and to ensure jq is run at the script's end if there are any leftover filters:

```shell
# Load the runtime so it's usable by mdsh
printf -v REPLY '%s\n' "${mdsh_raw_bash_runtime[@]}"; eval "$REPLY"

# Add runtime to the top of compiled (main) scripts
printf -v REPLY 'mdsh:file-header() { ! @is-main || echo -n %q; }' "$REPLY"; eval "$REPLY"

# Ensure (main) scripts process any leftover filters at end
mdsh:file-footer() { ! @is-main || echo $'if [[ $0 == "${BASH_SOURCE[0]-}" ]] && HAVE_FILTERS; then RUN_JQ; fi'; }
```

It also defines a few command line options for controlling compilation:

```shell
mdsh.--no-runtime() ( unset -f mdsh:file-header mdsh:file-footer; mdsh-main "$@"; )
mdsh.--yaml() (
	fn-exists "yaml2json:${1-}" || exit $? "No such yaml2json processor: ${1-}"
	eval 'yaml2json() { yaml2json:'"$1"'; }'
	mdsh-main "${@:2}"
)

mdsh.-R() { mdsh.--no-runtime "$@"; }
mdsh.-y() { mdsh.--yaml "$@"; }
```
