# Reproducing and verifying the fix for issue #279

## What was wrong

`SetSender` unconditionally called `menu_icon()` on every `ReloadConfig`.
`menu_icon()` creates a new `NSStatusItem` via the `tray-icon` crate, which
runs `NSBezierPath` + `NSImage` rendering on the main thread — a GPU-composited
draw call that WindowServer must process.

When a new Rustcast version is available, `UpdateAvailable` fires every 60
seconds → `ReloadConfig` → `SetSender` → new `NSStatusItem`. Repeated over
hours, this drives WindowServer CPU to 30–38% and causes system-wide mouse lag.

## The stress test

`stress_windowserver_repro()` (in `src/app/tile.rs`) fires `UpdateAvailable`
every 2 seconds, compressing hours of production load into minutes.

## How to run the test

**Requirements:** macOS, Rust toolchain, Rustcast dependencies installed.

### 1. Build this branch

```
cargo build
```

### 2. Kill any existing Rustcast instance

```
pkill -x rustcast
```

### 3. Launch and watch WindowServer CPU in a second terminal

Terminal A — launch Rustcast:
```
cargo run
```

Terminal B — poll WindowServer CPU every 5 seconds:
```
while true; do
  echo "$(date +%H:%M:%S)  WindowServer: $(ps aux | awk '/WindowServer/ && !/awk/{print $3}' | head -1)%"
  sleep 5
done
```

### 4. Observe

After ~30–60 seconds you should see `Reloading config` in the Rustcast logs
roughly every 2 seconds. WindowServer CPU should stay **below 15%** throughout.

### 5. Verify the fix is load-bearing

To see what happens without the fix, revert the `SetSender` block in
`src/app/tile/update.rs` back to the original one-liner:

```rust
// Revert to this to observe the bug:
if tile.config.show_trayicon {
    tile.tray_icon = Some(menu_icon(tile.config.clone(), sender));
}
```

Rebuild (`cargo build`) and rerun. Within 60–90 seconds WindowServer CPU
will spike to **30–38%**, matching the bug report.

### Expected results

| Code | WindowServer CPU after 5 min |
|------|------------------------------|
| With fix (this branch) | 5–15%, flat |
| Without fix (reverted) | 30–38%, sustained spikes |

### 6. Cleanup

The stress subscription should be removed before merging to production.
Delete `stress_windowserver_repro()` from `src/app/tile.rs` and remove
`Subscription::run(stress_windowserver_repro)` from the `subscription()` batch.
