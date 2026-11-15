# Phase 0 Fix R1 — engine freed while in use

Checked at `ee882ac`, submodule `1b9f89d`. Nothing edited, built or run. Shorthand as in PHASE0_VERIFY; `LE` = `K/interop/JamesDspLocalEngine.kt`, `BE` = `K/interop/JamesDspBaseEngine.kt`.

## 1. Threads and entry points

- **Main:** `alloc` (`LE:13`, `SVC:133`), `setSamplingRate` (`LE:18`, `SVC:460`), `close()` (`LE:29`, `SVC:227`).
- **Audio** (`SVC:464`): `processInt16` (`SVC:530`, `LE:53`, `JNI:224`), `processFloat` (`SVC:535`, `LE:85`, `JNI:304`).
- **IO worker** (`BE:33`, `:44`): 13 setters (`LE:91-163`, `JNI:326-643`).
- **Timer:** `free` (`LE:34-35`, `JNI:148`).
- **Library workers** (convolver, EEL script): joined inside free (`LIB/generalDSP/TwoStageFFTConvolver.c:138`, `LIB/Effects/eel2/nseel-compiler.c:1828`).
- **No callers found:** `processInt32` (`LE:69`), EEL utilities (`LE:173-183`), `isHandleValid`, `processInt8U24`, `processInt24Packed`.
- Every handle call goes through `LE`. Plugin flavor's audio thread: unverified.

## 2. Options

- **A. Stop and join** the audio thread and settings job, then free. Unbounded wait on main (ANR).
- **B. Lock in `LE`:** read lock around every native call; `close()` takes the write lock, then frees directly.
- **C. Global lock in the `JNI` macros.** Touches every JNI function; a refused `process` call writes no output.

**Recommended: B**, plus `super.close()` from A (cancels queued settings jobs).

- The lock lives in Kotlin, so it is never freed.
- Free waits for running calls; later calls see `handle == 0`.
- Audio uses `tryLock()`: never waits, falls back to copy-through.
- Wait capped at 2 s. On timeout the engine is leaked, not freed.
- Free ends before the next `alloc`, so it cannot wipe a new engine's process-wide state (`JNI:154`, `:160`).

## 3. Submodule

No change needed. R1A (a): free runs only under the write lock, so no thread holds or waits on the library mutex. The destroy (`LIB/jdspController.c:1255`) stays a no-op, the unlock (`:1261`) wakes nobody, and `JNI:157` frees a struct nobody uses. On bionic the undestroyed mutex leaks nothing (unverified). The clean fix (unlock, then destroy) needs a submodule fork; not required. R1A (b) is separate.

## 4. Edits — `LE` only

Line numbers are for `ee882ac`; apply in this order so they stay valid. `X` = the existing `JamesDspWrapper.…(handle, …)` expression.

1. Lines 183 and 18: `guard(Unit) { X }`.
2. Lines 91–178: `guard(false) { X }`. Exceptions: line 91, wrap the whole `… and …`; line 173, `guard(arrayListOf<EelVmVariable>()) { X }`.
3. Lines 75, 59, 43: replace `handle == 0L` with `!tryProcess { X }`, `X` taken from that function's `else` block; then delete the `else { … }` block (84–86, 68–70, 52–54).
4. Lines 29–38: replace `close()` with:

```kotlin
    override fun close() {
        super.close()
        if(!lock.writeLock().tryLock(2, TimeUnit.SECONDS)) {
            handle = 0
            Timber.e("close: engine leaked")
            return
        }
        try {
            val oldHandle = handle
            handle = 0
            JamesDspWrapper.free(oldHandle)
            Timber.d("Handle $oldHandle has been freed")
        } finally { lock.writeLock().unlock() }
    }

    private inline fun <T> guard(fallback: T, call: () -> T): T {
        lock.readLock().lock()
        try { return if(handle == 0L) fallback else call() }
        finally { lock.readLock().unlock() }
    }

    private inline fun tryProcess(call: () -> Unit): Boolean {
        if(!lock.readLock().tryLock()) return false
        try {
            if(handle == 0L) return false
            call()
            return true
        } finally { lock.readLock().unlock() }
    }
```

5. Line 13: prefix `@Volatile `. Insert above it: `private val lock = ReentrantReadWriteLock()`.
6. Lines 9–10: replace with imports of `java.util.concurrent.TimeUnit` and `java.util.concurrent.locks.ReentrantReadWriteLock`.

`SVC`, `BE`, `JNI` unchanged. Independent of R3.

## 5. Risks

- **Deadlock:** none found; `close()`'s wait is capped. Callbacks under the read lock only post broadcasts (`K/interop/ProcessorMessageHandler.kt:24`).
- **Main thread:** `close()` may block 2 s, then runs free, which joins EEL script threads. A stuck one means ANR (today it hangs the timer thread unseen; unverified whether a script can).
- **Audio:** one uncontended `tryLock` per block (cost unverified). Shutdown's last blocks pass through unprocessed.
- **Behaviour:** `super.close()` also broadcasts sample rate 0 (`BE:39`); UI effect unverified.
- **Killed during shutdown:** nothing persistent is written.
- **Not fixed:** stuck audio thread beside a new one (`SVC:567`, `:580`); Risk 6 flags.

## 6. Phone test

```
adb logcat -c
adb logcat -s JamesDspWrapper_JNI:* JamesDspLocalEngine:* JamesDspBaseEngine:* libc:F DEBUG:*
```

1. Run R1A test (a).
2. Pick a long impulse response, tap power off at once. 10 times.

Pass, after every off, in order: `Closing engine`, `dtor: memory freed`, then the next `ctor: memory allocated`.

Fail: `engine leaked`, any `libc` or `DEBUG` line, an ANR dialog. Kotlin tags unverified.
