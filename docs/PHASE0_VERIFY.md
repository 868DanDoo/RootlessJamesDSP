# Phase 0 Verify — risks 1, 2, 3

Checked at `7e52fea`, submodule `1b9f89d`. Nothing built or run. Audit shorthand, plus `LIB` = `N/libjamesdsp/Main/libjamesdsp/jni/jamesdsp/jdsp/`.

## Risk 1 — use-after-free on shutdown

| Claim | Label |
|---|---|
| Engine freed by a 100 ms timer | Confirmed (`K/interop/JamesDspLocalEngine.kt:29-38`) |
| Audio thread waited on for only 500 ms | Confirmed (`SVC:567`, then `close()` at `SVC:227`) |
| IO setters never cancelled | Confirmed (no `super.close()`) |
| A slow call can use freed memory | Confirmed, narrower than stated |

The only guard is `handle = 0`: calls starting after `close()` return early (`JNI:40`). At risk is a native call that started before `close()` and still runs 100 ms later.

Threads that can be in native code:

1. **Audio thread**, `process*` (`SVC:530`, `:535`). At risk. The library lock covers only convolver, DDC and Liveprog (`LIB/jdspController.c:327-338`); other stages run unlocked, so free does not wait for them.
2. **IO coroutine worker**, all setters. At risk. Slowest are `setLiveprog` (runs the user's `@init`) and `setConvolver`.
3. **Main thread**, `alloc` and `setSamplingRate` (`SVC:133`, `:460`). Safe: same thread as `close()`.
4. **Timer thread**, `free` itself.
5. **Library workers** (convolver, EEL script threads). Joined inside free (`LIB/generalDSP/TwoStageFFTConvolver.c:138`, `LIB/Effects/eel2/nseel-compiler.c:1828`).

Not in the audit:

- `JamesDSPFree` destroys the mutex while holding it, then unlocks it (`LIB/jdspController.c:1254`, `:1261`). A thread waiting on that lock wakes into freed memory.
- `JNI:622` unlocks a second time after `LiveProgStringParser` already unlocked (`LIB/Effects/liveprogWrapper.c:145`). It can release the lock while the audio thread holds it. Runtime effect unverified.
- `free` clears process-wide state: the stdout handler (`JNI:154`) and shared resampler tables (`JNI:160`). An engine created within 100 ms of the old `close()` loses both.
- `SVC:580` resets the stop flag after a timed-out join, so a stuck audio thread would keep running beside the new one.

## Risk 2 — invalid handle

| Claim | Label |
|---|---|
| `alloc` returns `1` on malloc failure | Wrong in practice: `memset` (`JNI:119`) crashes on null first |
| `alloc` returns `2` on mutex failure | Confirmed (`JNI:131-138`) |
| Only `0` is treated as invalid | Confirmed, in Kotlin and native (`JNI:40`). `isHandleValid` exists but is never called |
| Likelihood | Unverified. Needs `pthread_mutex_init` to fail (`LIB/jdspController.c:1049`). Rank 2 looks too high |

## Risk 3 — `JNIEnv*` across threads

| Claim | Label |
|---|---|
| Env saved from the creating thread | Confirmed (`JNI:89`). That thread is main (`SVC:133`) |
| Reused in the callback | Confirmed (`JNI:747`) |
| Which thread fires it | Resolved |

No queue: EEL `printf` and error text go straight through `EEL_STRING_STDOUT_WRITE` (`LIB/Effects/eel2/ns-eel.h:115`) to the handler, on whichever thread runs the script:

- **IO coroutine worker**: compile errors and `@init` (`LIB/Effects/liveprogWrapper.c:58-65`).
- **Audio thread**: `@sample` (`LIB/Effects/liveprogWrapper.c:157`, called from `LIB/jdspController.c:337`).
- **EEL script thread**: code started by `ThreadInit` (`LIB/Effects/eel2/nseel-compiler.c:3539`). A raw pthread, not attached to the JVM.

Never the main thread, so the saved env is wrong on every call. Worse than the audit stated.
