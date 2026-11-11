#!/usr/bin/env bash
# Installe / désinstalle des applications sur Chromebook (terminal Linux / Crostini)
# et installe le certificat RIDF_CA. Pas de Wi-Fi dans ce script.
#
# Usage (terminal Linux) :
#   bash install-apps.sh                     menu interactif (catégories dépliables)
#   bash install-apps.sh firefox vscode      installation de ces apps
#   bash install-apps.sh --min | --all       config minimum / toutes les apps
#   bash install-apps.sh --list              liste les apps disponibles
#   bash install-apps.sh --uninstall [-y]    désinstallation (apps, dépôts ajoutés, certificat)
#   bash install-apps.sh --download          phase 1 seule : télécharge tout dans le cache local
#   bash install-apps.sh --install-only      phase 2 seule : installe depuis le cache (sans réseau)
# Par défaut : phase 1 (téléchargement) puis phase 2 (installation depuis le cache local).
# Options : --no-cert  --keep-cache (garde le cache après installation)  --debug  -y/--yes
#
# Cache : /tmp/install-apps-cache ; s'il n'y a pas assez de place (estimée avant le téléchargement),
#         dossier install-apps-cache à côté du script, supprimé après l'installation.
# Journal : écrit dans /tmp ; en cas d'erreur, déplacé à côté du script (install-apps-DATE.log).
set -euo pipefail

# ====== CONFIG ================================================================
DEBUG=0
SILENT_APPS=""                      # ex. "firefox vscode thonny" pour installer sans menu
INSTALL_CERT=1
CERT_HOST="dl.20.doume.be/lycee/RIDF_CA.crt"   # cherché d'abord à côté du script (RIDF_CA.crt)
MIN_APPS=(wordcraft gridcraft deckcraft pdfcraft firefox brave vlc 7zip)
ORG="storytold"
export TMPDIR=/tmp                 # tous les fichiers temporaires vont dans /tmp
CACHE="${INSTALL_APPS_CACHE:-/tmp/install-apps-cache}"   # fichiers téléchargés (phase 1 -> phase 2)
APT_CACHE="$CACHE/apt"

# clé|Titre affiché
CATEGORIES=(
  "bureau|Bureautique"
  "creation|Création (images, audio, vidéo, CAO)"
  "web|Navigateurs"
  "dev|Programmation"
  "sciences|Maths & sciences"
  "secu|Réseau & cybersécurité"
  "outils|Utilitaires"
)

# id|catégorie|description|méthode|argument
#   apt    -> paquets Debian (argument = liste de paquets)
#   craft  -> .deb de la dernière release GitHub $ORG/<id>
#   custom -> fonction inst_<id> (dépôt externe)
CATALOG=(
  "wordcraft|bureau|Traitement de texte (type Word)|craft|"
  "gridcraft|bureau|Tableur (type Excel)|craft|"
  "deckcraft|bureau|Présentations (type PowerPoint)|craft|"
  "pdfcraft|bureau|Lecture et édition de PDF|craft|"
  "libreoffice|bureau|Suite bureautique complète|apt|libreoffice libreoffice-l10n-fr"
  "texstudio|bureau|Éditeur LaTeX + distribution TeX Live|apt|texstudio texlive-latex-recommended texlive-lang-french"

  "photocraft|creation|Retouche photo par calques (type Photoshop)|craft|"
  "vectorcraft|creation|Dessin vectoriel (type Illustrator)|craft|"
  "filmcraft|creation|Montage vidéo (type Premiere Pro)|craft|"
  "lightcraft|creation|Développement de photos RAW (type Lightroom)|craft|"
  "effectcraft|creation|Effets visuels / animation|craft|"
  "designcraft|creation|Mise en page / design graphique|craft|"
  "figcraft|creation|Maquettes d'interfaces (type Figma)|craft|"
  "soundcraft|creation|Enregistrement et édition audio|craft|"
  "cadcraft|creation|Dessin technique 2D/3D (type AutoCAD)|craft|"
  "gimp|creation|GIMP, retouche d'images|apt|gimp"
  "inkscape|creation|Inkscape, dessin vectoriel|apt|inkscape"
  "krita|creation|Krita, peinture numérique|apt|krita"
  "audacity|creation|Audacity, édition audio|apt|audacity"
  "kdenlive|creation|Kdenlive, montage vidéo|apt|kdenlive"

  "firefox|web|Firefox ESR (français)|apt|firefox-esr firefox-esr-l10n-fr"
  "brave|web|Brave (dépôt officiel)|custom|brave-browser"
  "librewolf|web|LibreWolf (Firefox durci, via extrepo)|custom|librewolf"
  "chromium|web|Chromium|apt|chromium chromium-l10n"

  "vscode|dev|Visual Studio Code (dépôt Microsoft)|custom|code"
  "thonny|dev|Thonny, éditeur Python pour débuter (équivalent EduPython)|apt|thonny"
  "idle|dev|IDLE, éditeur Python officiel|apt|idle"
  "spyder|dev|Spyder, environnement Python scientifique|apt|spyder"
  "python|dev|Python 3 + pip + venv|apt|python3 python3-pip python3-venv"
  "git|dev|Git|apt|git"
  "build-essential|dev|Compilateurs C/C++ et make|apt|build-essential gdb"
  "nodejs|dev|Node.js + npm|apt|nodejs npm"
  "java|dev|JDK Java|apt|default-jdk"
  "sqlitebrowser|dev|DB Browser for SQLite (cours de bases de données)|apt|sqlitebrowser sqlite3"
  "geany|dev|Geany, éditeur léger|apt|geany"

  "pysci|sciences|NumPy, SciPy, Matplotlib, SymPy, Pandas, Jupyter|apt|python3-numpy python3-scipy python3-matplotlib python3-sympy python3-pandas jupyter-notebook"
  "octave|sciences|GNU Octave (calcul numérique, type Matlab)|apt|octave"
  "scilab|sciences|Scilab|apt|scilab"
  "wxmaxima|sciences|wxMaxima (calcul formel)|apt|wxmaxima"

  "nmap|secu|Nmap, scanner réseau|apt|nmap"
  "wireshark|secu|Wireshark, analyse de paquets|apt|wireshark"
  "tcpdump|secu|tcpdump + netcat|apt|tcpdump netcat-openbsd"
  "john|secu|John the Ripper (audit de mots de passe)|apt|john"

  "vlc|outils|Lecteur multimédia VLC|apt|vlc"
  "keepassxc|outils|Gestionnaire de mots de passe|apt|keepassxc"
  "7zip|outils|Archives 7z/zip/rar|apt|p7zip-full unzip"
  "htop|outils|Moniteur de processus|apt|htop"
)

# --- Découpage du catalogue ---------------------------------------------------
IDS=(); CATS=(); DESCS=(); METHS=(); PKGARGS=()
for _e in "${CATALOG[@]}"; do
  IFS='|' read -r _a _b _c _d _f <<< "$_e"
  IDS+=("$_a"); CATS+=("$_b"); DESCS+=("$_c"); METHS+=("$_d"); PKGARGS+=("${_f:-}")
done
N=${#IDS[@]}

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
# Journal : écrit dans /tmp ; en cas d'erreur il est déplacé à côté du script (FINAL_LOG)
LOG_FILE="/tmp/install-apps-$(id -un).log"
if [ -w "$SCRIPT_DIR" ]; then FINAL_LOG="$SCRIPT_DIR/install-apps-$(date +%Y%m%d-%H%M%S).log"; else FINAL_LOG="$LOG_FILE"; fi
STATE_DIR="${XDG_STATE_HOME:-$HOME/.local/state}/install-apps"
STATE_FILE="$STATE_DIR/installed.txt"
CERT_PEM="/usr/local/share/ca-certificates/RIDF_CA.crt"
REPO_FILES=(
  /etc/apt/sources.list.d/brave-browser-release.list
  /usr/share/keyrings/brave-browser-archive-keyring.gpg
  /etc/apt/sources.list.d/vscode.list
  /usr/share/keyrings/microsoft.gpg
)

# --- Arguments ----------------------------------------------------------------
MODE="install"; UNINSTALL_PICKED=0; ASSUME_YES=0; SEL_ALL=0; SEL_MIN=0; ARGS_APPS=()
DL_ONLY=0; LOCAL_ONLY=0; KEEP_CACHE=0
usage() { sed -n '2,/^set -/p' "${BASH_SOURCE[0]}" | sed '$d' | sed 's/^# \{0,1\}//'; }
for a in "$@"; do
  case "$a" in
    --debug) DEBUG=1 ;;
    -y|--yes) ASSUME_YES=1 ;;
    --no-cert) INSTALL_CERT=0 ;;
    --download) DL_ONLY=1 ;;
    --install-only) LOCAL_ONLY=1 ;;
    --keep-cache) KEEP_CACHE=1 ;;
    --uninstall) MODE="uninstall" ;;
    --list) MODE="list" ;;
    --all) SEL_ALL=1 ;;
    --min) SEL_MIN=1 ;;
    -h|--help) usage; exit 0 ;;
    -*) echo "Option inconnue : $a (voir --help)" >&2; exit 2 ;;
    *) ARGS_APPS+=("$a") ;;
  esac
done

if [ "$MODE" = "list" ]; then
  for c in "${CATEGORIES[@]}"; do
    echo -e "\e[1m${c#*|}\e[0m"
    for ((i=0;i<N;i++)); do
      [ "${CATS[i]}" = "${c%%|*}" ] && printf '  %-16s %s\n' "${IDS[i]}" "${DESCS[i]}"
    done
  done
  exit 0
fi

# --- Affichage : journal, messages, barre de progression ----------------------
if [ -t 1 ] && [ "$DEBUG" != 1 ]; then FANCY=1; else FANCY=0; fi
STEP_TOTAL=0; STEP_DONE=0; BAR_ON=0; LAST_LABEL=""; FAILED=()

log() {
  printf '[%s] %s\n' "$(date +%T)" "$*" >> "$LOG_FILE" 2>/dev/null || true
  if [ "$DEBUG" = 1 ]; then printf '\e[90m[debug] %s\e[0m\n' "$*" >&2; fi
}
clearline() { if [ "$FANCY" = 1 ] && [ "$BAR_ON" = 1 ]; then printf '\r\e[2K'; fi; }
bar() {
  LAST_LABEL="$1"; BAR_ON=1
  if [ "$FANCY" != 1 ]; then echo "[$STEP_DONE/$STEP_TOTAL] $1"; return 0; fi
  local w=28 pct=100 f i filled="" empty=""
  if [ "$STEP_TOTAL" -gt 0 ]; then pct=$(( STEP_DONE * 100 / STEP_TOTAL )); fi
  f=$(( pct * w / 100 ))
  for ((i=0;i<w;i++)); do if [ "$i" -lt "$f" ]; then filled+="█"; else empty+="░"; fi; done
  printf '\r\e[2K[\e[32m%s\e[0m%s] %3d%%  %s' "$filled" "$empty" "$pct" "$1"
}
redraw() { if [ "$FANCY" = 1 ] && [ "$BAR_ON" = 1 ]; then bar "$LAST_LABEL"; fi; }
tick() { STEP_DONE=$((STEP_DONE+1)); redraw; }
bar_end() { if [ "$FANCY" = 1 ] && [ "$BAR_ON" = 1 ]; then echo; fi; BAR_ON=0; }
msg() { clearline; echo -e "$*"; redraw; }
err() { clearline; echo -e "\e[31m$*\e[0m" >&2; log "ERREUR: $*"; redraw; }

# run cmd... : sortie -> journal ; erreurs affichées seulement en cas d'échec
run() {
  local rc=0 errf w; errf="$(mktemp)"
  log "RUN: $*"
  if [ "$DEBUG" = 1 ]; then
    set +e; "$@" 2>&1 | tee -a "$LOG_FILE"; rc=${PIPESTATUS[0]}; set -e
  else
    "$@" >>"$LOG_FILE" 2>"$errf" || rc=$?
    cat "$errf" >> "$LOG_FILE" 2>/dev/null || true
    if [ "$rc" -ne 0 ]; then
      err "Erreur (code $rc) : $*"
      if [ -s "$errf" ]; then clearline; sed 's/^/    /' "$errf" >&2; redraw; fi
    else
      w="$(grep -E '^(E|W):' "$errf" || true)"
      if [ -n "$w" ]; then clearline; echo "$w" | sed 's/^/    /' >&2; redraw; fi
    fi
  fi
  rm -f "$errf"; log "RC=$rc"
  return "$rc"
}

if [ "$(id -u)" -eq 0 ]; then SUDO=""; else SUDO="sudo"; fi
aptq() { run $SUDO env DEBIAN_FRONTEND=noninteractive apt-get -y -qq -o Dpkg::Use-Pty=0 "$@"; }
pkg_installed() { dpkg-query -W -f='${Status}' "$1" 2>/dev/null | grep -q "install ok installed"; }
confirm() {
  [ "$ASSUME_YES" = 1 ] && return 0
  [ -t 0 ] || { echo "Confirmation impossible sans terminal : ajoutez --yes." >&2; return 1; }
  local ans; read -r -p "$1 [o/N] " ans
  case "$ans" in o|O|y|Y|oui|yes) return 0 ;; *) return 1 ;; esac
}
record_pkgs() { # mémorise les paquets absents avant l'installation (pour la désinstallation)
  local p; mkdir -p "$STATE_DIR"; touch "$STATE_FILE"
  for p in "$@"; do
    if ! pkg_installed "$p" && ! grep -qx "$p" "$STATE_FILE"; then echo "$p" >> "$STATE_FILE"; fi
  done
}

# --- Vérification de l'environnement ------------------------------------------
if grep -q '^CHROMEOS_RELEASE' /etc/lsb-release 2>/dev/null; then
  echo "Vous êtes dans le shell ChromeOS. Lancez ce script dans le terminal Linux (Debian)." >&2; exit 1
fi
command -v apt-get >/dev/null 2>&1 || { echo "apt-get introuvable : script prévu pour Debian/Ubuntu (Crostini)." >&2; exit 1; }

# --- Écran alternatif : le script s'affiche à part, le terminal retrouve son état à la sortie ---
WORK=""; EXIT_MSG=""; ALT=0
cleanup() {
  local rc=$?
  if [ -n "$WORK" ]; then rm -rf "$WORK" || true; fi
  if { [ ${#FAILED[@]} -gt 0 ] || [ "$rc" -ne 0 ]; } && [ "$FINAL_LOG" != "$LOG_FILE" ] && [ -f "$LOG_FILE" ]; then
    mv -f "$LOG_FILE" "$FINAL_LOG" 2>/dev/null || true
    if [ -z "$EXIT_MSG" ]; then EXIT_MSG="Une erreur est survenue (code $rc). Journal : $FINAL_LOG"; fi
  fi
  if [ "$ALT" = 1 ]; then printf '\e[?25h\e[?1049l'; fi
  if [ -n "$EXIT_MSG" ]; then echo "$EXIT_MSG" >&2; fi
}
pause_end() { # garde l'écran du script jusqu'à une touche, sinon il disparaîtrait aussitôt
  if [ "$ALT" = 1 ] && [ -t 0 ]; then echo; read -rsn1 -p "Appuyez sur une touche pour quitter… " || true; fi
}
trap cleanup EXIT
trap 'exit 130' INT TERM
if [ -t 1 ] && [ "$DEBUG" != 1 ]; then ALT=1; printf '\e[?1049h\e[H'; fi

{
  echo "================ $(date '+%F %T') ================"
  echo "args: $* | mode=$MODE debug=$DEBUG user=$(id -un) uid=$(id -u) bash=$BASH_VERSION"
} >> "$LOG_FILE" 2>/dev/null || true

# --- Menu à catégories dépliables ----------------------------------------------
# ↑/↓ déplacer · →/← déplier/replier · Espace cocher · a tout/rien · Entrée valider · q quitter
menu() {
  local nc=${#CATEGORIES[@]} n r t k key v ci
  local cur=0 open=-1 prev=0 CS=0 CT=0
  local -a sel=() RT=() RI=()
  for ((n=0;n<N;n++)); do sel[n]=0; done

  is_min() { local m; for m in "${MIN_APPS[@]}"; do [ "$m" = "$1" ] && return 0; done; return 1; }
  min_on() {
    local x
    for ((x=0;x<N;x++)); do
      if is_min "${IDS[x]}" && [ "${sel[x]}" -eq 0 ]; then return 1; fi
    done
    return 0
  }
  cat_stat() { # cat_stat index_catégorie -> CS (cochées) CT (total)
    local ck="${CATEGORIES[$1]%%|*}" x; CS=0; CT=0
    for ((x=0;x<N;x++)); do
      if [ "${CATS[x]}" = "$ck" ]; then CT=$((CT+1)); CS=$((CS+sel[x])); fi
    done
  }
  build() {
    local c x; RT=(m); RI=(0)
    for ((c=0;c<nc;c++)); do
      RT+=(c); RI+=("$c")
      if [ "$open" -eq "$c" ]; then
        for ((x=0;x<N;x++)); do
          if [ "${CATS[x]}" = "${CATEGORIES[c]%%|*}" ]; then RT+=(a); RI+=("$x"); fi
        done
      fi
    done
    RT+=(u); RI+=(0)
  }
  draw() {
    local rows mark line arrow
    build; rows=${#RT[@]}
    if [ "$cur" -ge "$rows" ]; then cur=$((rows-1)); fi
    if [ "$prev" -gt 0 ]; then printf '\e[%dA' "$prev"; fi
    for ((r=0;r<rows;r++)); do
      k=${RI[r]}
      case "${RT[r]}" in
        m) if min_on; then mark=x; else mark=" "; fi
           line="[$mark] \e[1mConfig minimum\e[0m \e[36m(${MIN_APPS[*]})\e[0m" ;;
        c) cat_stat "$k"
           if [ "$CS" -eq 0 ]; then mark=" "; elif [ "$CS" -eq "$CT" ]; then mark=x; else mark=-; fi
           if [ "$open" -eq "$k" ]; then arrow="▼"; else arrow="▶"; fi
           line="[$mark] $arrow \e[1m${CATEGORIES[k]#*|}\e[0m \e[36m($CS/$CT)\e[0m" ;;
        a) if [ "${sel[k]}" -eq 1 ]; then mark=x; else mark=" "; fi
           line="      [$mark] $(printf '%-16s' "${IDS[k]}") \e[36m${DESCS[k]}\e[0m" ;;
        u) if [ "$UNINSTALL_PICKED" -eq 1 ]; then mark=x; else mark=" "; fi
           line="[$mark] \e[1;31mDésinstaller tout\e[0m \e[36m(apps, dépôts ajoutés, certificat ; remplace l'installation)\e[0m" ;;
      esac
      if [ "$r" -eq "$cur" ]; then printf '\e[2K\e[7m>\e[0m %b\n' "$line"; else printf '\e[2K  %b\n' "$line"; fi
    done
    printf '\e[J'; prev=$rows
  }

  echo "Applications  (↑/↓ déplacer · →/← déplier · Espace cocher · a tout/rien · Entrée valider · q quitter)"
  printf '\e[?25l'
  trap 'printf "\e[?25h"' RETURN
  build; draw
  while true; do
    IFS= read -rsn1 key || break
    t=${RT[cur]}; k=${RI[cur]}
    case "$key" in
      $'\e') read -rsn2 -t 0.1 key || true
             case "$key" in
               '[A') cur=$(( (cur-1+${#RT[@]})%${#RT[@]} )) ;;
               '[B') cur=$(( (cur+1)%${#RT[@]} )) ;;
               '[C') if [ "$t" = c ]; then open=$k; fi ;;
               '[D') if [ "$t" = c ]; then
                       if [ "$open" -eq "$k" ]; then open=-1; fi
                     elif [ "$t" = a ]; then
                       for ((ci=0;ci<nc;ci++)); do [ "${CATEGORIES[ci]%%|*}" = "${CATS[k]}" ] && break; done
                       open=-1; build
                       for ((r=0;r<${#RT[@]};r++)); do
                         if [ "${RT[r]}" = c ] && [ "${RI[r]}" -eq "$ci" ]; then cur=$r; break; fi
                       done
                     fi ;;
             esac ;;
      ' ')  case "$t" in
              m) v=1; if min_on; then v=0; fi
                 for ((n=0;n<N;n++)); do if is_min "${IDS[n]}"; then sel[n]=$v; fi; done ;;
              c) cat_stat "$k"; v=1; if [ "$CS" -eq "$CT" ]; then v=0; fi
                 for ((n=0;n<N;n++)); do
                   if [ "${CATS[n]}" = "${CATEGORIES[k]%%|*}" ]; then sel[n]=$v; fi
                 done ;;
              a) sel[k]=$((1-sel[k])) ;;
              u) UNINSTALL_PICKED=$((1-UNINSTALL_PICKED)) ;;
            esac ;;
      a)    v=0; for ((n=0;n<N;n++)); do if [ "${sel[n]}" -eq 0 ]; then v=1; fi; done
            for ((n=0;n<N;n++)); do sel[n]=$v; done ;;
      q)    exit 0 ;;
      '')   break ;;
    esac
    draw
  done
  for ((n=0;n<N;n++)); do if [ "${sel[n]}" -eq 1 ]; then APPS+=("${IDS[n]}"); fi; done
  return 0
}

# =============================== Cache local ===================================
# Phase 1 : tout est téléchargé dans $CACHE (paquets apt + dépendances, .deb ArtCraft, certificat).
# Phase 2 : tout est installé depuis $CACHE, sans aucun téléchargement (apt --no-download).
aptc() { run $SUDO env DEBIAN_FRONTEND=noninteractive apt-get -y -qq -o Dpkg::Use-Pty=0 -o Dir::Cache::archives="$APT_CACHE" -o APT::Sandbox::User=root "$@"; }

cache_init() {
  local -a owner=()
  if getent passwd _apt >/dev/null 2>&1; then owner=(-o _apt); fi
  run $SUDO install -d -m 755 "$CACHE" "$CACHE/craft" "$CACHE/cert" "$APT_CACHE" || return 1
  run $SUDO install -d "${owner[@]}" -m 700 "$APT_CACHE/partial"
}

app_idx() { local i; for ((i=0;i<N;i++)); do if [ "${IDS[i]}" = "$1" ]; then echo "$i"; return 0; fi; done; return 1; }
meth_of() { local i; i="$(app_idx "$1")" || return 0; echo "${METHS[i]}"; }
is_bad() { local b; for b in "${BAD[@]}"; do [ "$b" = "$1" ] && return 0; done; return 1; }

# =============================== Certificat ====================================
fetch_cert() { # télécharge (ou copie) le certificat dans le cache
  local src tmp
  tmp="$(mktemp)"
  src="$(find "$SCRIPT_DIR" -maxdepth 1 -iname 'RIDF_CA.crt' -print -quit 2>/dev/null || true)"
  if [ -n "$src" ]; then
    log "certificat trouvé à côté du script : $src"
    cp "$src" "$tmp"
  else
    log "pas de certificat dans $SCRIPT_DIR -> téléchargement"
    if ! command -v curl >/dev/null 2>&1; then aptq update || true; aptq install curl ca-certificates openssl || return 1; fi
    if ! run curl -fsSL -o "$tmp" "https://$CERT_HOST"; then
      log "HTTPS impossible : repli en HTTP"
      run curl -fsSL -o "$tmp" "http://$CERT_HOST" || { rm -f "$tmp"; return 1; }
    fi
  fi
  if ! grep -q "BEGIN CERTIFICATE" "$tmp"; then
    log "format DER détecté -> conversion PEM"
    command -v openssl >/dev/null 2>&1 || aptq install openssl || return 1
    run openssl x509 -inform der -in "$tmp" -out "$tmp.pem" || return 1
    mv "$tmp.pem" "$tmp"
  fi
  if command -v openssl >/dev/null 2>&1; then
    log "Empreinte SHA-256 du certificat : $(openssl x509 -in "$tmp" -noout -fingerprint -sha256 2>/dev/null | cut -d= -f2)"
  fi
  cp "$tmp" "$WORK/RIDF_CA.crt" || { rm -f "$tmp"; return 1; }
  rm -f "$tmp"
}

apply_cert() { # installe le certificat dans le magasin système
  local src="$WORK/RIDF_CA.crt"
  [ -f "$src" ] || src="$CACHE/cert/RIDF_CA.crt"
  [ -f "$src" ] || { err "Certificat introuvable (relancez la phase de téléchargement)"; return 1; }
  run $SUDO install -m 644 "$src" "$CERT_PEM" || return 1
  run $SUDO update-ca-certificates
}

# ===================== Dépôts externes (configurés en phase 1) =================
repo_brave() {
  local k; k="$(mktemp)"
  run curl -fsSL -o "$k" https://brave-browser-apt-release.s3.brave.com/brave-browser-archive-keyring.gpg || return 1
  run $SUDO install -m 644 "$k" /usr/share/keyrings/brave-browser-archive-keyring.gpg || return 1
  echo "deb [signed-by=/usr/share/keyrings/brave-browser-archive-keyring.gpg] https://brave-browser-apt-release.s3.brave.com/ stable main" > "$k"
  run $SUDO install -m 644 "$k" /etc/apt/sources.list.d/brave-browser-release.list || return 1
  rm -f "$k"
}

repo_vscode() {
  local k; k="$(mktemp)"
  run bash -c 'curl -fsSL https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > "$1"' _ "$k" || return 1
  run $SUDO install -m 644 "$k" /usr/share/keyrings/microsoft.gpg || return 1
  echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/microsoft.gpg] https://packages.microsoft.com/repos/code stable main" > "$k"
  run $SUDO install -m 644 "$k" /etc/apt/sources.list.d/vscode.list || return 1
  rm -f "$k"
}

repo_librewolf() {
  record_pkgs extrepo
  aptq install extrepo || return 1
  run $SUDO extrepo enable librewolf
}

# ============================ Phase 1 : téléchargement =========================
fetch_craft() { # .deb ArtCraft (GitHub) -> cache, avec vérification SHA-256
  local app="$1" rel url file sums name expected actual
  rel="$WORK/$app.json"
  if [ ! -s "$rel" ]; then run curl -fsSL -o "$rel" "https://api.github.com/repos/$ORG/$app/releases/latest" || return 1; fi
  url="$(grep -o "https://[^\"]*-linux-$ARCH\.deb" "$rel" | head -n1 || true)"
  if [ -z "$url" ]; then err "$app : aucun .deb linux-$ARCH dans la dernière release"; return 1; fi
  file="$WORK/$(basename "$url")"; name="$(basename "$file")"
  run curl -fsSL -o "$file" "$url" || return 1
  sums="$WORK/$app.SHA256SUMS.txt"
  if run curl -fsSL -o "$sums" "$(dirname "$url")/SHA256SUMS.txt"; then
    expected="$(awk -v f="$name" '{n=$2; sub(/^\*/,"",n); if(n==f) print $1}' "$sums")"
    actual="$(sha256sum "$file" | cut -d' ' -f1)"
    log "sha256 attendu=$expected obtenu=$actual"
    if [ -n "$expected" ] && [ "$expected" != "$actual" ]; then
      err "$app : somme SHA-256 incorrecte, téléchargement rejeté"; return 1
    fi
    [ -n "$expected" ] || msg "   \e[33m$app : absent de SHA256SUMS.txt, vérification sautée\e[0m"
  else
    msg "   \e[33m$app : SHA256SUMS.txt indisponible, vérification sautée\e[0m"
  fi
  run $SUDO install -m 644 "$file" "$CACHE/craft/$app.deb" || return 1
  rm -f "$file"
}

download_app() { # télécharge l'app et toutes ses dépendances dans le cache (sans rien installer)
  local id="$1" idx meth arg
  idx="$(app_idx "$id")" || { err "Application inconnue : $id (voir --list)"; return 1; }
  meth="${METHS[idx]}"; arg="${PKGARGS[idx]}"
  log "téléchargement app=$id méthode=$meth"
  case "$meth" in
    apt|custom) # shellcheck disable=SC2086
                aptc install --download-only $arg ;;
    craft)      fetch_craft "$id" && aptc install --download-only "$CACHE/craft/$id.deb" ;;
    *)          err "Méthode inconnue pour $id : $meth"; return 1 ;;
  esac
}

estimate_kb() { # taille à télécharger (Ko) + marge ; chaque paquet n'est compté qu'une fois
  local list="$WORK/uris.txt" id idx meth arg url sz rel
  : > "$list"
  for id in "${APPS[@]}"; do
    if is_bad "$id"; then continue; fi
    idx="$(app_idx "$id")" || continue
    meth="${METHS[idx]}"; arg="${PKGARGS[idx]}"
    case "$meth" in
      apt|custom) # shellcheck disable=SC2086
                  apt-get -qq -y --print-uris install $arg 2>>"$LOG_FILE" \
                    | awk '$3 ~ /^[0-9]+$/ {print $2, $3}' >> "$list" || true ;;
      craft)      rel="$WORK/$id.json"
                  if [ ! -s "$rel" ]; then curl -fsSL -o "$rel" "https://api.github.com/repos/$ORG/$id/releases/latest" 2>>"$LOG_FILE" || true; fi
                  url="$(grep -o "https://[^\"]*-linux-$ARCH\.deb" "$rel" 2>/dev/null | head -n1 || true)"
                  if [ -n "$url" ]; then
                    sz="$(curl -fsSIL "$url" 2>>"$LOG_FILE" | tr -d '\r' | awk 'tolower($1)=="content-length:"{v=$2} END{print v+0}' || true)"
                    echo "$id.deb ${sz:-0}" >> "$list"
                  fi ;;
    esac
  done
  sort -u "$list" | awk '{b+=$2} END{printf "%d\n", (b*1.15)/1024 + 204800}'
}

avail_kb() { df -Pk "$1" 2>/dev/null | awk 'NR==2{print $4+0}'; }

choose_cache() { # /tmp si assez de place, sinon dossier à côté du script
  local need avail d base
  need="$(estimate_kb)"
  d="$(dirname "$CACHE")"; if [ ! -d "$d" ]; then d="/"; fi
  avail="$(avail_kb "$d")"; avail="${avail:-0}"
  log "espace : ~${need} Ko nécessaires, ${avail} Ko disponibles dans $d"
  if [ "$need" -le "$avail" ]; then return 0; fi
  base="$SCRIPT_DIR"; if [ ! -w "$base" ]; then base="$HOME"; fi
  avail="$(avail_kb "$base")"; avail="${avail:-0}"
  if [ "$need" -gt "$avail" ]; then
    EXIT_MSG="Pas assez de place : ~$((need/1024)) Mo nécessaires, $((avail/1024)) Mo disponibles dans $base."
    exit 1
  fi
  CACHE="$base/install-apps-cache"; APT_CACHE="$CACHE/apt"
  msg "Pas assez de place dans $d : cache dans $CACHE (supprimé après l'installation)"
}

phase_download() {
  local app tmpf repos=0
  READY=(); BAD=()
  msg "\e[1mPhase 1/2 : téléchargement\e[0m"
  STEP_TOTAL=$(( 5 + ${#APPS[@]} )); STEP_DONE=0

  bar "Certificat RIDF_CA"
  if [ "$INSTALL_CERT" = "1" ]; then
    { fetch_cert && apply_cert; } || FAILED+=("certificat")
  fi
  tick

  bar "Mise à jour de la liste des paquets"
  aptq update || FAILED+=("apt update")
  tick

  bar "Outils nécessaires (curl, gnupg)"
  aptq install curl ca-certificates gnupg || FAILED+=("outils")
  tick

  bar "Dépôts externes"
  for app in "${APPS[@]}"; do
    if [ "$(meth_of "$app")" = "custom" ]; then
      if "repo_$app"; then repos=1; else BAD+=("$app"); FAILED+=("$app"); fi
    fi
  done
  if [ "$repos" = 1 ]; then aptq update || FAILED+=("apt update (dépôts)"); fi
  tick

  bar "Calcul de l'espace nécessaire"
  choose_cache
  cache_init || { EXIT_MSG="Impossible de créer $CACHE"; exit 1; }
  if [ -f "$WORK/RIDF_CA.crt" ]; then run $SUDO install -m 644 "$WORK/RIDF_CA.crt" "$CACHE/cert/RIDF_CA.crt" || true; fi
  tick

  for app in "${APPS[@]}"; do
    bar "$app : téléchargement"
    if is_bad "$app"; then :
    elif download_app "$app"; then READY+=("$app")
    else FAILED+=("$app"); fi
    tick
  done
  bar_end

  tmpf="$(mktemp)"
  if [ ${#READY[@]} -gt 0 ]; then printf '%s\n' "${READY[@]}" > "$tmpf"; else : > "$tmpf"; fi
  run $SUDO install -m 644 "$tmpf" "$CACHE/ready.txt" || true
  rm -f "$tmpf"
  log "Cache : $(du -sh "$CACHE" 2>/dev/null | cut -f1) dans $CACHE"
}

# ====================== Phase 2 : installation 100 % locale ====================
install_local() {
  local id="$1" idx meth arg pkg
  idx="$(app_idx "$id")" || { err "Application inconnue : $id"; return 1; }
  meth="${METHS[idx]}"; arg="${PKGARGS[idx]}"
  log "installation locale app=$id méthode=$meth"
  case "$meth" in
    apt|custom) # shellcheck disable=SC2086
                record_pkgs $arg; aptc install --no-download $arg ;;
    craft)      [ -f "$CACHE/craft/$id.deb" ] || { err "$id : .deb absent du cache (relancez la phase de téléchargement)"; return 1; }
                pkg="$(dpkg-deb -f "$CACHE/craft/$id.deb" Package 2>/dev/null || echo "$id")"
                record_pkgs "$pkg"
                # apt refuse un .deb local ici (« Pathname to install is not absolute ») :
                # dpkg pose le paquet, puis apt installe ses dépendances depuis le cache et le configure.
                log "RUN: dpkg -i $CACHE/craft/$id.deb"
                $SUDO env DEBIAN_FRONTEND=noninteractive dpkg -i "$CACHE/craft/$id.deb" >>"$LOG_FILE" 2>&1 || true
                aptc -f install --no-download || return 1
                pkg_installed "$pkg" || { err "$id : le paquet $pkg n'est pas installé (voir le journal)"; return 1; } ;;
    *)          err "Méthode inconnue pour $id : $meth"; return 1 ;;
  esac
}

phase_install() {
  local app
  msg "\n\e[1mPhase 2/2 : installation depuis le cache local (aucun téléchargement)\e[0m"
  STEP_TOTAL=$(( 1 + ${#TO_INSTALL[@]} )); STEP_DONE=0

  bar "Certificat RIDF_CA"
  if [ "$INSTALL_CERT" = "1" ] && [ ! -e "$CERT_PEM" ]; then apply_cert || FAILED+=("certificat"); fi
  tick

  for app in "${TO_INSTALL[@]}"; do
    [ -n "$app" ] || { tick; continue; }
    bar "$app : installation"
    install_local "$app" || FAILED+=("$app")
    tick
  done
  bar_end
}

do_install() {
  case "$(dpkg --print-architecture)" in
    amd64) ARCH="x86_64" ;;
    arm64) ARCH="aarch64" ;;
    *) EXIT_MSG="Architecture non supportée : $(dpkg --print-architecture)"; exit 1 ;;
  esac
  log "architecture: $ARCH"
  WORK="$(mktemp -d)"; chmod 755 "$WORK"

  TO_INSTALL=()
  if [ "$LOCAL_ONLY" != 1 ]; then
    phase_download
    if [ ${#READY[@]} -gt 0 ]; then TO_INSTALL=("${READY[@]}"); fi
  else
    TO_INSTALL=("${APPS[@]}")
  fi
  if [ "$DL_ONLY" != 1 ]; then
    phase_install
    if [ ${#FAILED[@]} -eq 0 ] && [ "$KEEP_CACHE" != 1 ]; then
      run $SUDO rm -rf "$CACHE" || true
    fi
  fi
}

# ================================ Désinstallation ==============================
do_uninstall() {
  local pkgs=() p f
  if [ -f "$STATE_FILE" ]; then
    while IFS= read -r p; do
      if [ -n "$p" ] && pkg_installed "$p"; then pkgs+=("$p"); fi
    done < "$STATE_FILE"
  fi
  echo "Seront supprimés :"
  echo "  - paquets installés par ce script : ${pkgs[*]:-(aucun)}"
  echo "  - dépôts ajoutés (Brave, VS Code, LibreWolf)"
  echo "  - certificat $CERT_PEM"
  echo "  - fichiers d'état de ce script ($STATE_DIR) et cache de téléchargement ($CACHE)"
  echo "Vos documents ne sont pas touchés."
  confirm "Confirmer la désinstallation ?" || exit 0

  STEP_TOTAL=4; STEP_DONE=0
  bar "Suppression des paquets"
  if [ ${#pkgs[@]} -gt 0 ]; then aptq purge "${pkgs[@]}" || FAILED+=("purge"); fi
  tick
  bar "Suppression des dépôts"
  command -v extrepo >/dev/null 2>&1 && { run $SUDO extrepo disable librewolf || true; }
  for f in "${REPO_FILES[@]}"; do
    if [ -e "$f" ]; then run $SUDO rm -f "$f" || FAILED+=("$f"); fi
  done
  aptq update || true
  tick
  bar "Suppression du certificat RIDF_CA"
  if [ -e "$CERT_PEM" ]; then
    run $SUDO rm -f "$CERT_PEM" || FAILED+=("certificat")
    run $SUDO update-ca-certificates --fresh || FAILED+=("update-ca-certificates")
  fi
  tick
  bar "Nettoyage"
  rm -rf "$STATE_DIR"
  run $SUDO rm -rf "$CACHE" "$SCRIPT_DIR/install-apps-cache" "$HOME/install-apps-cache" || true
  tick
  bar_end
}

uninstall_and_exit() {
  do_uninstall
  echo "Désinstallation terminée."
  if [ ${#FAILED[@]} -gt 0 ]; then
    echo "Échecs : ${FAILED[*]}" >&2
    EXIT_MSG="Échecs : ${FAILED[*]} (journal : $FINAL_LOG)"
  fi
  pause_end
  [ ${#FAILED[@]} -eq 0 ] || exit 1
  exit 0
}

# ================================= Programme ===================================
if [ "$MODE" = "uninstall" ]; then uninstall_and_exit; fi
if [ "$DL_ONLY" = 1 ] && [ "$LOCAL_ONLY" = 1 ]; then EXIT_MSG="--download et --install-only sont exclusifs (sans option : les deux phases)."; exit 2; fi

APPS=()
if [ "$LOCAL_ONLY" = 1 ] && [ ! -s "$CACHE/ready.txt" ]; then
  for _d in "$SCRIPT_DIR/install-apps-cache" "$HOME/install-apps-cache"; do
    if [ -s "$_d/ready.txt" ]; then CACHE="$_d"; APT_CACHE="$CACHE/apt"; break; fi
  done
fi
if [ "$LOCAL_ONLY" = 1 ] && [ ${#ARGS_APPS[@]} -eq 0 ] && [ "$SEL_ALL" = 0 ] && [ "$SEL_MIN" = 0 ]; then
  [ -s "$CACHE/ready.txt" ] || { EXIT_MSG="Aucun téléchargement trouvé dans $CACHE : lancez d'abord « bash $0 --download »."; exit 1; }
  mapfile -t APPS < "$CACHE/ready.txt"
elif [ "$SEL_ALL" = 1 ]; then
  APPS=("${IDS[@]}")
elif [ "$SEL_MIN" = 1 ]; then
  APPS=("${MIN_APPS[@]}")
elif [ ${#ARGS_APPS[@]} -gt 0 ]; then
  APPS=("${ARGS_APPS[@]}")
elif [ -n "$SILENT_APPS" ]; then
  read -r -a APPS <<< "$SILENT_APPS"
else
  [ -t 0 ] || { echo "Pas de terminal interactif : passez des apps en argument ou remplissez SILENT_APPS." >&2; exit 1; }
  menu
  if [ "$UNINSTALL_PICKED" = 1 ]; then uninstall_and_exit; fi
fi
[ ${#APPS[@]} -gt 0 ] || { EXIT_MSG="Rien à installer."; exit 0; }
echo "À installer : ${APPS[*]}"
log "apps: ${APPS[*]}"
[ "${DRY_RUN:-0}" = "1" ] && exit 0

do_install

echo
if [ ${#FAILED[@]} -eq 0 ] && [ "$DL_ONLY" = 1 ]; then
  echo "Téléchargement terminé. Installez ensuite, sans réseau : bash $0 --install-only"
elif [ ${#FAILED[@]} -eq 0 ]; then
  echo "Terminé sans erreur. Les apps apparaissent dans le lanceur ChromeOS (dossier « Apps Linux »)."
else
  echo "Terminé avec des erreurs : ${FAILED[*]}" >&2
  EXIT_MSG="Terminé avec des erreurs : ${FAILED[*]} (journal : $FINAL_LOG)"
fi
pause_end
[ ${#FAILED[@]} -eq 0 ] || exit 1
