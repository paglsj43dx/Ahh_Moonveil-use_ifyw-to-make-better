```
        __  __                 _   _      _ _
       |  \/  | ___  ___ _ __ | | | | ___(_) |
       | |\/| |/ _ \/ _ \ '_ \| | | |/ _ \ | |
       | |  | | (_) | (_) | | | \ V /  __/ | |
       |_|  |_|\___/ \___/|_| |_|\_/ \___|_|_|
              d e v i r t   ·   v0.1
```
# IF YOU USE THIS ON POLSEC YOU GET [ Polsec ] No KEY WAS PROVIDED ERROR SO IFYW USE THIS SOURCE TO MAKE IT BETTER GO AHEAD AND TRY TO BYPASS THAT. u jsut need to put the key
above
# ✧ﾟ･: moonveil-devirt :･ﾟ✧

*a lil trace-based devirtualizer for scripts obfuscated with **MoonVeil** (getpolsec.com)*
*runs the blob in a sealed cage, watches what it does, hands u back readable lua* 🕯️🖤

> ⚠️ real talk before u hype urself up:
> MoonVeil is a **real vm obfuscator** (chacha20 + sha-256 + base85 + a register vm).
> u do NOT get ur exact original source back. u get **what it actually did** —
> every url it hit, every file it touched, every string it decrypted — rolled back
> into lua that reads like source. thats the honest deal. anyone promising a
> perfect one-click unmask is lying to u fr fr 💔

---

## ⟡ two engines (this is the important part) 🔧

MoonVeil's decoder is calibrated to **Luau's exact runtime** (double-only numbers
etc). run it on stock lua 5.1/5.2/5.5 and the decode corrupts + dies mid-bootstrap
(each version dies in a *different* spot lol). so moonveil-devirt has two backends:

- 🥇 **`luau` backend (default, the good one)** — runs ur blob under the **real Luau
  VM** (the official `luau` standalone binary, auto-downloaded to `~/.cache/luau`).
  exact semantics → the obfuscator actually runs to completion → u get real behavior.
  the luau CLI is itself sandboxed (no io/net/roblox), we just add logging stubs.
- 🥈 **`lupa` backend (fallback)** — an embedded lua runtime w/ a debug-hook profiler.
  good for lua-flavored blobs + when u cant grab the luau binary.

pick with `--backend luau|lupa|auto` (auto = luau if available).

## ⟡ what it does

- 🔒 **runs the script in a sealed sandbox** — network, filesystem, roblox api are
  all *stubbed*. nothing hits the real machine. ur cookies stay urs.
- 📝 **logs every environment interaction** — `print`, `HttpGet`, `request`,
  `readfile`/`writefile`, `loadstring`, `getgenv`, all of it — in order, with args.
- 🧩 **decrypts constants for free** — u dont reverse the chacha/base85 by hand.
  the vm decrypts its own strings while it runs; we just watch.
- 🔁 **rolls repeats back into loops** — 300 identical `print`s become
  `for i = 1, 300 do ... end`. reads like source, not a log dump.
- 🩸 **triage view** — flags webhooks, stage-2 pulls, file steals up top so u can
  tell "is this malware" in 5 seconds.
- 🛑 **anti-hang** — hostile infinite vm loop? hard step budget kills it.

## ⟡ what it does NOT do (be honest w urself)

- ❌ recover original variable / function names (theyre *gone*, compiled away)
- ❌ rebuild exact control flow of code paths that **didnt run** in the trace
  (dynamic tools only see what executes — feed it the right entry/inputs)
- ❌ magically defeat every MoonVeil version forever (obfuscators update; this is v0.1)

---

## ⟡ how 2 run it

```bash
pip install -r requirements.txt          # lupa (fallback engine); luau auto-downloads

python3 devirt.py yourblob.lua --report            # print to screen + stats
python3 devirt.py yourblob.lua -o clean.lua        # write reconstruction
python3 devirt.py yourblob.lua --json trace.json   # raw event trace too
python3 devirt.py yourblob.lua --backend luau      # force the real-Luau engine
python3 devirt.py yourblob.lua --timeout 60        # (luau) wall-clock abort
python3 devirt.py yourblob.lua --no-download       # dont fetch the luau binary
python3 devirt.py yourblob.lua --backend lupa --fill-globals   # fallback + proxy all globals
```

> the luau binary auto-downloads to `~/.cache/luau` on first run. offline? grab
> `luau-ubuntu.zip` from github.com/luau-lang/luau/releases yourself and point
> `MOONVEIL_LUAU=/path/to/luau` at it (or drop it on PATH).

### try it on the included samples

```bash
# a REAL MoonVeil blob (a "PolSec" key-gated script):
python3 devirt.py samples/real_sample.lua --report
#  -> runs under real Luau, recovers the no-key behavior:
#       getgenv() ; warn("[ PolSec ] Error: No key was provided.")
#       game:GetService("Players").LocalPlayer:Kick("No key was provided.")
#       game:GetService("CoreGui") ... shows an ErrorPrompt
#       while true do wait(9999) end   -- hangs forever without a key

# lua-flavored synthetic samples (use --backend lupa):
python3 devirt.py samples/synth_vm.lua --backend lupa
#  -> recovers:  for i = 1, 3 do print(i, "hello from moonveil") end
python3 devirt.py samples/synth_exfil.lua --backend lupa --fill-globals
#  -> triage flags the discord webhook + cookie steal + stage-2 loadstring
```

*(`real_sample.lua` is an actual MoonVeil blob. the `synth_*` ones are hand-made
 MoonVeil-**shaped** vms for the offline/lupa path.)*

## ⟡ what it found on a real sample 🩸

pointed at a real 398KB MoonVeil blob (`samples/real_sample.lua`) it fully executed
under Luau and recovered: it's a **PolSec key-gated loader**. with no valid key it
`getgenv()`s, `warn`s `"[ PolSec ] Error: No key was provided."`, **kicks the
player**, pops a roblox `ErrorPrompt`, then **hangs on `while true do wait(9999) end`**.
the real payload lives behind the key check. thats the honest result — no key,
no payload, but u can see exactly what the gate does.

---

## ⟡ how it works (the honest teardown)

```
  your blob.lua
       │
       ▼
  ┌─────────────┐   static fingerprint: banner? sha256 consts? vm shape?
  │  detect.py  │
  └─────────────┘
       │
       ▼
  ┌───────────────────────────────────────────────┐
  │  sandbox.py + stubs.lua  (sealed lua env)       │
  │   · bit32 shim (luau parity)                    │
  │   · print/warn         -> logged                │
  │   · game/HttpGet/req.  -> logged, faked         │
  │   · readfile/writefile -> logged, no-op         │
  │   · loadstring/require -> logged, NOT executed  │
  │   · io/os side effects -> sealed off            │
  │  instrument.lua: debug hook = dispatch profile  │
  │                  + hard step budget             │
  └───────────────────────────────────────────────┘
       │  ordered event trace + vm hot-loop profile
       ▼
  ┌────────────────┐  roll repeats into loops, format as lua,
  │ reconstruct.py │  add triage summary up top
  └────────────────┘
       │
       ▼
  readable behavioral .lua  🖤
```

(the `luau` backend swaps the middle box for the **real Luau VM** + `prelude.luau`
logging stubs, and parses the `__MVEVENT__` lines it prints. the `lupa` backend is
what's drawn above.)

why dynamic and not static? bc MoonVeil replaces ur code with **bytecode for a
machine it invents per build**. statically proving what its hundreds of opcodes
mean is a multi-week symbolic project. running it and *watching* gets u the
answer today. this is the same call the serious frameworks make.

why real Luau and not just lupa? bc MoonVeil's decoder is **calibrated to Luau's
number semantics** (all doubles). on stock lua (integer subtype) the decode
corrupts and the vm dies mid-bootstrap. only the real Luau VM runs it clean. 💡

## ⟡ wanna go further (real static devirt)

got **several** MoonVeil samples? thats when a real opcode-mapping devirtualizer
becomes worth building — one sample isnt enough to map the instruction set
confidently. the `--json` dump is designed to feed that next stage. hmu.

---

## ✦ FOR MORE STUFF LIKE THIS!! ✦

### ➤➤ https://discord.gg/9bgECTqeE ➤➤

pull up 🖤🕯️ we do obfuscation / deobf / lua wizardry & talk shop

---

*stay spooky. trust no blob u didnt trace urself.* 🦇
```
                              .  *  .  ˚  ✦
                          ˚  ✦   .   *  .
```
