# jka
#!/system/bin/sh
TS_DIR="/data/adb/tricky_store"
TARGET_KEYBOX="$TS_DIR/keybox.xml"
TMP_DIR="/data/local/tmp/keybox_update"
TMP_RAW="$TMP_DIR/raw.tmp"
TMP_KEYBOX="$TMP_DIR/keybox_tmp.xml"

YURIKEY_URL="https://raw.githubusercontent.com/Yurii0307/yurikey/main/key"
TRICKYADDON_URL="https://raw.githubusercontent.com/KOWX712/Tricky-Addon-Update-Target-List/keybox/.extra"
INTEGRITYBOX_URL="https://raw.githubusercontent.com/MeowDump/MeowDump/refs/heads/main/NullVoid/OptimusPrime"
UPDATE_JSON_URL="https://raw.githubusercontent.com/Alan-qwq/TrickyStoreHelper/main/update.json"
SECURITY_BULLETIN_URL="https://source.android.google.cn/docs/security/bulletin/pixel"

CURRENT_VERSION="1.0.2"
SCRIPT_PATH=""
GITHUB_PROXY=""

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
PURPLE='\033[0;35m'
CYAN='\033[0;36m'
NC='\033[0m'

TOYBOX_COMMANDS=""
BUSYBOX_COMMANDS=""

init_command_cache() {
  init_command_cache__toybox_out=""
  init_command_cache__busybox_out=""
  if command -v toybox >/dev/null 2>&1; then
    init_command_cache__toybox_out=$(toybox --list 2>/dev/null | tr '\n' ' ')
    TOYBOX_COMMANDS=" $init_command_cache__toybox_out "
  fi
  if command -v busybox >/dev/null 2>&1; then
    init_command_cache__busybox_out=$(busybox --list 2>/dev/null | tr '\n' ' ')
    BUSYBOX_COMMANDS=" $init_command_cache__busybox_out "
  fi
  unset init_command_cache__toybox_out init_command_cache__busybox_out
}

run() {
  run__cmd="$1"
  shift
  case "$TOYBOX_COMMANDS" in
    *" $run__cmd "*)
      toybox "$run__cmd" "$@"
      return $?
      ;;
  esac
  case "$BUSYBOX_COMMANDS" in
    *" $run__cmd "*)
      busybox "$run__cmd" "$@"
      return $?
      ;;
  esac
  if command -v "$run__cmd" >/dev/null 2>&1; then
    "$run__cmd" "$@"
    return $?
  fi
  printf "${RED}[ERROR]${NC} 命令 '%s' 不可用（toybox/busybox/系统均未找到）\n" "$run__cmd" >&2
  unset run__cmd
  return 127
}

log_info() {
  printf "${GREEN}[INFO]${NC} %s\n" "$1"
}

log_warn() {
  printf "${YELLOW}[WARN]${NC} %s\n" "$1" >&2
}

log_error() {
  printf "${RED}[ERROR]${NC} %s\n" "$1" >&2
}

clear_screen() {
  printf "\033c"
}

version_ge() {
  version_ge__ver1="$1"
  version_ge__ver2="$2"
  version_ge__i=1
  while [ "$version_ge__i" -le 5 ]; do
    version_ge__n1=$(printf "%s" "$version_ge__ver1" | run cut -d. -f"$version_ge__i" 2>/dev/null)
    version_ge__n2=$(printf "%s" "$version_ge__ver2" | run cut -d. -f"$version_ge__i" 2>/dev/null)
    version_ge__n1=${version_ge__n1:-0}
    version_ge__n2=${version_ge__n2:-0}
    version_ge__n1=$(printf "%s" "$version_ge__n1" | run sed 's/^0*//')
    version_ge__n1=${version_ge__n1:-0}
    version_ge__n2=$(printf "%s" "$version_ge__n2" | run sed 's/^0*//')
    version_ge__n2=${version_ge__n2:-0}
    if [ "$version_ge__n1" -gt "$version_ge__n2" ]; then
      unset version_ge__ver1 version_ge__ver2 version_ge__i version_ge__n1 version_ge__n2
      return 0
    fi
    if [ "$version_ge__n1" -lt "$version_ge__n2" ]; then
      unset version_ge__ver1 version_ge__ver2 version_ge__i version_ge__n1 version_ge__n2
      return 1
    fi
    version_ge__i=$((version_ge__i + 1))
  done
  unset version_ge__ver1 version_ge__ver2 version_ge__i version_ge__n1 version_ge__n2
  return 0
}

show_decode_progress() {
  show_decode_progress__current="$1"
  show_decode_progress__total="$2"
  show_decode_progress__bar_len=20
  show_decode_progress__filled=$((show_decode_progress__current * show_decode_progress__bar_len / show_decode_progress__total))
  show_decode_progress__empty=$((show_decode_progress__bar_len - show_decode_progress__filled))

  show_decode_progress__bar=""
  show_decode_progress__j=0
  while [ "$show_decode_progress__j" -lt "$show_decode_progress__filled" ]; do
    show_decode_progress__bar="$show_decode_progress__bar#"
    show_decode_progress__j=$((show_decode_progress__j + 1))
  done

  show_decode_progress__empty_bar=""
  show_decode_progress__j=0
  while [ "$show_decode_progress__j" -lt "$show_decode_progress__empty" ]; do
    show_decode_progress__empty_bar="$show_decode_progress__empty_bar-"
    show_decode_progress__j=$((show_decode_progress__j + 1))
  done

  printf "${BLUE}[DECODE]${NC} [%s%s] %d/%d 层 \r" "$show_decode_progress__bar" "$show_decode_progress__empty_bar" "$show_decode_progress__current" "$show_decode_progress__total"
  if [ "$show_decode_progress__current" -eq "$show_decode_progress__total" ]; then
    printf "\n"
  fi
  unset show_decode_progress__current show_decode_progress__total show_decode_progress__bar_len show_decode_progress__filled show_decode_progress__empty show_decode_progress__bar show_decode_progress__empty_bar show_decode_progress__j
}

check_root() {
  check_root__uid=$(run id -u)
  if [ "$check_root__uid" -ne 0 ]; then
    log_error "需要 Root 权限!"
    exit 1
  fi
  unset check_root__uid
}

init_env() {
  run rm -rf "$TMP_DIR"
  run mkdir -p "$TMP_DIR"
  run mkdir -p "$TS_DIR"
}

get_script_path() {
  get_script_path__has_readlink=0
  get_script_path__dir=""
  get_script_path__name=""
 
