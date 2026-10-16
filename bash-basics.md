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
