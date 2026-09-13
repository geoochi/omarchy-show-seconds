# omarchy-show-seconds

A ticking seconds field on the Omarchy bar clock: `ddd d MMM HH:mm` becomes
`ddd d MMM HH:mm:ss`, updated every second instead of once a minute.

Everything here exists because the clock plugin can show seconds but does not,
for two independent reasons, and both are in files an Omarchy update owns.
This repository is the small patch and the clone that works around that.

Written against **Omarchy 4.0.3** (`/usr/share/omarchy/version` reports
`4.0.0.alpha`). File paths below are the upstream ones; line numbers are
indicative and will drift.

## The two blockers

### 1. Minute precision

`/usr/share/omarchy/shell/plugins/panels/clock/BarWidget.qml` builds the label
from

```qml
SystemClock {
  id: clock
  precision: SystemClock.Minutes
  onDateChanged: root.displayDate = date
}
```

The label text comes from `Qt.formatDateTime`, which understands `ss` just
fine — so adding `:ss` to the `format` in `shell.json` does render seconds. They
are simply stale: `dateChanged` fires on the minute, so the field reads `:00`
(or whatever second it was at the boundary) for a full minute. This is the trap
that makes the feature look half-applied rather than absent.

`SystemClock.Precision` (upstream `quickshell-core.qmltypes`) has exactly three
values — `Hours`, `Minutes`, `Seconds` — so `SystemClock.Seconds` is the fix.

### 2. A seconds-less format ring

Right-clicking the clock walks a ring of label formats, built in `Model.js`:

```js
function clockFormatRing(configured, configuredAlt, presets) {
  var candidates = (presets || []).concat([configuredAlt, configured])
  ...
}
```

`presets` are `CLOCK_FORMATS`, eight entries, every one of them minute-only.
Worse, `cycleFormat()` writes the chosen format back to `shell.json`, so one
right click does not just hide the seconds — it makes the minute format the
config from then on. A hand-written seconds format is appended to the end of the
ring, so right-clicking still leaves it on the very first click.

## Why a clone

Both files live under `/usr/share/omarchy`, which belongs to the `omarchy`
package; `pacman -Syu` overwrites any edit there. The supported way to own a
built-in is `omarchy-plugin-clone`, which copies it to
`~/.config/omarchy/plugins/<user>.<id>/` and switches the bar to the clone.

A third-party plugin can never do this by squatting the id: `PluginRegistry`
merges first-party plugins over third-party ones and drops any non-first-party
plugin whose id starts with `omarchy.` ("id is reserved for first-party Omarchy
plugins"). Third-party ids must be namespaced, which is exactly what
`omarchy-plugin-clone` does with the `${USER}.` prefix.

## Plugin contracts worth knowing

These are the non-obvious bits; they are why the clone needs no restructuring
and why some obvious-looking approaches fail.

- **The clone keeps `moduleName: "omarchy.clock"` on purpose.** The host does not
  read that property — it overwrites it: `Bar.qml` sets
  `moduleName: root.entryId(entry)` and `settings: root.entrySettings(entry)` on
  every widget instance. Identity and settings follow the *bar entry*, so
  renaming `moduleName` in the clone would only break the IPC target.
- **The bar entry id becomes the clone's id.** `omarchy-plugin-clone
  omarchy.clock` rewrites the `bar.layout.center` entry from `omarchy.clock` to
  `geoochi.clock` and carries the entry's existing settings (format, plus
  anything custom) across untouched.
- **Calls that name the built-in are routed to the clone.**
  `PluginRegistry.resolveEnabledId` maps `omarchy.clock` to the enabled plugin
  whose manifest says `omarchy.clonedFrom: "omarchy.clock"`.
- **The clone's own `IpcHandler` is dead.** The built-in plugin's
  `IpcHandler` wins the `omarchy.clock` target, and Quickshell warns that the
  clone's handler "was registered but will not be used". Consequence: you cannot
  drive the clone's `cycleFormat`/`toggleWeekStart` with
  `omarchy-shell omarchy.clock ...`, and `shell call omarchy.clock <method>`
  cannot reach it either — `callIfLoaded` only serves `panel`/`overlay`/`menu`
  plugins, not `bar-widget` ones. The right-click path is a direct QML call and
  is unaffected; it is only *testing* the ring through IPC that is impossible.
- **`shell.json` is watched; plugin code is not.** `shell.qml` loads the user
  config with `watchChanges: true`, so format edits apply live. Plugin QML/JS
  does not — see below.

## The restart is mandatory

This is the single biggest source of "I fixed it and nothing happened".

Editing a local plugin fires `localPluginChanged`, and the shell logs
`Local plugin changed, reloading: <id>`. It does **not** make the change take
effect. Both the precision fix and the preset fix sat unused until
`omarchy-restart-shell`; before that the label looked identical and right-click
still produced a minute-only format. Do not trust the log line, and do not trust
that the on-disk files are correct — they can be, while the running shell holds
the old code.

### Proving what the running shell actually has

Qt compiles the plugin's QML and JS into `~/.cache/quickshell/qmlcache/*.jsc`.
Grepping for the patched literals there is the only local check of what the
running process compiled:

```sh
strings -a -e l ~/.cache/quickshell/qmlcache/*.jsc | grep -c 'HH:mm:ss'
```

The strings are UTF-16, hence `strings -e l`; without it the grep misses and the
check silently reports "not patched". The built-in clock's cache entry contains
only `HH:mm`, so a hit proves the clone's patched code was compiled.
`omarchy-clock-seconds status` wraps this as its `runtime` line.

Two implementation notes for that check, both learned the hard way:

- Do not pipe into `grep -q`. `grep` exits on the first match, `strings` takes a
  SIGPIPE, and under `set -o pipefail` the pipeline reports failure on a
  successful match. Read the dump into a variable and test the string instead.
- The compiler cache is only rewritten once the widget instance exists, so treat
  a miss as inconclusive right after a restart, not as a failure.

## Layout

| Path | Purpose |
| --- | --- |
| `bin/omarchy-clock-seconds` | `install` / `uninstall` / `status` |
| `patch/clock-seconds.patch` | The two-file diff against upstream, applied inside the clone |

The patch is a diff against the pristine upstream files, `a/` -> `b/` prefixed,
so it applies with `patch -p1 -d <clone-dir>`. It was generated by diffing
`/usr/share/omarchy/shell/plugins/panels/clock/{BarWidget.qml,Model.js}` against
the working clone, and is verified to reproduce that clone byte-for-byte from a
pristine copy.

Nothing here vendors upstream source. The clone is produced on the machine by
`omarchy-plugin-clone`, so this repository never republishes Omarchy's files —
only the two-hunk delta.

## Install

```sh
ln -s "$PWD/bin/omarchy-clock-seconds" ~/.local/bin/omarchy-clock-seconds
omarchy-clock-seconds install
```

What `install` does, in order, and why:

1. `omarchy-plugin-clone omarchy.clock` — skipped if the clone already exists. It
   refuses to run if the bar already carries a clock owned by another clone,
   which is a state for a human to resolve.
2. `patch -p1` into the clone — skipped if `SystemClock.Seconds` is already
   there, so rerunning after an update is safe.
3. Add `:ss` to the bar entry's `format` in `shell.json`, via `jq`, preserving
   every other setting. The rule is "insert `:ss` after the first `mm` that is
   not already followed by `:ss`", so it is a no-op on the second run. Qt's
   minutes token is `mm`; months are `MM`, so there is no ambiguity.
4. `omarchy-restart-shell`, then verify via the QML cache.

`--no-restart` stops before step 4 for hands-off use.

## Uninstall

```sh
omarchy-clock-seconds uninstall
```

`omarchy-plugin-remove <clone> --yes` restores the built-in clock (it reads
`omarchy.clonedFrom` and re-enables the source), then `:ss` is stripped from the
format and the shell is restarted.

## Caveats

- **Horizontal bars only.** The patch rewrites `CLOCK_FORMATS`. A vertical bar
  uses `VERTICAL_CLOCK_FORMATS` — `"HH\n—\nmm"` and friends — which is untouched
  because seconds there need a fourth stacked line, a layout decision rather
  than a bug. Extending it means patching that array too.
- **One ring entry has no time at all.** `d MMMM 'W'ww yyyy` is a date-only
  preset, kept by upstream to answer "what's the date?". Enough right clicks
  land on it and the clock loses its time entirely, seconds included. It is the
  only remaining way right-click can take the seconds away, and it is upstream
  behaviour, not a defect in this patch. Drop it from `CLOCK_FORMATS` in the
  clone to make the ring seconds-only.
- **Upstream drift.** A clone is a fork: it stops receiving Omarchy's own clock
  changes. To pick them up, remove the clone and rerun `install`, which
  re-clones and re-applies the patch against the newer upstream.
- **The patch is context-sensitive.** It will fail loudly rather than silently
  mis-apply if upstream moves those hunks. That is the intended failure mode:
  re-clone, re-diff, regenerate `patch/clock-seconds.patch` from the new
  upstream.

## Development

```sh
bash -n bin/omarchy-clock-seconds
bin/omarchy-clock-seconds status
```

To regenerate the patch after an upstream change:

```sh
git clone --depth=1 https://github.com/basecamp/omarchy /tmp/omarchy   # or use the package files
mkdir -p /tmp/pg/a /tmp/pg/b
cp /usr/share/omarchy/shell/plugins/panels/clock/{BarWidget.qml,Model.js} /tmp/pg/a/
cp ~/.config/omarchy/plugins/geoochi.clock/{BarWidget.qml,Model.js}       /tmp/pg/b/
cd /tmp/pg && git diff --no-index a b | sed -e 's|^diff --git 1/a/|diff --git a/|' \
  -e 's| 2/b/| b/|' -e 's|^--- 1/a/|--- a/|' -e 's|^+++ 2/b/|+++ b/|' \
  > /home/geoochi/Work/omarchy-show-seconds/patch/clock-seconds.patch
```

The `sed` normalises the `1/a/`, `2/b/` prefixes `git diff --no-index` emits for
two untracked trees into the `a/`, `b/` pair `patch -p1` expects. Verify before
committing:

```sh
SB=$(mktemp -d)
cp /usr/share/omarchy/shell/plugins/panels/clock/{BarWidget.qml,Model.js} "$SB/"
patch -p1 -d "$SB" < patch/clock-seconds.patch
diff "$SB/BarWidget.qml" ~/.config/omarchy/plugins/geoochi.clock/BarWidget.qml
diff "$SB/Model.js"     ~/.config/omarchy/plugins/geoochi.clock/Model.js
```

Both diffs must be empty.
