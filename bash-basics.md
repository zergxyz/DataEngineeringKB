# Bash Notes: Parsing a Pipe-Delimited Config File

### Fail fast, fail on undefined variables, and don't hide errors inside pipelines.
It's usually set up at the beginning of the script file. 
```bash
set -euo pipefail
```
### Bash parameter expansion.
Regular variable: 
```bash
${VAR:-default}
TABLE_CONFIG_FILE="${TABLE_CONFIG_FILE:-${SCRIPT_DIR}/others.txt}"
```
Required variable expansion: 
```bash
${VAR:?error_message}
JSL_LICENSE="${JSL_LICENSE:?export the JSL licence JWT before submitting}"
```

### String Operations
non-empty string test
```bash
if [ -n "$INPUT_PARTITIONS" ]; then
```
Remove leading and trailing hypens
```bash
# These remove at most one hyphen. If an input started with __, the first step turned it into --, and this will leave a single leading -.
value="${value#-}"
value="${value%-}"
```
Array and string convertion
```bash
SPARK_PROPERTIES=(
  "spark.serializer=org.apache.spark.serializer.KryoSerializer"
  ...
  "spark.dataproc.executorEnv.SPARK_NLP_LICENSE=${JSL_LICENSE}"
)
SPARK_PROPERTIES_CSV="$(IFS=,; printf '%s' "${SPARK_PROPERTIES[*]}")"
# When (Internal Field Separator)IFS=,, this expands to:prop1,prop2,prop3,prop4,...
```






A study guide based on a real loop that reads `table_name|text_columns` rows, validates them, and stores them in parallel arrays.

---

## 1. The Full Pattern

```bash
TABLE_NAMES=()
TEXT_COLUMN_SETS=()

while IFS='|' read -r raw_table_name raw_text_columns extra || [ -n "${raw_table_name:-}${raw_text_columns:-}${extra:-}" ]; do
  table_name="$(trim "${raw_table_name:-}")"
  text_columns="$(trim "${raw_text_columns:-}")"
  extra_field="$(trim "${extra:-}")"

  [ -z "$table_name$text_columns$extra_field" ] && continue   # blank line
  [[ "$table_name" == \#* ]] && continue                      # comment
  [ "$table_name" = "table_name" ] && [ "$text_columns" = "text_columns" ] && continue  # header

  if [ -n "$extra_field" ]; then
    echo "Too many fields: ${table_name}|${text_columns}|${extra_field}" >&2
    exit 1
  fi
  if [ -z "$table_name" ] || [ -z "$text_columns" ]; then
    echo "Expected table_name|text_columns" >&2
    exit 1
  fi

  TABLE_NAMES+=("$table_name")
  TEXT_COLUMN_SETS+=("$text_columns")
done < "$TABLE_CONFIG_FILE"
```

---

## 2. Trick: `IFS` scoped to a single command

`IFS` (Internal Field Separator) controls how `read` splits a line. Putting the assignment *before* the command applies it **only for that command**.

```bash
IFS='|' read -r a b <<< "users|name,bio"
echo "$a"   # users
echo "$b"   # name,bio

echo "$IFS" | od -c   # IFS is still the default (space, tab, newline) afterwards
```

**Remember:** `VAR=value command` = temporary override, no cleanup needed.

---

## 3. Trick: `read -r`

Without `-r`, backslashes are treated as escape characters and get eaten.

```bash
echo 'C:\temp\new' | read x;     echo "$x"   # C:tempnew   (backslashes lost)
echo 'C:\temp\new' | read -r x;  echo "$x"   # C:\temp\new
```

**Rule of thumb:** always use `read -r` unless you specifically want escape processing.

---

## 4. Trick: Extra variable catches the leftovers

When `read` has more fields than variables, the **last variable gets the entire remainder** of the line (including delimiters).

```bash
IFS='|' read -r a b c <<< "x|y|z|w"
echo "$a"   # x
echo "$b"   # y
echo "$c"   # z|w      <-- remainder, delimiter included
```

When there are fewer fields than variables, the rest are empty:

```bash
IFS='|' read -r a b c <<< "x|y"
echo "[$c]"   # []
```

The loop uses this on purpose: a non-empty `extra` means the row had too many fields, so it can be rejected.

---

## 5. Trick: Last line without a trailing newline

`read` returns a **non-zero exit code** at EOF if the final line has no `\n`, even though it successfully filled the variables. A plain `while read` would skip that line.

```bash
printf 'a|b\nc|d' > test.txt      # note: no trailing newline

# BUG: misses "c|d"
while IFS='|' read -r x y; do echo "got $x,$y"; done < test.txt
# got a,b

# FIX: also run the body if the variables contain data
while IFS='|' read -r x y || [ -n "$x$y" ]; do echo "got $x,$y"; done < test.txt
# got a,b
# got c,d
```

**Pattern:** `while read ... || [ -n "$var" ]; do`

---

## 6. Trick: `${var:-}` (safe with `set -u`)

With `set -u` (error on unset variables), referencing an unset variable aborts the script. `${var:-default}` supplies a default; an empty default is fine.

```bash
set -u
unset foo
echo "$foo"        # error: foo: unbound variable
echo "${foo:-}"    # prints empty string, no error
echo "${foo:-N/A}" # N/A
```

---

## 7. Trick: Redirect into the loop, don't pipe into it

Pipes run each side in a **subshell**, so variables set inside the loop vanish afterwards.

```bash
# BAD: array is empty after the loop
items=()
cat file.txt | while read -r line; do items+=("$line"); done
echo "${#items[@]}"   # 0

# GOOD: loop runs in the current shell
items=()
while read -r line; do items+=("$line"); done < file.txt
echo "${#items[@]}"   # number of lines
```

Alternative when the source is a command: process substitution.

```bash
while read -r line; do items+=("$line"); done < <(some_command)
```

---

## 8. A `trim` helper (assumed to exist elsewhere)

The script calls `trim`, which isn't shown. A common implementation:

```bash
trim() {
  local s="$1"
  s="${s#"${s%%[![:space:]]*}"}"   # remove leading whitespace
  s="${s%"${s##*[![:space:]]}"}"   # remove trailing whitespace
  printf '%s' "$s"
}

trim "   hello world  "   # "hello world"
```

Called as `"$(trim "$x")"`: command substitution captures the output.

Why trim? `read` with `IFS='|'` does **not** strip spaces around fields, so `users | name` would give `"users "` and `" name"`.

---

## 9. Concatenation test for "all empty"

Joining strings and testing once is a compact way to check that *all* are empty.

```bash
a=""; b=""; c=""
[ -z "$a$b$c" ] && echo "all empty"

c="x"
[ -z "$a$b$c" ] || echo "at least one has content"
```

Same idea in the `||` clause: `[ -n "$a$b$c" ]` = "at least one is non-empty".

---

## 10. Matching a literal `#` in `[[ ]]`

Inside `[[ ... == pattern ]]`, the right side is a **glob pattern**. `#` is special to the shell in some contexts, so it's escaped.

```bash
s="# a comment"
[[ "$s" == \#* ]] && echo "comment"     # matches strings starting with #
[[ "$s" == "#"* ]] && echo "comment"    # equivalent: quote the #, leave * unquoted
```

**Remember:** quote the literal part, leave the wildcard unquoted. `[[ ]]` is Bash-specific (not POSIX `sh`).

---

## 11. `continue` for skip-rules (guard clauses)

Handle "ignore this line" cases early, so the rest of the loop body only deals with valid data.

```bash
for n in 1 2 3 4 5; do
  [ $((n % 2)) -eq 0 ] && continue
  echo "odd: $n"
done
# odd: 1, 3, 5
```

---

## 12. Errors to stderr, then exit non-zero

```bash
echo "Something went wrong" >&2   # send to stderr (fd 2)
exit 1                            # non-zero = failure
```

- `>&2` keeps error messages out of normal output, so they don't pollute pipes or redirected files.
- `exit 1` aborts the whole script; use `return 1` inside a function instead.

---

## 13. Appending to arrays and parallel arrays

```bash
names=()
cols=()

names+=("users");  cols+=("name,bio")
names+=("posts");  cols+=("title,body")

echo "${#names[@]}"     # 2  (length)
echo "${names[0]}"       # users
echo "${names[@]}"       # users posts

# Iterate both together by index
for i in "${!names[@]}"; do
  echo "${names[$i]} -> ${cols[$i]}"
done
# users -> name,bio
# posts -> title,body
```

Always quote `"$value"` inside `+=(...)` so values with spaces stay as one element.

---

## 14. Worked Example

**Config file** (`tables.conf`):

```
table_name|text_columns
# user data
users|name,bio

posts | title,body
```

**Result:**

```bash
TABLE_NAMES=(users posts)
TEXT_COLUMN_SETS=("name,bio" "title,body")
```

**Rows that would abort the script:**

| Row            | Reason                          |
|----------------|---------------------------------|
| `users\|a\|b`  | third field → "too many fields" |
| `users\|`      | empty `text_columns`            |
| `\|name`       | empty `table_name`              |

---

## 15. Cheat Sheet

| Goal                              | Snippet                                      |
|-----------------------------------|----------------------------------------------|
| Split on a custom delimiter       | `IFS='\|' read -r a b`                       |
| Don't mangle backslashes          | `read -r`                                    |
| Catch extra fields                | one more variable than expected              |
| Handle missing final newline      | `\|\| [ -n "$var" ]`                         |
| Safe with `set -u`                | `"${var:-}"`                                 |
| Keep variables after loop         | `done < file` (not `cat file \|`)            |
| Check all vars empty              | `[ -z "$a$b$c" ]`                            |
| Skip a line                       | `continue`                                   |
| Match "starts with #"             | `[[ "$s" == \#* ]]`                          |
| Print error                       | `echo "msg" >&2`                             |
| Abort script                      | `exit 1`                                     |
| Append to array                   | `arr+=("$value")`                            |

---

## 16. Practice Ideas

1. Modify the loop to also accept a 3rd optional `schema` column instead of rejecting it.
2. Add support for `;` comments after a row (`users|name ; note`).
3. Rewrite `trim` using `xargs` or `sed`, then compare with the pure-Bash version.
4. Turn the two parallel arrays into one associative array (`declare -A cols; cols[users]="name,bio"`).
