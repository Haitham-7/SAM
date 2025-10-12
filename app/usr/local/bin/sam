#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'

#=== CONFIG ===#
SSH_CONFIG="${SAM_CONFIG:-$HOME/.ssh/config}"
DEFAULT_USER="root"
DEFAULT_KEY="$HOME/.ssh/id_rsa"

#=== COLORS ===#
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[0;33m'
NC='\033[0m'

#=== HELP ===#
print_help() {
  cat <<EOF
SSH Alias Manager (sam)
Usage:
  sam help
  sam ls                                 List all SSH aliases (with index)
  sam add [options]                      Add SSH alias (interactive or CLI)
     [ -h HOST/IP ] [ -u USER ] [ -i KEY ] [ -a ALIAS ]
  sam remove -a ALIAS                    Remove SSH alias (by alias or index)
  sam edit -a ALIAS                      Edit SSH alias (interactive; alias or index)
  sam ssh <index>                        SSH into server by its index in the list

Options also available as: --add, --remove, --edit, --ls, --help
Environment:
  SAM_CONFIG   Path to primary SSH config (default: ~/.ssh/config)
EOF
}

#=== UTIL ===#
backup_config() {
  [[ -f "$SSH_CONFIG" ]] && cp "$SSH_CONFIG" "${SSH_CONFIG}.bak.$(date +%s)" || true
}

# Expand Include directives into a single temp file (read-only use)
expand_config() {
  local src="$1"
  local out; out="$(mktemp)"
  local dir line inc pattern
  dir="$(dirname "$src")"

  if [[ ! -f "$src" ]]; then
    echo -e "${RED}No SSH config found at $src${NC}" >&2
    : > "$out"
    echo "$out"
    return
  fi

  while IFS= read -r line || [[ -n "$line" ]]; do
    if [[ "$line" =~ ^[Ii]nclude[[:space:]]+(.+)$ ]]; then
      pattern="${BASH_REMATCH[1]}"
      pattern="${pattern/#\~/$HOME}"
      [[ "$pattern" != /* ]] && pattern="$dir/$pattern"
      while IFS= read -r inc; do
        [[ -f "$inc" ]] && cat "$inc" >> "$out"
      done < <(compgen -G "$pattern" || true)
    else
      echo "$line" >> "$out"
    fi
  done < "$src"

  echo "$out"
}

# Return newline-separated list of aliases from Host lines (ignore pure globs like "*" or "?")
collect_aliases() {
  local cfg="$1"
  awk '
    function is_glob(s) { return (s ~ /\*/ || s ~ /\?/); }
    tolower($1)=="host" {
      for (i=2; i<=NF; i++) if (!is_glob($i)) print $i;
    }
  ' "$cfg" | sed '/^$/d'
}

# Resolve effective values from OUR config only (apply wildcard/negation precedence)
# Prints: hostname=<v>\nuser=<v>\nidentityfile=<v>\nport=<v>
resolve_from_our_config() {
  local cfg="$1" alias="$2"
  awk -v A="$alias" '
    function trim(s){ sub(/^[ \t\r\n]+/,"",s); sub(/[ \t\r\n]+$/,"",s); return s }

    # Safe glob -> regex (no bracket classes that break older awk)
    function glob2re(s,    out,i,c,specials){
      out=""; specials=".+^$()[]{}|\\"
      for(i=1;i<=length(s);i++){
        c=substr(s,i,1)
        if(c=="*")        out=out ".*"
        else if(c=="?")   out=out "."
        else if(index(specials,c)>0) out=out "\\" c
        else              out=out c
      }
      return "^" out "$"
    }

    function token_match(alias, pat){
      if (substr(pat,1,1)=="!") {
        pat=substr(pat,2)
        return ( alias ~ glob2re(pat) ) ? -1 : 0
      } else {
        return ( alias ~ glob2re(pat) ) ? 1 : 0
      }
    }
    function entry_matches(alias, e,    j,res,pos,neg){
      pos=0; neg=0
      for(j=1;j<=np[e];j++){
        res = token_match(alias, pat[e,j])
        if(res==1) pos=1
        else if(res==-1) neg=1
      }
      return (pos && !neg)
    }

    BEGIN{
      entry=0; inmatch=0
    }
    # Ignore Match blocks entirely (too dynamic)
    tolower($1)=="match" { inmatch=1; next }
    inmatch && tolower($1)=="host" { inmatch=0 }

    tolower($1)=="host" {
      entry++
      np[entry]=0
      for(i=2;i<=NF;i++){ p=$i; if(p!=""){ np[entry]++; pat[entry,np[entry]]=p } }
      u[entry]=""; k[entry]=""; h[entry]=""; prt[entry]=""
      next
    }
    entry>0 && tolower($1)=="user"          { $1=""; u[entry]=trim($0); next }
    entry>0 && tolower($1)=="identityfile"  { $1=""; k[entry]=trim($0); next }
    entry>0 && tolower($1)=="hostname"      { $1=""; h[entry]=trim($0); next }
    entry>0 && tolower($1)=="port"          { $1=""; prt[entry]=trim($0); next }

    END{
      effu=""; effk=""; effh=""; effp=""
      # last-wins merge (common ssh_config practice: put Host * first, specific later)
      for(e=1; e<=entry; e++){
        if (entry_matches(A, e)) {
          if(u[e]  != "") effu=u[e]
          if(k[e]  != "") effk=k[e]
          if(h[e]  != "") effh=h[e]
          if(prt[e]!= "") effp=prt[e]
        }
      }
      if(effp=="") effp="22"
      if(effh=="") effh=A

      print "hostname=" effh
      print "user=" effu
      print "identityfile=" effk
      print "port=" effp
    }
  ' "$cfg"
}

# Determine if an alias is defined in the primary SSH_CONFIG (not only in Included files)
alias_in_main_config() {
  local alias="$1"
  awk -v a="$alias" '
    tolower($1)=="host" {
      for (i=2;i<=NF;i++) if ($i==a) {print NR; exit}
    }
  ' "$SSH_CONFIG" | grep -q '^[0-9]\+$'
}

# Validate host (IPv4 or hostname)
validate_host() {
  local h=$1
  if [[ ! "$h" =~ ^(([0-9]{1,3}\.){3}[0-9]{1,3}|[A-Za-z0-9.-]+)$ ]]; then
    echo -e "${RED}Error: Invalid host/IP: $h${NC}"
    exit 1
  fi
}

validate_key() {
  local key=$1
  if [[ ! -f "$key" ]]; then
    echo -e "${RED}Error: SSH key not found at $key${NC}"
    exit 1
  fi
  local perms
  perms=$(stat -c "%a" "$key" 2>/dev/null || echo "")
  if [[ "$perms" != "600" && -n "$perms" ]]; then
    echo -e "${YELLOW}Warning: SSH key should be 600. Current: $perms${NC}"
    read -rp "$(echo -e ${YELLOW}Fix permissions? [y/N]:${NC}) " fix
    [[ "$fix" =~ ^[Yy]$ ]] && chmod 600 "$key"
  fi
}

#=== COMMANDS ===#
list_aliases() {
  if [[ ! -f "$SSH_CONFIG" ]]; then
    echo -e "${RED}No SSH config found at $SSH_CONFIG${NC}"
    exit 1
  fi

  local EXPANDED; EXPANDED="$(expand_config "$SSH_CONFIG")"
  local HEADER_COLOR="\033[1m"
  local RESET="\033[0m"
  local idx=1

  printf "${HEADER_COLOR}┌────┬──────────────────────────┬─────────────────┬──────────────────┬────────────────────────────────┬──────┐${RESET}\n"
  local key_header="KEY_PATH"; while (( ${#key_header} < 30 )); do key_header="${key_header} "; done
  printf "${HEADER_COLOR}│ %-2s │ %-24s │ %-15s │ %-16s │ %s │ %-4s │${RESET}\n" "ID" "ALIAS" "HOST" "USER" "$key_header" "PORT"
  printf "${HEADER_COLOR}├────┼──────────────────────────┼─────────────────┼──────────────────┼────────────────────────────────┼──────┤${RESET}\n"

  mapfile -t ALIASES < <(collect_aliases "$EXPANDED")

  if (( ${#ALIASES[@]} == 0 )); then
    echo -e "${YELLOW}No non-glob aliases found (only wildcard patterns?).${NC}"
  fi

  for a in "${ALIASES[@]}"; do
    local H=""; local U=""; local K=""; local P=""
    while IFS='=' read -r k v; do
      case "$k" in
        hostname) H="$v" ;;
        user)     U="$v" ;;
        identityfile) K="$v" ;;
        port)     P="$v" ;;
      esac
    done < <(resolve_from_our_config "$EXPANDED" "$a")

    [[ -z "$U" ]] && U="$DEFAULT_USER"
    [[ -z "$K" ]] && K="$DEFAULT_KEY"
    [[ -z "$P" ]] && P="22"
    [[ -z "$H" ]] && H="$a"

    local alias_disp="$a" user_disp="$U" key_disp="$K"
    (( ${#alias_disp} > 24 )) && alias_disp="${alias_disp:0:22}…"
    (( ${#user_disp}  > 16 )) && user_disp="${user_disp:0:14}…"
    (( ${#key_disp}   > 30 )) && key_disp="${key_disp:0:28}…"

    printf "│${GREEN} %-2d ${NC}│ %-24s │ %-15s │ %-16s │ %-30s │ %-4s │\n" \
      "$idx" "$alias_disp" "$H" "$user_disp" "$key_disp" "$P"
    ((idx++))
  done

  printf "${HEADER_COLOR}└────┴──────────────────────────┴─────────────────┴──────────────────┴────────────────────────────────┴──────┘${RESET}\n"

  rm -f "$EXPANDED"
}

get_alias_by_index() {
  local idx="${1:-0}"
  local EXPANDED; EXPANDED="$(expand_config "$SSH_CONFIG")"
  mapfile -t ALIASES < <(collect_aliases "$EXPANDED")
  rm -f "$EXPANDED"
  (( idx >= 1 && idx <= ${#ALIASES[@]} )) || { echo ""; return; }
  echo "${ALIASES[$((idx-1))]}"
}

ssh_by_index() {
  if [[ $# -ne 1 ]]; then
    echo -e "${RED}Usage: sam ssh <index>${NC}"
    exit 1
  fi
  local idx="$1"
  if ! [[ "$idx" =~ ^[0-9]+$ ]] || [[ "$idx" -le 0 ]]; then
    echo -e "${RED}Error: Index must be a positive integer${NC}"
    exit 1
  fi
  local alias
  alias=$(get_alias_by_index "$idx")
  if [[ -z "$alias" ]]; then
    echo -e "${RED}No alias at index $idx${NC}"
    exit 1
  fi
  echo -e "${GREEN}Connecting to $alias ...${NC}"
  exec ssh "$alias"
}

add_alias() {
  backup_config
  local HOST ALIAS USER KEY_PATH opt
  USER="$DEFAULT_USER"; KEY_PATH="$DEFAULT_KEY"; HOST=""; ALIAS=""
  OPTIND=1

  while getopts ":h:u:i:a:" opt; do
    case "$opt" in
      h) HOST="$OPTARG" ;;
      u) USER="$OPTARG" ;;
      i) KEY_PATH="$OPTARG" ;;
      a) ALIAS="$OPTARG" ;;
      *) echo -e "${RED}Invalid option -$OPTARG${NC}"; exit 1 ;;
    esac
  done
  shift $((OPTIND-1))

  [[ -z "$HOST" ]] && read -rp "Enter Host/IP (required): " HOST
  read -rp "Enter username [${DEFAULT_USER}]: " USER
  USER="${USER:-$DEFAULT_USER}"
  read -rp "Enter SSH key path [${DEFAULT_KEY}]: " KEY_PATH
  KEY_PATH="${KEY_PATH:-$DEFAULT_KEY}"
  [[ -z "$ALIAS" ]] && read -rp "Enter alias (required): " ALIAS
  [[ -z "$HOST" || -z "$ALIAS" ]] && echo -e "${RED}Host and alias are required.${NC}" && exit 1

  validate_host "$HOST"
  validate_key "$KEY_PATH"

  if grep -qE "^Host[[:space:]]+$ALIAS$" "$SSH_CONFIG"; then
    echo -e "${RED}Alias $ALIAS already exists in $SSH_CONFIG${NC}"
    exit 1
  fi

  {
    echo
    echo "Host $ALIAS"
    echo "  HostName $HOST"
    echo "  User $USER"
    echo "  IdentityFile $KEY_PATH"
  } >> "$SSH_CONFIG"

  echo -e "${GREEN}Added alias '$ALIAS' for $USER@$HOST${NC}"
}

remove_alias() {
  local ALIAS=""
  OPTIND=1

  if [[ $# -gt 0 && "$1" =~ ^[0-9]+$ ]]; then
    ALIAS=$(get_alias_by_index "$1")
    if [[ -z "$ALIAS" ]]; then
      echo -e "${RED}No alias at index $1${NC}"; exit 1
    fi
    shift
  else
    while getopts ":a:" opt; do
      case "$opt" in a) ALIAS="$OPTARG" ;; *) exit 1 ;; esac
    done
    shift $((OPTIND-1))
    if [[ -z "$ALIAS" ]]; then
      read -rp "Enter alias to remove (index or name): " ALIAS
    fi
    if [[ "$ALIAS" =~ ^[0-9]+$ ]]; then
      local tmp_alias
      tmp_alias=$(get_alias_by_index "$ALIAS")
      [[ -z "$tmp_alias" ]] && { echo -e "${RED}No alias at that index${NC}"; exit 1; }
      ALIAS="$tmp_alias"
    fi
  fi

  if ! alias_in_main_config "$ALIAS"; then
    echo -e "${RED}Refusing to modify alias '$ALIAS' because it is defined in an Included file.${NC}"
    echo -e "${YELLOW}Edit/remove it in its original file, or create an overriding block in $SSH_CONFIG.${NC}"
    exit 1
  fi

  backup_config
  local tmp; tmp=$(mktemp)
  awk -v target="$ALIAS" '
    BEGIN {skip=0}
    tolower($1)=="host" {
      n=split($0, toks, /[ \t]+/)
      count=0
      for (i=2;i<=n;i++) if (toks[i]!=target) kept[++count]=toks[i]
      if (count==0) { skip=1; next }
      else {
        printf "Host"
        for (i=1;i<=count;i++) printf " %s", kept[i]
        printf "\n"
        next
      }
    }
    skip && tolower($1)=="host" { skip=0 }
    !skip { print }
  ' "$SSH_CONFIG" > "$tmp"
  mv "$tmp" "$SSH_CONFIG"
  echo -e "${GREEN}Removed alias '$ALIAS'${NC}"
}

edit_alias() {
  local OLD_ALIAS ALIAS HOST USER KEY_PATH
  USER="$DEFAULT_USER"; KEY_PATH="$DEFAULT_KEY"; HOST=""
  OPTIND=1

  if [[ $# -gt 0 && "$1" =~ ^[0-9]+$ ]]; then
    OLD_ALIAS=$(get_alias_by_index "$1")
    [[ -z "$OLD_ALIAS" ]] && { echo -e "${RED}No alias at index $1${NC}"; exit 1; }
    shift
  else
    local opt
    while getopts ":a:" opt; do
      case "$opt" in a) OLD_ALIAS="$OPTARG" ;; *) exit 1 ;; esac
    done
    shift $((OPTIND-1))
    if [[ -z "$OLD_ALIAS" ]]; then
      read -rp "Enter alias to edit (index or name): " OLD_ALIAS
    fi
    if [[ "$OLD_ALIAS" =~ ^[0-9]+$ ]]; then
      local tmp_alias
      tmp_alias=$(get_alias_by_index "$OLD_ALIAS")
      [[ -z "$tmp_alias" ]] && { echo -e "${RED}No alias at that index${NC}"; exit 1; }
      OLD_ALIAS="$tmp_alias"
    fi
  fi

  local EXPANDED; EXPANDED="$(expand_config "$SSH_CONFIG")"
  local OUT; OUT="$(resolve_from_our_config "$EXPANDED" "$OLD_ALIAS")"
  rm -f "$EXPANDED"
  local eff_host="" eff_user="" eff_key="" eff_port=""
  while IFS='=' read -r k v; do
    case "$k" in
      hostname) eff_host="$v";;
      user)     eff_user="$v";;
      identityfile) eff_key="$v";;
      port)     eff_port="$v";;
    esac
  done <<<"$OUT"
  [[ -z "$eff_user" ]] && eff_user="$DEFAULT_USER"
  [[ -z "$eff_key"  ]] && eff_key="$DEFAULT_KEY"

  echo -e "${YELLOW}Current effective config for '${OLD_ALIAS}':${NC}"
  echo "  HostName: ${eff_host:-unknown}"
  echo "  User    : ${eff_user}"
  echo "  Key     : ${eff_key}"
  echo "  Port    : ${eff_port:-22}"

  read -rp "New alias name [${OLD_ALIAS}]: " ALIAS; ALIAS="${ALIAS:-$OLD_ALIAS}"
  read -rp "New Host (IP or DNS) [${eff_host:-required}]: " HOST; [[ -z "$HOST" ]] && HOST="${eff_host:-}"
  read -rp "Username [${eff_user}]: " USER; USER="${USER:-$eff_user}"
  read -rp "Key Path [${eff_key}]: " KEY_PATH; KEY_PATH="${KEY_PATH:-$eff_key}"

  [[ -z "$HOST" ]] && { echo -e "${RED}Host is required.${NC}"; exit 1; }
  validate_host "$HOST"
  validate_key "$KEY_PATH"

  if ! alias_in_main_config "$OLD_ALIAS"; then
    echo -e "${YELLOW}Alias '$OLD_ALIAS' is defined in an Included file.${NC}"
    echo -e "${YELLOW}Will NOT modify the included file. Appending an overriding Host block in $SSH_CONFIG.${NC}"
    backup_config
    {
      echo
      echo "Host $ALIAS"
      echo "  HostName $HOST"
      echo "  User $USER"
      echo "  IdentityFile $KEY_PATH"
    } >> "$SSH_CONFIG"
    echo -e "${GREEN}Appended override for '$ALIAS' in $SSH_CONFIG${NC}"
    return
  fi

  backup_config
  local tmp; tmp=$(mktemp)
  awk -v target="$OLD_ALIAS" '
    BEGIN {skip=0}
    tolower($1)=="host" {
      n=split($0, toks, /[ \t]+/)
      count=0
      for (i=2;i<=n;i++) if (toks[i]!=target) kept[++count]=toks[i]
      if (count==0) { skip=1; next }
      else {
        printf "Host"
        for (i=1;i<=count;i++) printf " %s", kept[i]
        printf "\n"
        next
      }
    }
    skip && tolower($1)=="host" { skip=0 }
    !skip { print }
  ' "$SSH_CONFIG" > "$tmp"
  mv "$tmp" "$SSH_CONFIG"

  {
    echo
    echo "Host $ALIAS"
    echo "  HostName $HOST"
    echo "  User $USER"
    echo "  IdentityFile $KEY_PATH"
  } >> "$SSH_CONFIG"

  echo -e "${GREEN}Updated alias '$OLD_ALIAS' -> '$ALIAS'${NC}"
}

#=== MAIN ===#
[[ $# -eq 0 ]] && print_help && exit 0

case "${1:-}" in
  -h|--help|help) print_help ;;
  -ls|--list|ls) list_aliases ;;
  --add|add) shift; add_alias "$@" ;;
  --remove|remove) shift; remove_alias "$@" ;;
  --edit|edit) shift; edit_alias "$@" ;;
  ssh) shift; ssh_by_index "$@" ;;
  *)
    echo -e "${RED}Unknown command: $1${NC}"
    print_help
    exit 1
    ;;
esac
