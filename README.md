# Ardour Windows Builder

Build a standalone 64-bit Windows installer for Ardour directly from source with a single command.

## **READ THIS FIRST**

**There is ZERO SUPPORT for this.** I do not maintain Ardour. I am just an idiot with six PCs (yes really) producing a small speedrun marathon called Speedromizer. If your ASIO drivers crackle, your audio engine refuses to start, or your favorite 12-band harmonizer EQ crashes, **do not open an issue here and do not harass the upstream Ardour developers**. You built this yourself, you own the broken shards.

**You need free time.** The build takes between **15 and 45 minutes** depending on your CPU and RAM. Also bring 10GB of free disk space (my builds hover around 5.5GB).

**If possible, you should support the devs.** Ardour is open-source (GPL), but its development is funded almost entirely by paid downloads of pre-built binaries on [ardour.org](https://ardour.org). This script exists for two reasons and two reasons only:
1) You're an Ardour developer and actually want to streamline Ardour Windows building. (Hi, please check [DEV_NOTES.md](DEV_NOTES.md)!)
2) You need an Ardour Windows build but do not want your banking info online.

**There is a genuine possibility that running your PC for the build time costs you MORE MONEY than $1.** If you are OK with the Ardour project having your banking info, pay $1. It literally costs $1. Especially if you live in Germany. Hi, I would know. FUCKING 40c/kWh???????????

## Instructions

1. Download and install MSYS2 from [msys2.org](https://www.msys2.org/) (keep the default `C:\msys64` path).
2. Open your Windows Start Menu and launch **MSYS2 MINGW64** (do not use the standard MSYS or UCRT shortcuts).
3. Choose how you want to run the script (Option A or B).
4. Follow the prompt to choose whether you want to build the latest release tag, master, or a custom branch.
5. When finished, find your completed installer (`Ardour-<VERSION>-x64-Setup.exe`) in the folder and run it.
6. Find out Windows SmartScreen thinks you got ratted because the installer isn't signed with a $400/yr corporate certificate. **Click More info -> Run anyway.**

---

### Option A: MICROSOFT WONT RAT YOU CHALLENGE (IMPOSSIBLE)

Paste this into MSYS2 MINGW64, hit Enter, and hope to god Microsoft (who owns GitHub now) doesn't decide to rat you randomly today.

```bash
bash <(curl -sSL https://gist.githubusercontent.com/fGeorjje/eec7dc193705cbeded240e0cf2b5f368/raw/25f3ecb5ab3a9ec5991311cc484ecffd9e47eeff/build-ardour.sh)
```

---

### Option B: Hard working, raw artisan 250+ copypasted lines of bash

Please don't forget your grocery pickup at Erewhon today.

```bash
bash <(cat << 'EOF'
#!/usr/bin/env bash
set -euo pipefail

[ "${MSYSTEM:-}" = "MINGW64" ] || { echo "Error: Run inside MSYS2 MINGW64." >&2; exit 1; }

export PATH="/mingw64/bin:/usr/bin:/c/Windows/system32:/c/Windows"
rm -f /var/lib/pacman/db.lck 2>/dev/null || true

pacman -Sy --noconfirm

if pacman -Qu 2>/dev/null | grep -qE "msys2-runtime|pacman|bash"; then
    echo "Fossil age MSYS2 detected. Upgrading..."
    pacman -Su --noconfirm --overwrite "*"
    echo "MSYS2 updated! Windows requires you to close this terminal,"
    echo "re-open MSYS2 MINGW64, and paste this command one more time."
    echo "Why? Fuck you. With love, Microsoft"
    exit 0
fi

# Refresh keys in case PEBKAC hasn't touched MSYS2 since the Obama administration
pacman -S --needed --noconfirm --overwrite "*" msys2-keyring 2>/dev/null || true

echo -e "\n1) Build latest release tag\n2) Build master branch\n3) Other options..."
read -r -p "Select [1-3]: " INIT_CHOICE < /dev/tty

if [ -f "wscript" ] && [ -d ".git" ]; then
    REPO_DIR="$(pwd)"
elif [ -d "ardour/.git" ]; then
    cd ardour && REPO_DIR="$(pwd)"
else
    command -v git &>/dev/null || pacman -S --needed --noconfirm git
    git clone "https://git.ardour.org/ardour/ardour.git" ardour 2>/dev/null || git clone "https://github.com/Ardour/ardour.git" ardour
    cd ardour && REPO_DIR="$(pwd)"
fi

git fetch --tags --quiet 2>/dev/null || true
mapfile -t TAG_LIST < <(git tag -l | grep -E '^[0-9]+(\.[0-9]+)+' | sort -V -r)
LATEST_TAG="${TAG_LIST[0]:-master}"

if [ "$INIT_CHOICE" -eq 1 ]; then
    git checkout "${LATEST_TAG}"
elif [ "$INIT_CHOICE" -eq 2 ]; then
    git checkout master 2>/dev/null || git checkout main 2>/dev/null || true
    git pull --quiet 2>/dev/null || true
else
    echo "1) Latest: ${LATEST_TAG}"
    echo "2) Current: $(git rev-parse --abbrev-ref HEAD 2>/dev/null) @ $(git rev-parse --short HEAD 2>/dev/null)"
    for i in {1..5}; do [ "$i" -lt "${#TAG_LIST[@]}" ] && echo "$((i+2))) Tag: ${TAG_LIST[$i]}"; done
    echo "8) Custom ref"
    read -r -p "Select [1-8]: " SUB_CHOICE < /dev/tty
    case "$SUB_CHOICE" in
        1) git checkout "${LATEST_TAG}" ;;
        2) ;;
        8) read -r -p "Ref: " REF < /dev/tty; git checkout "${REF}" ;;
        *) git checkout "${TAG_LIST[$((SUB_CHOICE-2))]}" ;;
    esac
fi

# Fucking obliterate vanilla portaudio if installed so ASIO doesn't explode
if pacman -Q mingw-w64-x86_64-portaudio &>/dev/null; then
    rm -f /mingw64/include/pa_asio.h 2>/dev/null || true
    pacman -Rdd --noconfirm mingw-w64-x86_64-portaudio 2>/dev/null || true
fi

# Do NOT install base-devel here — that pulls msys2-runtime and kills the running bash session!!!!
SYS_PKGS=(git tar unzip zip p7zip patch)
MINGW_PKGS=(
    toolchain cmake ninja yasm nasm pkgconf python python-setuptools nsis boost glib2
    glibmm libsigc++ cairo cairomm pango pangomm fontconfig freetype libpng libjpeg-turbo
    libxml2 libsndfile libsamplerate curl libarchive liblo taglib vamp-plugin-sdk rubberband
    fftw aubio lv2 serd sord sratom lilv portaudio-asio jack2 flac libogg libvorbis libusb
    libwebsockets readline gettext-tools itstool cppunit drmingw
)

echo "Fetching the entire universe of dependencies. Please wait warmly"
pacman -S --needed --noconfirm --overwrite "*" "${SYS_PKGS[@]}" "${MINGW_PKGS[@]/#/mingw-w64-x86_64-}"

# Don't let GCC eat all your RAM and bluescreen your stream
RAM_GB=$(($(awk '/MemTotal/ {print $2}' /proc/meminfo 2>/dev/null || echo 16777216) / 1048576))
CORES=$(nproc 2>/dev/null || echo 4)
JOBS=$(( RAM_GB <= 8 ? 2 : (RAM_GB <= 16 ? 4 : (CORES > 8 ? 8 : CORES)) ))

MINGW_DIR=$(cygpath -m /mingw64)
PYTHON_BIN="${MINGW_DIR}/bin/python.exe"

export CC="${MINGW_DIR}/bin/x86_64-w64-mingw32-gcc.exe" \
       CXX="${MINGW_DIR}/bin/x86_64-w64-mingw32-g++.exe" \
       AR="${MINGW_DIR}/bin/ar.exe" \
       WINDRES="${MINGW_DIR}/bin/windres.exe" \
       CFLAGS="-mstackrealign -Wa,-mbig-obj -D_USE_MATH_DEFINES" \
       CXXFLAGS="-mstackrealign -Wa,-mbig-obj -D_USE_MATH_DEFINES -DBOOST_NO_AUTO_PTR" \
       LDFLAGS="-L/mingw64/lib" \
       PKG_CONFIG_PATH="/mingw64/lib/pkgconfig" \
       PKG_CONFIG_LIBDIR="/mingw64/lib/pkgconfig"

rm -rf build
"${PYTHON_BIN}" ./waf configure \
    --check-c-compiler=gcc --check-cxx-compiler=g++ --dist-target=mingw \
    --with-backends=jack,dummy,portaudio --windows-key=Mod4 --cxx17 \
    --optimize --keepflags --also-include=/mingw64/include --also-libdir=/mingw64/lib \
    --prefix=/mingw64 --libdir=/mingw64/lib

# Bypass cmd.exe choking on '><' redirects like it's 1995
mkdir -p build/gtk2_ardour
(cd gtk2_ardour && "${PYTHON_BIN}" ../tools/fmt-bindings.py --platform=win32 --winkey=Mod4 --accelmap 1 ardour.keys.in > ../build/gtk2_ardour/ardour.keys)

"${PYTHON_BIN}" ./waf build -j"${JOBS}"
"${PYTHON_BIN}" ./waf i18n

BUILT_EXE=$(ls build/gtk2_ardour/ardour-*.exe 2>/dev/null | head -n1)
FULL_VER=$(basename "${BUILT_EXE}" | sed -E 's/^ardour-|\.exe$//g')
MAJ_VER=$(echo "${FULL_VER}" | cut -d. -f1)
APP_DIR="ardour${MAJ_VER}"
INSTALLER_EXE="Ardour-${FULL_VER}-x64-Setup.exe"

rm -rf dist_staging
mkdir -p dist_staging/{bin,lib/gtk-2.0/engines,share/locale,"share/${APP_DIR}"} \
         "dist_staging/lib/${APP_DIR}/"{backends,surfaces,panners,vamp,suil,LV2}

cp "${BUILT_EXE}" dist_staging/bin/Ardour.exe
cp build/libs/fst/ardour-vst*.exe dist_staging/bin/ 2>/dev/null || true
find build/libs/ -maxdepth 3 -name "*.dll" -exec cp -t dist_staging/bin/ {} +

for mod in backends surfaces panners; do
    find "build/libs/${mod}/" -name "*.dll" -exec cp -t "dist_staging/lib/${APP_DIR}/${mod}/" {} +
done
find build/libs/vamp-* -name "*.dll" -exec cp -t "dist_staging/lib/${APP_DIR}/vamp/" {} +
cp build/libs/clearlooks-newer/clearlooks.dll dist_staging/lib/gtk-2.0/engines/libclearlooks.dll 2>/dev/null || true
cp -r build/libs/plugins/*.lv2 "dist_staging/lib/${APP_DIR}/LV2/" 2>/dev/null || true

cp -t "dist_staging/share/${APP_DIR}/" \
    build/gtk2_ardour/{ardour.menus,ardour.keys,default_ui_config,clearlooks.rc,clearlooks.ardoursans.rc} \
    gtk2_ardour/{ArdourMono.ttf,ArdourSans.ttf} \
    share/chords.txt
cp -r -t "dist_staging/share/${APP_DIR}/" \
    gtk2_ardour/{icons,resources,themes} \
    share/{midi_maps,patchfiles,plugin_metadata,scripts,web_surfaces,export}
cp gtk2_ardour/icons/Ardour{,Bug}.ico COPYING dist_staging/share/

find build/ -name "*.mo" | while read -r mo; do
    install -D -m 644 "$mo" "dist_staging/share/locale/$(basename "$mo" .mo)/LC_MESSAGES/gtk2_${APP_DIR}.mo"
done

# Scrape the PE import table recursively so we don't dump 4 GB of unused crap into the installer
export MINGW_BIN_WIN="${MINGW_DIR}/bin"
"${PYTHON_BIN}" - << 'PYEOF'
import os, shutil, subprocess

staging = os.path.abspath("dist_staging")
bin_dir = os.path.abspath("dist_staging/bin")
mingw_bin = os.environ["MINGW_BIN_WIN"]
objdump_bin = os.path.join(mingw_bin, "objdump.exe")

copied = set()
sys_dlls = (
    "kernel", "user32", "gdi32", "msvcrt", "shell32", "ole", "ws2_", 
    "advapi", "winmm", "ntdll", "imm32", "rpc", "shlwapi", "comctl", 
    "crypt", "api-ms", "ext-ms", "d3d", "opengl32", "glu32", "setupapi", 
    "iphlpapi", "version", "uxtheme", "dwmapi", "winspool", "secur32", "dxgi"
)

def get_deps(fp):
    try:
        out = subprocess.check_output([objdump_bin, "-p", fp], text=True, stderr=subprocess.DEVNULL)
        return {l.split(":", 1)[1].strip() for l in out.splitlines() if l.strip().startswith("DLL Name:")}
    except Exception:
        return set()

while True:
    new = False
    for r, _, fs in os.walk(staging):
        for f in fs:
            if f.lower().endswith((".dll", ".exe")):
                for dep in get_deps(os.path.join(r, f)):
                    if not dep.lower().startswith(sys_dlls) and dep not in copied:
                        src, dst = os.path.join(mingw_bin, dep), os.path.join(bin_dir, dep)
                        if os.path.exists(src) and not os.path.exists(dst):
                            shutil.copy2(src, dst)
                            copied.add(dep)
                            new = True
    if not new:
        break

# Murder DLLs that break standalone PortAudio runtime
for dead in ("libjack.dll", "libjack64.dll", "dbghelp.dll"):
    p = os.path.join(bin_dir, dead)
    if os.path.exists(p):
        os.remove(p)
PYEOF

cp tools/x-win/nsis/FileAssociation.nsh .

cat << 'NSIS_EOF' > ardour_installer.nsi
SetCompressor /SOLID lzma
SetCompressorDictSize 32
!include "MUI2.nsh"
!include "FileAssociation.nsh"

Name "${PRODUCT_NAME}"
OutFile "${INSTALLER_EXE}"
RequestExecutionLevel admin
InstallDir "$PROGRAMFILES64\Ardour${MAJOR_VERSION}"
InstallDirRegKey HKLM "Software\Ardour\Ardour${MAJOR_VERSION}" "Install_Dir"

!define MUI_ICON "dist_staging\share\Ardour.ico"
!define MUI_UNICON "dist_staging\share\Ardour.ico"
!define MUI_FINISHPAGE_RUN "$INSTDIR\bin\Ardour.exe"
!define MUI_FINISHPAGE_RUN_TEXT "Launch ${PRODUCT_NAME}"

!insertmacro MUI_PAGE_WELCOME
!insertmacro MUI_PAGE_LICENSE "dist_staging\share\COPYING"
!insertmacro MUI_PAGE_DIRECTORY
!insertmacro MUI_PAGE_INSTFILES
!insertmacro MUI_PAGE_FINISH
!insertmacro MUI_UNPAGE_CONFIRM
!insertmacro MUI_UNPAGE_INSTFILES
!insertmacro MUI_LANGUAGE "English"

Section "${PRODUCT_NAME} Core (required)" SecMain
  SectionIn RO
  SetOutPath "$INSTDIR"
  File /r "dist_staging\bin"
  File /r "dist_staging\lib"
  File /r "dist_staging\share"
  
  WriteRegStr HKLM "Software\Ardour\Ardour${MAJOR_VERSION}" "Install_Dir" "$INSTDIR"
  WriteRegStr HKLM "Software\Microsoft\Windows\CurrentVersion\Uninstall\Ardour${MAJOR_VERSION}" "DisplayName" "${PRODUCT_NAME} (64-bit)"
  WriteRegStr HKLM "Software\Microsoft\Windows\CurrentVersion\Uninstall\Ardour${MAJOR_VERSION}" "DisplayVersion" "${FULL_VERSION}"
  WriteRegStr HKLM "Software\Microsoft\Windows\CurrentVersion\Uninstall\Ardour${MAJOR_VERSION}" "Publisher" "Ardour Community Build"
  WriteRegStr HKLM "Software\Microsoft\Windows\CurrentVersion\Uninstall\Ardour${MAJOR_VERSION}" "DisplayIcon" "$INSTDIR\bin\Ardour.exe,0"
  WriteRegStr HKLM "Software\Microsoft\Windows\CurrentVersion\Uninstall\Ardour${MAJOR_VERSION}" "UninstallString" '"$INSTDIR\uninstall.exe"'
  
  WriteUninstaller "$INSTDIR\uninstall.exe"
  CreateDirectory "$SMPROGRAMS\${PRODUCT_NAME}"
  CreateShortCut "$SMPROGRAMS\${PRODUCT_NAME}\${PRODUCT_NAME}.lnk" "$INSTDIR\bin\Ardour.exe" "" "$INSTDIR\bin\Ardour.exe" 0
  CreateShortCut "$SMPROGRAMS\${PRODUCT_NAME}\Uninstall.lnk" "$INSTDIR\uninstall.exe" "" "$INSTDIR\uninstall.exe" 0
  CreateShortCut "$DESKTOP\${PRODUCT_NAME}.lnk" "$INSTDIR\bin\Ardour.exe" "" "$INSTDIR\bin\Ardour.exe" 0
  ${registerExtension} "$INSTDIR\bin\Ardour.exe" ".ardour" "Ardour Session"
SectionEnd

Section "Uninstall"
  DeleteRegKey HKLM "Software\Microsoft\Windows\CurrentVersion\Uninstall\Ardour${MAJOR_VERSION}"
  DeleteRegKey HKLM "Software\Ardour\Ardour${MAJOR_VERSION}"
  RMDir /r "$INSTDIR\bin"
  RMDir /r "$INSTDIR\lib"
  RMDir /r "$INSTDIR\share"
  Delete "$INSTDIR\uninstall.exe"
  RMDir "$INSTDIR"
  Delete "$SMPROGRAMS\${PRODUCT_NAME}\*.*"
  RMDir "$SMPROGRAMS\${PRODUCT_NAME}"
  Delete "$DESKTOP\${PRODUCT_NAME}.lnk"
  ${unregisterExtension} ".ardour" "Ardour Session"
SectionEnd
NSIS_EOF

makensis \
    -DPRODUCT_NAME="Ardour ${MAJ_VER}" \
    -DFULL_VERSION="${FULL_VER}" \
    -DINSTALLER_EXE="${INSTALLER_EXE}" \
    -DMAJOR_VERSION="${MAJ_VER}" \
    -V3 ardour_installer.nsi

echo -e "\nHoly shit, it actually compiled. Here's your installer:"
ls -lh "${INSTALLER_EXE}"
EOF
)
```