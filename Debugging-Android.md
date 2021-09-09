# How to debug Android components

Debugging Android components is a process which tends to depend on the component in question being debugged. In these directions we will cover three methods:
1. Debugging framework or services Java code
2. Debugging native code
3. Debugging native code using `gdb`

## Debugging framework or services Java code

If your desired component is part of the `frameworks/base` Java repository, then follow these steps:

1. Find the faulty line or lines using the [Android Code Search](https://cs.android.com).
2. Add debug logging to the relevant location or locations. For example, you can use the code from Listing 1 to print the current Java stack trace as a warning log.
3. Recompile the component you modified using `make framework` or `make services`.
4. Push the component to your target device and restart it:
   ```
   adb root
   adb remount
   adb push $TARGET_OUT/system/framework/framework.jar /system/framework/framework.jar
   adb reboot
   ```
5. Now observe the log of the problematic behavior and adjust your debugging steps as necessary.

## Debugging native code

If your desired component is part of a C++ repository like `frameworks/native` or `frameworks/av`, then follow these steps:

1. Find the faulty line or lines using the [Android Code Search](https://cs.android.com).
2. Add debug logging to the relevant location or locations. For example, you can use the code from Listing 2 to print the current native stack trace as a warning log.
3. Recompile the component you modified using, for example, `make audioserver`.
4. Push the component to your target device and restart it:
   ```
   adb root
   adb remount
   adb push $TARGET_OUT/system/lib64/your_library.so /system/lib64/your_library.so
   adb reboot
   ```
5. Now observe the log of the problematic behavior and adjust your debugging steps as necessary.

## Debugging native code using `gdb`

In general, system components and Binder interactions can be debugged using `gdbclient.py`. To debug a component using `gdb`, then follow these steps:
1. From your source root, run `gdbclient.py -n audioserver` (to debug a process on the device named `audioserver`).
2. Use the breakpoint commands from Listing 3 to automatically print a stack trace and resume execution when a specific function or line is encountered.
3. Add additional breakpoint or watchpoint commands as desired.

## Additional Listings

### Listing 1 - Tracing Java

```java
StackTraceElement[] frames = Thread.currentThread().getStackTrace();
for (StackTraceElement frame : frames) {
    Log.w(TAG, "    " + frame.toString());
}
```

### Listing 2 - Tracing C++

```cpp
#include <unwind.h>
#include <dlfcn.h>

namespace {

struct BacktraceState
{
    void** current;
    void** end;
};

static _Unwind_Reason_Code unwindCallback(struct _Unwind_Context* context, void* arg)
{
    BacktraceState* state = static_cast<BacktraceState*>(arg);
    uintptr_t pc = _Unwind_GetIP(context);
    if (pc) {
        if (state->current == state->end) {
            return _URC_END_OF_STACK;
        } else {
            *state->current++ = reinterpret_cast<void*>(pc);
        }
    }
    return _URC_NO_REASON;
}

static size_t captureBacktrace(void** buffer, size_t max)
{
    BacktraceState state = {buffer, buffer + max};
    _Unwind_Backtrace(unwindCallback, &state);

    return state.current - buffer;
}

static void dumpBacktrace(void** buffer, size_t count)
{
    for (size_t idx = 0; idx < count; ++idx) {
        const void* addr = buffer[idx];
        const char* symbol = "";

        Dl_info info;
        if (dladdr(addr, &info) && info.dli_sname) {
            symbol = info.dli_sname;
        }

	ALOGW("  #%2d: %p %s\n", idx, addr, symbol);
    }
}

static void backtraceToLog()
{
    const size_t max = 30;
    void* buffer[max];
    dumpBacktrace(buffer, captureBacktrace(buffer, max));
}

}
```

### Listing 3 - `gdb` commands

```
set pagination off
break android::AudioFlinger::AudioFlinger
commands
silent
backtrace
continue
end

```
