# Phase 0 Fix R1A — two lock bugs

Checked at `ce03448`, submodule `1b9f89d`. Nothing edited, built or run. Shorthand as in PHASE0_VERIFY.

## (a) Mutex destroyed while held — confirmed, with a correction

- `JamesDSPFree` locks at `LIB/jdspController.c:1225`, destroys at `:1255` (VERIFY cited `:1254`, the `if`), unlocks at `:1261`.
- On Android the destroy does nothing: bionic returns `EBUSY` for a locked mutex and leaves it untouched ([bionic source](https://android.googlesource.com/platform/bionic.git/+/refs/heads/main/libc/bionic/pthread_mutex.cpp)).
- The damage comes later. `:1261` wakes any waiter (audio thread at `LIB/jdspController.c:327`, setters such as `LIB/Effects/liveprogWrapper.c:98`), then `JNI:157` frees the struct holding the mutex. That is Risk 1 again.

**Smallest fix:** move `:1254-1255` below `:1261` (unlock, then destroy).

**Catch:** the file is in the upstream submodule, so this needs your own fork of `JamesDSPManager`. It also does not stop the crash. Recommended: skip it now and include it in the Risk 1 fix.

**Phone test** (regression only; the race is rare):

1. Play music. Turn on Convolver and Liveprog.
2. Tap the power button off, wait 1 second, tap it on, accept the capture prompt. Repeat 20 times.
3. Pass: the app never closes, music returns every time, and logcat shows `dtor: memory freed` after each off.

## (b) Second unlock at JNI:622 — confirmed

- `LiveProgStringParser` locks at `LIB/Effects/liveprogWrapper.c:98` and unlocks on every exit (`:103`, `:111`, `:145`).
- `JNI:622` unlocks again. It is the only lock call in the wrapper.
- The mutex is the default type (`LIB/jdspController.c:1049`). Bionic's unlock for that type has no owner check (same source). If the audio thread holds the lock (`LIB/jdspController.c:327-338`), `JNI:622` releases it, and a setter or `free` can enter mid-block. If nobody holds it, nothing happens.
- Runtime effect still unverified.

**Smallest fix:** delete `JNI:621-622` (the comment and the call). Your own code, no submodule change.

**Phone test:**

1. Play music. Turn on Convolver and Liveprog.
2. Switch between two Liveprog scripts 30 times, fast.
3. Pass: music never stops, each script is audible, the app never closes.
4. Fail: sound goes silent after a switch and stays silent (lock never released).

## Logcat for both tests

Phone on USB, USB debugging on. On the PC:

```
adb logcat -c
adb logcat -s JamesDspWrapper_JNI:* libc:F DEBUG:*
```

Any `libc` or `DEBUG` line during a test is a native crash: fail.

Run each test before and after the fix. Both need a debug build, which BASELINE_BUILD.md says does not exist yet.
