# Garmin Connect IQ on Ubuntu 26.04

Getting from zero to a compiling watch-face boilerplate for the Forerunner 265.
Recorded 25–26 Oct 2026.

**Environment**

|Instrument |Description |
|---|---|
| OS | Ubuntu 26.04.1 LTS |
| Java | OpenJDK 25.0.4.1 |
| Connect IQ compiler | 9.3.0 |
| VS Code extension | `garmin.monkey-c` v1.1.3 |
| Target device | Forerunner 265 (`fr265`), 416×416 AMOLED, API level 5.2 |

---

## 1. The SDK Manager has no installer

`connectiq-sdk-manager-linux.zip` contains only `bin/` and `share/` — no
top-level folder, no install script. Unzipping *is* the installation.

**The trap:** because the archive has no root directory, it unpacks straight
into whatever directory you happen to be in. Extracting twice from two
different locations scattered copies across the filesystem:

```
/home/rohit/Documents/connectiq-sdk-manager-linux/bin/sdkmanager   # intended
/home/rohit/bin/sdkmanager                                         # stray
/home/rohit/.local/share/Trash/files/.../bin/sdkmanager            # deleted
```

The stray copy in `~/bin` matters beyond tidiness — that directory is on
`$PATH` on most distros, so a bare `sdkmanager` would launch the wrong one.

**Fix:** always `mkdir` a target directory and extract into it. To locate
existing copies:

```bash
find ~ -name sdkmanager -type f 2>/dev/null
```

---

## 2. WebKitGTK ABI mismatch — the main blocker

```
./bin/sdkmanager: error while loading shared libraries:
libwebkit2gtk-4.0.so.37: cannot open shared object file
```

Garmin still links the SDK Manager against WebKitGTK **4.0**, which Ubuntu
dropped after 22.04. Ubuntu 26.04 ships 4.1 only. This has been an open bug
report against Garmin for well over a year and affects Debian 13, Fedora 42/43
and every recent Ubuntu LTS.

The same mismatch can also present as a window that opens but renders
**blank or white** rather than failing outright.

### Check what's available

```bash
ls /usr/lib/x86_64-linux-gnu/ | grep -E 'webkit2gtk|javascriptcoregtk|libsoup'
```

On 26.04 this returns `libwebkit2gtk-4.1.so.0`,
`libjavascriptcoregtk-4.1.so.0` and `libsoup-3.0.so.0` — all three present,
so symlinking works.

### Fix that worked

```bash
cd /usr/lib/x86_64-linux-gnu/
sudo ln -s libwebkit2gtk-4.1.so.0        libwebkit2gtk-4.0.so.37
sudo ln -s libjavascriptcoregtk-4.1.so.0 libjavascriptcoregtk-4.0.so.18
sudo ln -s libsoup-3.0.so.0              libsoup-2.4.so.1
```

4.0 and 4.1 are not a guaranteed-compatible ABI, so this is pragmatic rather
than correct. It held fine for the SDK Manager. A future WebKit update could
replace the targets and break the links.

To list everything still unresolved in one go rather than discovering missing
libraries one error at a time:

```bash
ldd ./bin/sdkmanager | grep "not found"
```

### Fallbacks if symlinking fails

**Distrobox on 22.04** — no system-wide symlinks:

```bash
distrobox create --name garmin-sdk --image ubuntu:22.04
distrobox enter garmin-sdk
sudo apt install libwebkit2gtk-4.0-37 libjavascriptcoregtk-4.0-18 \
  libsoup2.4-1 libsecret-1-0 libxkbcommon0 libsm6 libgtk-3-0
```

**Third-party CLI** — skips the GUI entirely, scriptable, CI-friendly:

```bash
go install github.com/lindell/connect-iq-sdk-manager-cli@latest
connect-iq-sdk-manager agreement view
connect-iq-sdk-manager agreement accept --acceptance-hash=<hash>
connect-iq-sdk-manager login
connect-iq-sdk-manager sdk set 9.3.0
connect-iq-sdk-manager device download
```

---

## 3. Two directories, easily confused

| Path | What it is |
|---|---|
| `~/Documents/connectiq-sdk-manager-linux/` | The manager application, from the zip |
| `~/.Garmin/` | Everything the manager downloads |

Inside `~/.Garmin/ConnectIQ/`:

- `current-sdk.cfg` — one-line text file holding the path to the active SDK
- `Sdks/` — the SDKs themselves (compiler, simulator, API docs)
- `Devices/` — per-device configs: screen dimensions, API level, memory limits

The manager app can be moved or re-extracted freely; `~/.Garmin/` is the state
that matters.

---

## 4. Devices are grouped by API level, not name

The Devices tab lists `API level 5.1`, `API level 5.0`, `3.4`… and you must
expand the right group to find your watch. **Forerunner 265 sits under
5.2** — this reflects the device's *current firmware* level, not what it
shipped with, so don't go looking under the launch-era levels.

Both `fr265` (46mm, 416×416) and `fr265s` (42mm, 360×360) are available. Taking
both lets one project target the pair, provided the layout is derived from
`dc.getWidth()`/`dc.getHeight()` rather than hardcoded.

---

## 5. PATH setup

```bash
echo 'export PATH=$PATH:$(cat $HOME/.Garmin/ConnectIQ/current-sdk.cfg)/bin' >> ~/.bashrc
```

The `$(cat ...)` stays unevaluated in the file, so it re-resolves on every new
shell. That means switching SDK versions in the manager automatically
repoints `monkeyc` — no edit needed.

Verify:

```bash
monkeyc --version                        # Connect IQ Compiler version: 9.3.0
ls ~/.Garmin/ConnectIQ/Devices/ | grep -i fr265
```

---

## 6. Java version

Ubuntu 26.04 ships OpenJDK 25, newer than anything Garmin validates against.
It worked without intervention. Worth remembering as a suspect if the
simulator or packaging misbehaves later.

---

## 7. Developer key — the folder-vs-file confusion

`Monkey C: Generate a Developer Key` asks for a **folder** and chooses the
filename itself (`developer_key`). Typing `developer_key` at the prompt
produces:

```
/home/rohit/.Garmin/developer_key/developer_key
```

Functionally fine — the extension writes the full path into
`monkeyC.developerKeyPath` either way — but the doubled name is confusing
later. VS Code settings live at `~/.config/Code/User/settings.json`:

```json
{
  "monkeyC.developerKeyPath": "/home/rohit/.Garmin/developer_key"
}
```

**Back this file up off the machine.** Published apps are tied to the key that
signed them; losing it means no further updates to that store listing.

For CLI builds the extension setting doesn't apply — pass it explicitly:

```bash
monkeyc -y ~/.Garmin/developer_key -f monkey.jungle -d fr265 -o bin/face.prg
```

Otherwise: `ERROR: The private key was not specified. Use command line option: -y.`

---

## 8. Project creation choices

Command Palette → `Monkey C: New Project`.

**Type:** Watch Face

**Template:** *Simple with Settings* over *Simple*. Both draw only the time;
the former additionally scaffolds the settings plumbing, which spans three
files that have to agree with each other —
`resources/settings/properties.xml` (defaults),
`resources/settings/settings.xml` (Garmin Connect UI), and
`Application.Properties.getValue()` in code. Easier to start from a working
example than retrofit.

**Minimum API level: 4.2.0.** This was the one decision that got revised.
The initial instinct was to pick something low (3.0) to keep compatibility
wide. But VO2 max is only reachable through the Complications API, which
landed at 4.2.0 — see below. A lower floor simply won't compile
`Toybox.Complications`.

The minimum level is a compatibility *floor*, not a capability switch.
Setting 4.2 does not prevent using 5.x features on a 5.x device; it only
declares what a watch needs in order to install the face. Picking 5.2 would
needlessly exclude everything in between.

---

## 9. Data sources split into two kinds

Relevant to the API-level decision above.

**Direct, synchronous** — low API level, read on demand:

```monkeyc
var info = ActivityMonitor.getInfo();
var steps = info.steps;
var goal  = info.stepGoal;
var cals  = info.calories;

var hr   = Activity.getActivityInfo().currentHeartRate;  // may be null
var batt = System.getSystemStats().battery;              // Float 0–100
```

**Complications API only** — API 4.2.0+, asynchronous callback:

VO2 max, Body Battery, Training Status, Recovery Time. VO2 max is split into
`COMPLICATION_TYPE_VO2MAX_RUN` (24) and `COMPLICATION_TYPE_VO2MAX_BIKE` (25).

```monkeyc
_vo2Id = new Complications.Id(Complications.COMPLICATION_TYPE_VO2MAX_RUN);
Complications.registerComplicationChangeCallback(self.method(:onComplicationChanged));
Complications.subscribeToUpdates(_vo2Id);
```

Known gotchas: `getComplication()` requires a `Complications.Id` object, not
the raw type number — passing `24` fails unhelpfully. And simulator/device
divergence is commonly reported here, so verify on the actual watch.

---

## 10. Known ahead: the AMOLED always-on constraint

Not hit yet, but it shapes the design from the start.

In always-on mode the face updates once per minute against a strict budget.
From API 5.0.0 the heuristic changed from *10% of pixels* to *10% of total
luminance*, which allows more pixels if they're dimmer. Exceed it and the
system blanks the face. There is also a per-pixel rule, handled by shifting
drawn content — commonly by an offset derived from the current minute.

So the face is two renderers behind one `onUpdate`, switched on
`requiresBurnInProtection` plus sleep state. The simulator has a 24-hour
burn-in test under **File → View Screen Heat Map**.

Watch faces also get a much smaller memory allocation than device apps;
out-of-memory crashes on load are a classic failure mode as bitmaps and fonts
accumulate.

---

## Summary

| Step | Friction |
|---|---|
| Extract SDK Manager zip | No root dir — scatters files |
| Launch SDK Manager | **WebKitGTK 4.0 vs 4.1 — the real blocker** |
| Find device in manager | Grouped by API level; FR265 under 5.2 |
| PATH | Use `$(cat current-sdk.cfg)` for version independence |
| Java 25 | Newer than validated; worked anyway |
| Developer key | Folder prompt, not file — easy to nest |
| Min API level | 4.2.0 required for VO2 max, not 3.0 |

One real blocker, several papercuts. Everything downstream of the symlinks
worked first time.

---

## References

- [Connect IQ SDK](https://developer.garmin.com/connect-iq/)
- [Toybox.Complications API docs](https://developer.garmin.com/connect-iq/api-docs/Toybox/Complications.html)
- [SDK Manager does not work on Ubuntu 24.04](https://forums.garmin.com/developer/connect-iq/i/bug-reports/sdk-manager-does-not-work-on-ubuntu-24-04)
- [Update SDK dependencies for newer Ubuntu LTS versions](https://forums.garmin.com/developer/connect-iq/i/bug-reports/update-sdk-dependencies-for-newer-ubuntu-lts-versions)
- [Symlink workaround on Ubuntu 26.04](https://github.com/pcolby/connectiq-sdk-manager/issues/10)
- [connect-iq-sdk-manager-cli](https://github.com/lindell/connect-iq-sdk-manager-cli)
- [Watch Face Low- and High-Power Modes](https://forums.garmin.com/developer/connect-iq/b/news-announcements/posts/changes-to-watch-face-low--and-high-power-modes)
