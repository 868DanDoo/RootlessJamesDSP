# Phase 0 Fix R3 — JNIEnv reused across threads

Checked at `93a16c1`, submodule `1b9f89d`. Nothing edited, built or run. Shorthand as in PHASE0_VERIFY; `H` = `N/libjamesdsp-wrapper/JamesDspWrapper.h`.

## 1. Where the env is used

Declared at `H:9`, saved at `JNI:89` in `alloc` (main thread, `SVC:133`). Read only at `JNI:747`, twice.

| Callback | Where | Env | Thread |
|---|---|---|---|
| `onLiveprogOutput` | `JNI:747` | saved (wrong) | IO worker: compile errors, `@init` (`JNI:618`). Audio thread: `@sample` (`LIB/Effects/liveprogWrapper.c:157`). EEL script thread (`LIB/Effects/eel2/nseel-compiler.c:3536`) |
| `onLiveprogExec` | `JNI:616` | own | IO worker |
| `onLiveprogResult` | `JNI:635` | own | IO worker |
| `onVdcParseError` | `JNI:392` | own | IO worker |

`JArrayList` and `EelVmVariable` hold an env only within one JNI call: fine.

## 2. Fix (A)

`JavaVM*` is not kept: `JNI_OnLoad` ignores it (`JNI:750`). Save it there, delete the saved env, and get the env per call in `receiveLiveprogStdOut`:

- IO worker, audio thread: already inside a JNI call. `GetEnv` returns `JNI_OK`. No attach.
- EEL script thread: `JNI_EDETACHED`. Attach, call, detach in the same call.
- Other callbacks: no change.

Sharing:

- `callbackInterface`: already a global ref (`JNI:88`). Safe.
- `jmethodID`s: not references; valid on any thread while the class stays loaded (the global ref ensures it). Safe.
- `callbackClass` (`JNI:91`): local, never saved. No global class ref needed unless a callback calls `FindClass`.
- `JavaVM*`: safe. `JNIEnv*`: never share.

## 3. What could go wrong

- **Thread exit:** the script thread ends with `pthread_exit` (`nseel-compiler.c:3546`). ART aborts if a thread exits still attached (unverified). Same-call detach covers it.
- **Attach cost:** per `printf`, script thread only. The audio thread pays it if `@sample` calls `ThreadJoin` (`nseel-compiler.c:3598`). Size unverified.
- **Audio thread:** A adds only `GetEnv`. The existing path stays real-time unsafe: each `@sample` `printf` allocates a Java string and broadcasts an Intent (`K/interop/ProcessorMessageHandler.kt:18-31`) under the DSP lock (`LIB/jdspController.c:327-338`).
- **Not fixed:** callback racing `free` (`JNI:162`): Risk 1.

**Safer option (B):** nothing displays this output (`K/activity/MainActivity.kt:651` ignores it; `K/activity/LiveprogEditorActivity.kt:140` handles only results). Do edits 1–2, then replace `JNI:747` with `__android_log_print(ANDROID_LOG_INFO, "LiveProg", "%s", buffer);`. No env, no attach, but `onLiveprogOutput` goes dead.

Recommended: A. Switch to B if test step 3 crackles.

## 4. Edits (A)

1. `H:9`: delete `    JNIEnv* env;`
2. `JNI:89`: delete `    self->env = env;`
3. `JNI:16`: after the `}` closing `extern "C" {`, add `static JavaVM* gJvm = nullptr;`
4. `JNI_OnLoad`: change `(JavaVM *, void *)` to `(JavaVM *vm, void *)`. Add `    gJvm = vm;` as the first body line.
5. Replace the `self->env->CallVoidMethod(...)` line in `receiveLiveprogStdOut` with:

```cpp
    if(gJvm == nullptr) { return; }
    JNIEnv* env = nullptr;
    bool attached = false;
    jint st = gJvm->GetEnv((void**)&env, JNI_VERSION_1_6);
    if(st == JNI_EDETACHED)
    {
        if(gJvm->AttachCurrentThread(&env, nullptr) != JNI_OK)
        {
            LOGE("JamesDspWrapper::receiveLiveprogStdOut: AttachCurrentThread failed");
            return;
        }
        attached = true;
    }
    else if(st != JNI_OK) { return; }
    jstring msg = env->NewStringUTF(buffer);
    if(msg != nullptr)
    {
        env->CallVoidMethod(self->callbackInterface, self->callbackOnLiveprogOutput, msg);
        env->DeleteLocalRef(msg);
    }
    if(env->ExceptionCheck()) { env->ExceptionClear(); }
    if(attached) { gJvm->DetachCurrentThread(); }
```

Keep the braces: `LOGE` is two statements.

## 5. Phone test

Debug build. Script (EEL syntax unverified):

```
@init
printf("init ok\n");
t = ThreadInit("printf(\"thread ok\\n\");");
ThreadCreate(t); ThreadJoin(t);
@sample
n += 1; n % 48000 == 0 ? printf("sample ok\n");
```

```
adb logcat -c
adb shell "logcat | grep -E 'JNI DETECTED|JNIEnv|DetachCurrentThread|AttachCurrentThread failed|Fatal signal'"
```

1. Play music, enable Liveprog, pick the script.
2. Switch scripts and back, 10 times.
3. Listen for one minute.

- Before the fix: `JNI DETECTED ERROR IN APPLICATION` or `Fatal signal 11` (unverified).
- Pass: no output, music plays, no crackle each second.
- Fail: any line.
