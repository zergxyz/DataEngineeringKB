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
