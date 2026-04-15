Last updated: {{ git_revision_date_localized }}


# Installation
 All commands in this guide are run from cage/ unless stated otherwise. This guide is optimized for windows users planning to use anemulator for the game
## Folder Layout


```
cage/
  lunar-tear/               ← server repo (https://gitlab.com/walter-sparrow-group/lunar-tear)
    server/
      assets/               ← game data
        master_data/        ← JSON tables dumped from .bin.e
        release/            ← .bin.e file goes here
        revisions/          ← extracted asset bundles go here
      snapshots/         
  lunar-scripts/            ← scripts repo (https://gitlab.com/walter-sparrow-group/lunar-scripts)
    dump_masterdata.py
    patch_masterdata.py
    patch_apk.py
    schemas.json            
  20240404193219.bin.e      ← download from #resources
  NieR Re[in]carnation 3.7.1.apk        ← download from #resources
  resource_dump_android.7z      ← download from #resources
```

---

## Prerequisites

Install all of the following before starting (do remember to include them in your PATH, if the installers doesn't do it for you):

| Tool                                              | Where to get it |
| ------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Go 1.24+**                                      | https://go.dev |
| **Python 3**                                      | https://python.org |
| **Java (JDK)**                                    | https://adoptium.net |
| **apktool**                                       | https://apktool.org |
| **Android build tools** (`apksigner`, `zipalign`) | Install Android Studio → SDK Manager → SDK Tools → "Android SDK Build-Tools" |
| **protoc**                                        | https://github.com/protocolbuffers/protobuf/releases |
| Make                                              | https://sourceforge.net/projects/gnuwin32/files/make/3.81/make-3.81.exe/download |
| Terminal                                          | https://apps.microsoft.com/detail/9n0dx20hk701 |

- Install the following dependencies

```bash
pip install pycryptodome msgpack lz4
```

```bash
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
```

---

## Step 1 — Prepare the Asset Folders

Open your terminal in cage/

Create the required folders and copy the master data binary into place:

```bash
mkdir -p lunar-tear/server/assets/release
mkdir -p lunar-tear/server/assets/master_data
mkdir -p lunar-tear/server/assets/revisions
```

Copy/move the `.bin.e` into the release folder (keep the original at the `cage` root untouched).

Extract `resource_dump_android.7z` into `lunar-tear/server/assets/revisions/`. 

Note : Only *Revision 0* is necessary currently so only extract folder 0 to save on space 

---

## Step 2 — Dump the Master Data

This decrypts the `.bin.e` and converts it into JSON files the server can read. Column names are resolved automatically from `lunar-scripts/schemas.json` — no external dependencies needed.

To avoid an issue with Unicode, run this line first :

```bash
$env:PYTHONUTF8 = 1
```

And then 

```bash
python3 lunar-scripts/dump_masterdata.py  --input lunar-tear/server/assets/release/20240404193219.bin.e --output lunar-tear/server/assets/master_data
```

When done, `lunar-tear/server/assets/master_data/` should be filled with `EntityM*.json` files.

---

## Step 3 — Patch the Master Data

```bash
python3 lunar-scripts/patch_masterdata.py  --input lunar-tear/server/assets/release/20240404193219.bin.e
```

> **Important:** This extends all content expiry dates inside the .bin.e so events, quests, and banners don't show as expired. It also empties the maintenance table so the game never shows a maintenance screen.

---

## Step 4 — Patch the APK

Decompile the APK

```bash
apktool d "NieR Re[in]carnation 3.7.1.apk" -o patched
```
Replace `100.x.x.x` with :
For Tailscale users : Tailscale IP
For Emulator users : `10.0.2.2`

```bash
python3 lunar-scripts/patch_apk.py patched --server-ip 100.x.x.x --http-port 8080
```
Rebuild the APK

```bash
apktool b patched -o patched.apk
```
Generate a signing key (run one-time only)
 *Skip this if you already have debug.keystore.*

```bash
keytool -genkeypair -v -keystore client/debug.keystore -alias androiddebugkey -storepass android -keypass android -keyalg RSA -keysize 2048 -validity 10000 -dname "CN=Android Debug,O=Android,C=US"
```
Sign the APK 

```bash
apksigner sign --ks debug.keystore --ks-pass pass:android patched.apk
```
You now have patched.apk ready to install. 

---

## Step 5 — Generate Server Code

```bash
cd lunar-tear/server
make proto
```
This generates Go code from the .proto files. Only needs to be done once.

---

## Step 6 — Run the Server

```bash
cd lunar-tear/server
go run ./cmd/lunar-tear --host 100.0.2.2 --http-port 8080 --scene 0
```

Use `--scene x`  argument to skip over the undesired content.

> **Important:** The server only saves your progress when you advance to a new story scene. Gacha pulls, purchases, and enhancements done between scene transitions are lost if the server shuts down before you reach the next checkpoint. This is a current limitation of the server.

Leave the terminal open — the server must keep running while you play.

---

## Step 7 — Install and Play

### Android phone or tablet (not tested personally)

Install the Tailscale app from the Play Store (see [Tailscale Setup](#tailscale-setup)), then install the patched APK:

```bash
adb install patched.apk
```

Or transfer `patched.apk` to your phone and tap to install.

### Android emulator (BlueStacks, MuMuPlayer...) - Tested

Drag and drop `patched.apk` onto the emulator window, or:

```bash
adb connect 127.0.0.1:<emulator-adb-port>
adb install patched.apk
```

Since the emulator runs on the same PC as the server, Tailscale isn't required for home use.

> If you're using the emulator on a **different PC away from home**, install Tailscale on that PC and it will route through automatically.

### Make sure the server is running before launching the game.
