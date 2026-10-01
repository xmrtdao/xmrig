# XMRT DAO rig on Android/Termux

Operational notes for running XMRig as an XMRT DAO worker on a Termux
phone. Nothing here changes mining logic; it documents the three upstream
behaviours that reliably waste an hour if you do not know them.

## 1. "donate-level": 0 is silently ignored by upstream XMRig

`config.json` accepts `"donate-level": 0`, the banner does not complain,
and XMRig keeps donating 1% of your hashrate.

`Pools::setDonateLevel()` in `src/base/net/stratum/Pools.cpp` only accepts
a level at or above `kMinimumDonateLevel`:

```cpp
void xmrig::Pools::setDonateLevel(int level)
{
    if (level >= kMinimumDonateLevel && level <= 99) {
        m_donateLevel = level;
    }
}
```

Upstream `src/donate.h` sets that floor to 1, so a configured 0 is
dropped and the default 1 stands. To genuinely disable donations you must
change the floor and rebuild:

```diff
 constexpr const int kDefaultDonateLevel = 1;
-constexpr const int kMinimumDonateLevel = 1;
+constexpr const int kMinimumDonateLevel = 0;
```

Rebuild with `cmake --build . -j$(nproc) --target xmrig` from `build/`.
Verify on the startup banner (`* DONATE       0%`) and on
`GET /2/summary` (`donate_level`). If the banner says 1%, the patch did
not take effect.

## 2. Redirecting stdout produces an empty log

XMRig block-buffers stdout when stdout is not a TTY. Launched from a boot
script or nohup with `>> boot.log 2>&1`, the log stays 0 bytes for the
entire life of the process and only appears at exit.

Use XMRig's own logger instead:

```bash
./xmrig -c ~/config.json --no-color --log-file ~/xmrig/logs/boot.log
```

## 3. Never run two miners

A second XMRig allocates its own ~2.3 GB RandomX dataset. On a phone with
~7.6 GB RAM that will OOM-crash the device. Two miners also contend for
the HTTP API port, and the loser's failure surfaces only as an
"HTTP API server failed to start" line while the miner keeps running — it
looks like a broken API, not a duplicate miner.

Guard on both a lock and liveness, because check-then-start races:

```bash
#!/data/data/com.termux/files/usr/bin/bash
CONFIG="$HOME/config.json"
XMRIG="$HOME/xmrig/build/xmrig"
LOGDIR="$HOME/xmrig/logs"

for _ in $(seq 1 30); do
    [ -x "$XMRIG" ] && [ -f "$CONFIG" ] && break
    sleep 2
done

LOCK="$HOME/xmrig/.miner.lock"
mkdir -p "$HOME/xmrig"
exec 9>"$LOCK"
if ! flock -n 9; then
    echo "$(date '+%F %T') another start-xmrig.sh holds the lock - skipping"
    exit 0
fi

if pgrep -f "$XMRIG" >/dev/null 2>&1; then
    echo "$(date '+%F %T') xmrig already running - skipping"
    exit 0
fi

mkdir -p "$LOGDIR"
cd "$HOME/xmrig/build" || exit 1
exec ./xmrig -c "$CONFIG" --no-color --log-file "$LOGDIR/boot.log"
```

Install at `~/.termux/boot/start-xmrig.sh` and grant the Termux:Boot app
"Run on boot" permission. Test the guard by firing several invocations at
a running rig; exactly one miner should survive.

## 4. Read hashrate from the log, not from /2/summary

`hashrate.total` in `GET /2/summary` is a fixed-size array with `null`
entries for threads that have not been ready long enough. Summing it
reports 550-680 H/s on a rig that is actually hashing at ~320 H/s,
because the populated entries change as threads settle.

The authoritative sustained figure is the miner's own line:

```
[2026-10-01 12:55:35.307]  miner    speed 10s/60s/15m 320.7 318.7 n/a H/s max 336.4 H/s
```

## 5. Keep one canonical config

`src/config.json` ships with `YOUR_WALLET_ADDRESS` and points at
`donate.v2.xmrig.com`. Running from `build/` with the stock
`-c ../../config.json` mines to a dead wallet with no visible error.

Keep the real config outside the tree (for example `~/config.json`) and
pass it explicitly with `-c`. This branch syncs `src/config.json` to the
live rig config so the in-tree copy is not a booby trap, but the
canonical file remains the one outside the repo.
