# Balatro web build

The editable LÖVE sources are in `C:/Users/Marc/Documents/Balatro/Balatro_web`.
Build with `lovejs_tools/build.sh` there (love.js **compatibility** mode, `-c`,
256 MiB initial memory). Do not switch to the pthread build: a Chrome debugger
capture showed the main thread waiting in nested `emscripten_futex_wait` calls.

## Startup freeze fixed in 20261010-5

Reproduced by clicking Play, then Play on the New Run screen. A temporary Lua
instruction hook identified an endless resampling loop in `get_next_tag_key`
(`functions/common_events.lua`), reached from `Game:start_run`.

Balatro supplies fractional pseudoseeds to `math.randomseed`. Stock Lua 5.1
converts its argument to an integer, so seeds between zero and one all reset
the RNG to the same state. If that state picks an unavailable tag, resampling
never terminates. It also explains repeated cards with different run seeds.
See [Lua 5.1 implementation](https://www.lua.org/source/5.1/lmathlib.c.html#math_randomseed).

`engine/web_random.lua`, installed from `main.lua` on Web only, hashes the full
17-digit numeric seed representation into an integer before calling the native
seeder. The same seed remains deterministic, while fractional seeds no longer
all collapse to zero. Web seeded runs are not guaranteed to match LuaJIT runs.
Keep this source patch when re-exporting. No diagnostic instruction hook is
included in the distributed game.

The Web error handler also prints the traceback and returns a frame callback,
instead of trapping the browser in the desktop error loop.

## Publishing

Keep the customized `game.js` split-package loader. Split the exported
`game.data` into 20 MiB parts; update its package UUID and byte counts from the
generated loader. Copy the matching compatibility `love.js` and `love.wasm`.
Expose `Module.FS` only **after** initial IndexedDB loading completes, as in
`build.sh`, to preserve browser saves. Increment the asset version in
`balatro/index.html`. No link is added to the site's root index.

Before pushing, compare the concatenated parts with the original archive,
check every asset is below 25 MiB, and test New Run → Select Small Blind →
select cards → Play Hand in a real browser. Confirm a second new run starts
and gets a different hand. Merely loading the menu does not test this bug.
