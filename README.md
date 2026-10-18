# tv-vibehack · Coocaa edition

**Your TV, minus the ads.**

🇻🇳 **Tiếng Việt:** xem [README.vi.md](README.vi.md), hướng dẫn từng bước cho TV Coocaa.

Strip the bloatware from any Android TV or Google TV over Wi-Fi, and replace the ad-filled home
screen with a clean launcher that looks like an Apple TV. No root, no cable, nothing uninstalled,
and a one-line restore if you change your mind.

This is a fork of [Gathin23/tv-vibehack](https://github.com/Gathin23/tv-vibehack). It adds a
tested setup for **Coocaa / Skyworth Google TVs**, which need an extra step the other brands
don't, plus a few Windows fixes. Everything else works as in the original.

![The launcher running on a Coocaa 24" Google TV](docs/coocaa-home.png)

---

## Table of contents

- [How it works](#how-it-works)
- [Coocaa / Skyworth Google TV](#coocaa--skyworth-google-tv)
- [Quick start (any brand)](#quick-start-any-brand)
- [What you get](#what-you-get)
- [Supported TVs](#supported-tvs)
- [Doing it by hand](#doing-it-by-hand)
- [The launcher](#the-launcher)
- [Troubleshooting](#troubleshooting)
- [Questions people ask](#questions-people-ask)
- [Contributing](#contributing)
- [Licence](#licence)

---

## How it works

Android TVs have the same debug bridge as Android phones. Once you switch on USB debugging in the
TV's developer options, the TV listens on your Wi-Fi and asks you once to allow your computer.
After that, everything is standard Android tooling:

1. **Back up** the list of installed packages and settings.
2. **Disable** the manufacturer's telemetry, ad services, content rows and preloaded junk with
   `pm disable-user`. Nothing is uninstalled, so a restore script can undo every step.
3. **Install** a small native launcher and make it the default home screen.

The bloat differs by brand, which is why the prompt asks your coding assistant to discover it
on your TV and ask you before touching anything you might use.

## Coocaa / Skyworth Google TV

Tested on a **Coocaa 24" FHD Google TV** (model `SWTV-24AE-FHD`, device `beomil`, MediaTek
MT5867, Mali-G31 GPU, 1 GB RAM, Android 14, firmware `UKN6.260116.048`). Other recent Coocaa and
Skyworth Google TVs share the same vendor packages (`com.smartdevice.*`, `com.skyworth.*`,
`com.coocaa.*`), so the list should be a good starting point for them too.

The debloat list with every package explained is
[`examples/coocaa-skyworth-beomil/debloat.txt`](examples/coocaa-skyworth-beomil/debloat.txt).

### What gets turned off

| Group | Packages |
|---|---|
| Coocaa content and ads | Coolita recommendation rows, vendor tweaks to the Google TV home rows |
| Store and service tools | retail demo mode, activation tracking, after-sales menu (with a silent APK installer), e-manual, the "Daily Manage" cleaner |
| Google extras | Feedback, TalkBack, Play Games, calendar sync, the Ambient mode screensaver |
| Launcher switch | the Google TV launcher and its setup wizard (see below) |
| Your choice | Chromecast, Coocaa screen mirroring (AirPlay/DLNA/Miracast), and any streaming apps you don't use |

Left alone on purpose: every `com.mediatek.*` package (HDMI, tuner, picture), the remote and
Bluetooth service `com.smartdevice.global.service`, the factory and panel services, the
on-screen keyboard, Play Store and Play Services.

### The extra step: Google TV won't give up the Home button

On most TVs, `cmd package set-home-activity` is enough. On this Google TV it isn't: even with the
new launcher set as default and holding the HOME role, the Home button still opens the stock
Google TV launcher (`com.google.android.apps.tv.launcherx`), and ADB is not allowed to disable
just its home activity. So the whole stock launcher has to be disabled.

That alone triggers a second problem: the Google TV setup wizard
(`com.google.android.tungsten.setupwraith`) notices the launcher is gone and covers the screen
with a "something went wrong" recovery page. Disabling the setup wizard too fixes it. Both come
back with the restore command, and you should restore them before a factory reset.

Expect the Wi-Fi to drop for about 30 seconds right after the stock launcher is disabled. ADB
reconnects on its own.

### Run it

On Windows, from PowerShell in the repo folder (replace the IP with your TV's):

```powershell
python -m venv tools\venv; tools\venv\Scripts\python.exe -m pip install -r tools\requirements.txt
$env:TV_HOST = '192.168.1.50'

# 1. connect: accept "Allow USB debugging?" on the TV the first time
tools\venv\Scripts\python.exe -I tools\tv.py --probe

# 2. debloat (edit the list first: comment out anything you use)
powershell -ExecutionPolicy Bypass -File tools\disable.ps1 examples\coocaa-skyworth-beomil\debloat.txt

# 3. build and install the launcher, replacing the stock Google TV launcher
powershell -ExecutionPolicy Bypass -File launcher\build-and-install.ps1 -StockLauncher com.google.android.apps.tv.launcherx
```

Step 3 needs JDK 17 and the Android SDK. Without Android Studio, this is enough:

```powershell
winget install -e --id EclipseAdoptium.Temurin.17.JDK
# then download "Command line tools only" from developer.android.com/studio, unzip it to
# %LOCALAPPDATA%\Android\Sdk\cmdline-tools\latest, and run:
sdkmanager "platforms;android-34" "build-tools;34.0.0"
```

Set `JAVA_HOME` to the JDK folder (for example `C:\Program Files\Eclipse Adoptium\jdk-17...`)
before step 3.

**Undo everything**, back to the stock Google TV home:

```powershell
powershell -ExecutionPolicy Bypass -File tools\restore.ps1 examples\coocaa-skyworth-beomil\debloat.txt -Stock com.google.android.apps.tv.launcherx
```

### Making it faster

On a 1 GB TV the biggest win is memory, not the launcher. On the test unit, disabling the list
above raised available RAM from about 240 MB to about 360 MB. Two more things helped:

- **Pre-compile the apps you use**, so they start faster and scroll without the JIT warming up.
  It takes a few minutes per app and survives until the app updates:

  ```bash
  tools/tv.py 'cmd package compile -m speed -f com.google.android.youtube.tv'
  ```

  Long compiles can outlast the client's 2-minute read timeout; they keep running on the TV.
- **Keep the "Daily Manage" cleaner off.** It force-stops apps in the background, so every app
  you open has to cold-start.

The launcher itself renders at a steady 60 fps on this TV: frames arrive every 16.7 ms with about
3.5 % missing their deadline (measured with `dumpsys gfxinfo` while moving through the dock and
grid). `dumpsys gfxinfo` also reports a "50th percentile" of about 30 ms, but that is pipeline
latency on this GPU, not the frame rate.

## Quick start (any brand)

You need a computer on the same Wi-Fi as the TV and a coding assistant with shell access
(Claude Code, or anything similar).

**1. On the TV**
Settings → Device Preferences (or System) → About → press OK on **Build** (or **Android TV OS
build**) seven times → go back → Developer options → turn on **USB debugging**. Some TVs also show
a Network or ADB debugging toggle; turn it on too. Keep the TV switched on.

**2. On your computer**
Open your assistant, paste the prompt from [`PROMPT.md`](PROMPT.md), and fill in your TV model.

**3. Follow along**
Accept the "Allow USB debugging?" prompt when it appears on the TV, then answer the checklist
questions: which streaming apps you use, which four go in the dock, whether to keep the voice
assistant, Chromecast and firmware updates.

The assistant backs up, debloats, builds and installs the launcher, and checks every step with
screenshots pulled from the TV. With this repo as context it takes about fifteen minutes.

## What you get

A home screen that is yours again:

- **Pitch-black background** and a dock of your four favourite apps at the bottom, in a lighter
  rounded band, exactly where Apple TV puts it.
- **A top shelf** showing the focused app's name in large type, with an ambient glow in a colour
  taken from the app's own artwork.
- **The rest of your apps in a grid** that slides up when you press Down.
- **One floating glass highlight** that springs from tile to tile instead of each tile drawing its
  own border. The focused tile lifts slightly and shows its name.
- **Three frosted buttons**: Inputs (a dropdown of HDMI ports, AV and Live TV), Wi-Fi (showing the
  network you're on) and Settings.
- **A greeting, a large thin clock and the date**, set in
  [Outfit](https://fonts.google.com/specimen/Outfit), in the TV's language.
- **A matching Clock screensaver**, optional, that drifts around the screen so nothing burns in.

And a TV that stops working against you: no sponsored rows, no "Discover" nags, no analytics
phoning home, no apps you never asked for.

## Supported TVs

Anything running **Android TV** or **Google TV**: Coocaa and Skyworth, Xiaomi and Redmi, TCL,
Hisense, Sony, OnePlus, Realme, Nvidia Shield and most others.

Not supported: Samsung (Tizen), LG (webOS), Amazon Fire TV, and Coocaa models that run Coolita OS
instead of Google TV. They run different systems or won't let a third-party launcher take over
Home.

Lists that people have tested live in [`examples/`](examples/):

| TV | OS |
|---|---|
| Coocaa / Skyworth 24" FHD (SWTV-24AE-FHD) | Google TV 14 |
| Xiaomi Mi TV 4A (MiTV-MOOQ1) | Android TV 10 |
| Xiaomi MiTV-AXSO0 | Android TV 9 |
| TCL 55" 4K Google TV, 2023 (G10) | Google TV 12 |
| SANYO ATV R2 | Android TV 9 |
| OnePlus TV (D00) | Google TV 14 |

Your TV not here? The prompt works regardless, it discovers the bloat itself. If you add your
list afterwards, the next person with your model skips that step. See
[Contributing](#contributing).

## Doing it by hand

Everything the assistant does, you can run yourself. The tools need Python 3 and nothing else.

```bash
python3 -m venv tools/venv && tools/venv/bin/pip install -r tools/requirements.txt
export TV_HOST=192.168.1.50                                   # your TV's IP address

tools/venv/bin/python -I tools/tv.py --probe                 # model, Android version, firmware
tools/venv/bin/python -I tools/tv.py 'pm list packages'      # run one shell command on the TV
printf 'cmd1\ncmd2\n' | tools/venv/bin/python -I tools/tv.py --batch   # several, one connection

tools/disable.sh examples/xiaomi-mi-tv-4a/debloat.txt        # apply a debloat list
tools/restore.sh examples/xiaomi-mi-tv-4a/debloat.txt        # undo it and restore the stock launcher
```

| Tool | What it does |
|---|---|
| [`tools/tv.py`](tools/tv.py) | A pure-Python ADB client: shell, batch, push, pull. Works where the stock `adb` binary is blocked or not installed. |
| [`tools/b64push.py`](tools/b64push.py) | Uploads larger files such as APKs as base64 chunks and verifies the checksum. |
| [`tools/disable.sh`](tools/disable.sh) / [`tools/restore.sh`](tools/restore.sh) | Apply or undo a debloat list. PowerShell versions sit next to them. |

### Windows

The PowerShell scripts work without WSL or Git Bash. They use `JAVA_HOME` and `ANDROID_HOME` when
set, otherwise Android Studio's bundled JDK and its default SDK location.

```powershell
python -m venv tools\venv; tools\venv\Scripts\python.exe -m pip install -r tools\requirements.txt
powershell -ExecutionPolicy Bypass -File launcher\build-and-install.ps1 -TvHost 192.168.1.50
powershell -ExecutionPolicy Bypass -File tools\disable.ps1 -TvHost 192.168.1.50 examples\xiaomi-mi-tv-4a\debloat.txt
powershell -ExecutionPolicy Bypass -File tools\restore.ps1 -TvHost 192.168.1.50 examples\xiaomi-mi-tv-4a\debloat.txt
```

The install script stops before touching the stock launcher if the install or the home-screen
switch fails. Git Bash also works; the `.sh` scripts detect a Windows venv and disable MSYS path
rewriting themselves.

## The launcher

A small native Android TV app (`com.gathin.tvhome`, about 1.6 MB), written in plain Java with
AndroidX RecyclerView and nothing heavier. It is built for sets with 1 GB of RAM: no blur, no
video, no network, no background services.

**Configure** it in [`launcher/app/src/main/res/values/config.xml`](launcher/app/src/main/res/values/config.xml):
the dock order, apps to hide, your TV's Live TV package, and the number of columns. On Coocaa the
Live TV package is `com.mediatek.wwtv.tvcenter`, which is already the default.

**Build and install** with one command:

```bash
TV_HOST=192.168.1.50 launcher/build-and-install.sh
```

It needs JDK 17 and the Android SDK (platform 34, build-tools 34.0.0). On macOS:

```bash
brew install openjdk@17 android-commandlinetools
sdkmanager "platforms;android-34" "build-tools;34.0.0"
```

Without Homebrew, the script also finds a JDK 17 known to `/usr/libexec/java_home` and an SDK in
`~/Library/Android/sdk`, Android Studio's default. A release build (`./gradlew assembleRelease`)
is smaller and faster on 1 GB sets; sign it yourself before installing.

**Navigation** is deliberate: Up and Down move exactly one row, Up from the dock goes to the top
buttons, the ends of the button row don't fall through to tiles, and the last focused app is
remembered when you come back, even after the app list changes.

**Rendering** is kept cheap for weak GPUs: the black background is drawn once by the window, and
the fading glow and labels skip the offscreen layer Android would otherwise allocate for every
alpha animation.

<details>
<summary><strong>Optional: the Clock screensaver</strong></summary>

A large thin clock and the date over an ambient glow, in the launcher's style. Every minute the
clock drifts to a new spot and the glow shifts colour. The system runs it only while the TV is
idle. Turn it on over ADB, or pick "Clock" in the TV's screensaver settings:

```bash
tools/venv/bin/python -I tools/tv.py 'settings put secure screensaver_components com.gathin.tvhome/.ClockDream; settings put secure screensaver_enabled 1'
```

It starts after the TV's "screen off" idle time (`settings get system screen_off_timeout`).
</details>

## Troubleshooting

Things that bit us or contributors, so they don't bite you.

**Connecting**

- **macOS: `adb connect` says "No route to host"** although the port is open. Recent macOS treats
  the Google-signed `adb` as its own app under Local Network privacy and silently blocks it.
  `tools/tv.py` avoids the binary entirely.
- **Finding the TV's IP**: Settings → Network on the TV, or your router's device list. A Google TV
  also answers on port 8008 (Chromecast), which shows up in a quick network scan even before ADB
  is on.
- **No "wireless debugging" toggle** on some TVs, Xiaomi among them. USB debugging alone opens
  network ADB on port 5555. The same goes for the Coocaa Google TV tested here.
- **Windows PowerShell 5.1 adds a byte-order mark** to text piped into a program, which turned the
  first batch command into gibberish. `tv.py` now strips it, and the `.ps1` scripts pipe clean UTF-8.
- **Windows: `UnicodeEncodeError: 'charmap' codec`** when the TV's output holds Vietnamese or other
  non-Latin text (a device or Wi-Fi name, `settings list`). `tv.py` now writes UTF-8 itself.
- **Git Bash rewrites paths** like `/data/local/tmp` into Windows paths. The scripts set
  `MSYS_NO_PATHCONV=1` for you; set it yourself when calling `tv.py --pull` directly. Set
  `JAVA_HOME` and `ANDROID_HOME` before building there.

**Working on the TV**

- **Chained `pm` commands** (`pm disable a; pm disable b`) may silently stop after the first. Run
  one per shell call; `--batch` does exactly that.
- **Large pushes hang** through the Python client. Use `tools/b64push.py` for APKs.
- **Screenshots come out empty** while Live TV, an HDMI input or a protected video is on screen
  (seen on TCL and Coocaa; logcat says `FB is protected`). Press Home before `screencap`.
- **Disable the stock launcher only after the new one is installed**, or the Home button lands on
  a factory test app.
- **Home still opens Google TV** after installing the launcher: disable the stock launcher, and
  if a "something went wrong" page appears, the setup wizard too. See
  [the Coocaa section](#the-extra-step-google-tv-wont-give-up-the-home-button).
- **No on-screen keyboard** after switching launchers, seen on Xiaomi: the default input method
  pointed at a Baidu keyboard that isn't installed. Fix:
  `tools/tv.py 'ime set com.google.android.inputmethod.latin/com.android.inputmethod.latin.LatinIME'`
- **Third-party launchers** often charge for basics like a custom wallpaper. That's why this repo
  has its own.

## Questions people ask

**Is this safe? Can it brick the TV?**
No. Nothing is flashed, rooted or deleted. Packages are disabled for the user profile, which
lives in the data partition, not in the system image. The firmware is untouched. If something
goes wrong, `tools/restore.sh` puts everything back, and even without the tools you can re-enable
apps from the TV's own settings.

**Does it touch Wi-Fi or Bluetooth?**
No. Wi-Fi, Bluetooth, the remote and paired devices are left exactly as they were and reconnect
on their own as before.

**What about firmware updates?**
The Coocaa list leaves the update service alone. If an update re-enables a package, re-run the
disable script. Other lists may disable the manufacturer's updater; check their comments.

**Won't the apps break as their APIs change?**
The streaming apps still update through the Play Store, which is left alone. The launcher
doesn't talk to them; it lists what's installed and launches it, using Android launcher intents
that have been stable for over a decade.

**How did it get access? Isn't that a security hole?**
Through the Android Debug Bridge, with one explicit "allow this computer" tap on the TV, the same
as developing an Android app. The TV only trusts the key you approved, which stays on your
computer in `tools/adbkey/` (ignored by git). You can switch USB debugging off again afterwards
and the launcher keeps working.

**Can I just download the launcher APK?**
Build it yourself with the one-line script above; it takes about a minute and you control what
goes on your TV. We don't publish prebuilt binaries.

## Contributing

The most useful contribution is your TV's debloat list, so the next person with that model
doesn't have to rediscover it. Launcher improvements and sharper prompt wording are welcome too.
See [CONTRIBUTING.md](CONTRIBUTING.md) for the format and the testing we ask for. If your list
would help everyone, consider sending it upstream to
[Gathin23/tv-vibehack](https://github.com/Gathin23/tv-vibehack) as well.

## Licence

MIT, see [`LICENSE`](LICENSE). Original work by Gathin. The Outfit typeface is under the SIL Open
Font License, see `launcher/FONT-LICENSE-OFL.txt`.
