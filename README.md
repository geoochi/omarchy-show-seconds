# omarchy-show-seconds

Put a ticking seconds field on the [Omarchy](https://github.com/basecamp/omarchy)
bar clock.

The clock ships as `ddd d MMM HH:mm` and refreshes once a minute. This is the
small, update-safe change that makes it `ddd d MMM HH:mm:ss` and refresh every
second.

## Why this is not just a config edit

Two things in the built-in clock plugin stop `shell.json` from doing it alone,
and both live in package-owned files under `/usr/share/omarchy`:

- **Minute precision.** `SystemClock { precision: SystemClock.Minutes }` is
  hardcoded in the clock's `BarWidget.qml`. A `ss` you add to the format still
  renders — it just only changes once a minute.
- **A seconds-less format ring.** Right-clicking the clock cycles its label
  through `CLOCK_FORMATS` in `Model.js`, and every preset is minute-only. One
  right click drops the seconds and writes the minute format back to
  `shell.json`.

Editing `/usr/share/omarchy` directly is lost on the next Omarchy update, so
this clones the built-in plugin into `~/.config/omarchy/plugins` — the supported
way to own a built-in — and patches the two lines that matter. The bar keeps
routing the built-in id to the clone.

## Install

```sh
ln -s "$PWD/bin/omarchy-clock-seconds" ~/.local/bin/omarchy-clock-seconds
omarchy-clock-seconds install
```

`install` clones the built-in clock if needed, applies
[`patch/clock-seconds.patch`](patch/clock-seconds.patch), adds `:ss` to the
configured format, and restarts the shell. It is idempotent — rerunning it after
an Omarchy update re-applies the patch to the fresh clone.

The restart is not optional. This plugin ignores hot reload: a QML edit stays
inert until the shell is restarted, so without one the label keeps rendering the
old code and looks unchanged.

## Verify

```sh
omarchy-clock-seconds status
```

```
clone:   /home/geoochi/.config/omarchy/plugins/geoochi.clock
patch:   applied
format:  ddd d MMM HH:mm:ss
runtime: patched code compiled into the QML cache
```

The `runtime` line is the one that matters — it reads the seconds literals out
of the shell's compiled QML cache, which is the only way to tell that the
*running* shell has the patched code. `clone: ... / patch: applied` can be true
while the running shell still holds the old code.

## Uninstall

```sh
omarchy-clock-seconds uninstall
```

Restores the built-in clock (Omarchy backs up the removed clone), drops `:ss`
from the format, and restarts the shell.

## Caveats

- **Horizontal bars only.** The patch changes the horizontal presets. A vertical
  bar uses `VERTICAL_CLOCK_FORMATS`, which is left alone.
- **One preset has no time at all.** `d MMMM 'W'ww yyyy` is a date-only entry in
  the ring, so clicking far enough removes the clock's time entirely — seconds
  included. That is upstream's design; see [`AGENTS.md`](AGENTS.md) to change it.
- **Upstream drift.** A clone is a fork: future Omarchy improvements to the
  clock reach you only by re-cloning (remove the clone, rerun `install`).

`AGENTS.md` has the design rationale, the plugin contracts, and the pitfalls in
full.

## License

MIT. The patched files are derived from
[basecamp/omarchy](https://github.com/basecamp/omarchy), also MIT.
