# Phase 0 Apply Plan — R1A, R1, R3 (option B)

Checked at `e39ea12`, submodule `1b9f89d`. Nothing built or run.

`JNI` = `app/src/main/cpp/libjamesdsp-wrapper/JamesDspWrapper.cpp`, `H` = `JamesDspWrapper.h` beside it, `LE` = `app/src/main/java/me/timschneeberger/rootlessjamesdsp/interop/JamesDspLocalEngine.kt`.

## 1. Files and lines

| Plan | File | Lines |
|---|---|---|
| R1A (a) | submodule `jdspController.c` | 1254–1255. **Not applied** (skip) |
| R1A (b) | `JNI` | 621–622 (branch: 620–623 become one blank line, net −3) |
| R3 (B) | `H` | 9 |
| R3 (B) | `JNI` | 89, 747 |
| R1 | `LE` | 9–10, 13, 18, 29–38, 43, 52–54, 59, 68–70, 75, 84–86, 91–178 (15 call lines), 183 |

**Overlap:** (b) and R3 share `JNI` but no lines. After (b), R3's 747 becomes 744. R1 shares nothing.

## 2. Dependencies and conflicts

- No dependencies or conflicts found.
- With B, Liveprog output never enters Java, so R3 and R1 no longer interact.
- B's cost: `onLiveprogOutput` goes dead. Confirmed nothing displays it (`MainActivity.kt:651`). The audio thread still logs once per `@sample` `printf`; cost unverified.
- (b) is safe: `LiveProgStringParser` unlocks on every exit.
- The branch commit message claims added error logging; the diff adds none.

## 3. Order

Unchanged by B: R1A (b), then R1, then R3. R3's test switches scripts, so (b) goes first; R1 removes shutdown crashes that would blur R3's test. Roll back one with `git revert <sha>` (clean revert unverified).

## 4. Checklist

**Setup:** `git checkout master`, `git pull`, `git checkout -b phase0-apply`.

**Commit 1:** `git cherry-pick d91c073`.

**Commit 2, `LE`, in order:**

1. Lines 183, 18: wrap the `JamesDspWrapper.…` call as `guard(Unit) { … }`.
2. Lines 96, 101, 106, 111, 116, 121, 130, 140, 144, 155, 159, 163, 178: `return guard(false) { <existing call> }`.
3. Line 91: `return guard(false) { <both calls and the "and"> }`.
4. Line 173: `return guard(arrayListOf<EelVmVariable>()) { <existing call> }`.
5. Line 75: replace `handle == 0L` with `!tryProcess { <call from line 85> }`. Delete 84–86.
6. Line 59: same, call from line 69. Delete 68–70.
7. Line 43: same, call from line 53. Delete 52–54.
8. Lines 29–38: replace with the Kotlin block in PHASE0_FIX_R1.md §4 step 4, verbatim.
9. Line 13: prefix `@Volatile `. Insert above: `    private val lock = ReentrantReadWriteLock()`.
10. Lines 9–10: replace with `import java.util.concurrent.TimeUnit` and `import java.util.concurrent.locks.ReentrantReadWriteLock`.

**Commit 3 (R3 option B):**

1. `H:9`: delete `    JNIEnv* env;`.
2. `JNI:89`: delete `    self->env = env;`.
3. `JNI`: replace the whole line starting `    self->env->CallVoidMethod(` with:

```cpp
    __android_log_print(ANDROID_LOG_INFO, "LiveProg", "%s", buffer);
```

4. Check: `grep -n "self->env" <JNI>` prints nothing.

Change nothing else: keep the `self == nullptr` block, `callbackOnLiveprogOutput` and `JNI_OnLoad`.

## 5. Tests

Debug build after each commit. Music playing, Convolver and Liveprog on.

```
adb logcat -c
adb shell "logcat | grep -E 'JamesDspWrapper_JNI|LiveProg|Closing engine|has been freed|engine leaked|JNI DETECTED|Fatal signal'"
```

Always fail: `Fatal signal`, `JNI DETECTED`, app closes.

- **Commit 1** (repeat after 2 and 3): switch between two Liveprog scripts 30 times, fast. Fail: lasting silence.
- **Commit 2:** power off, wait 1 s, on; 20 times. Then pick a long impulse response, power off at once; 10 times. Pass, in order: `Closing engine`, `dtor: memory freed`, `Handle … has been freed`, next `ctor: memory allocated`. Fail: `engine leaked`, ANR dialog.
- **Commit 3:** load the script in PHASE0_FIX_R3.md §5 (syntax unverified). Switch away and back 10 times, listen a minute. Pass: `LiveProg` lines `init ok` and `thread ok` after each switch, `sample ok` about once a second, no crackle. Fail: no `LiveProg` lines.

**Unverified:** compiles; runtime behaviour; Kotlin log lines; plugin flavor.
