# Hybrid C++ addons (UXP Hybrid Plugin SDK)

These facts come from a native audio-analysis addon in production (Windows
x64, Premiere 26.2.x). The SDK is under the Adobe developer terms. Get the
SDK from Adobe. Do not distribute the SDK with your repository.

## Entry points

```cpp
addon_value uxp_addon_init(addon_env env, addon_value exports, addon_apis&& addonAPIs);
void        uxp_addon_terminate(addon_env env);
```

The DLL must export both functions. The `UXP_ADDON_INIT` and
`UXP_ADDON_TERMINATE` macros (in `utilities/UxpAddon.h`) define the
exports. They keep `addonAPIs` in a singleton. The `UxpAddonApis` macro
gives access to the singleton.

## The thread model: incorrect use stops Premiere

- Init and terminate run on the Premiere **main thread**.
- Callbacks from JS run on a **scripting thread**, never on the main
  thread.
- `uxp_addon_schedule_on_main_queue(...)` sends work to the Premiere UI
  thread. **Heavy work here stops the application.**
- `uxp_addon_schedule_on_javascript_queue(...)` sends work to the JS
  queue.
- The `Task` helper in the SDK schedules on the main queue first, then on
  the JS queue. Use it only for short host-API calls.
- **For long work (file I/O, decoding):** start a `std::thread` from the
  JS-thread callback. Do the work on that thread. Then send the result
  back with `uxp_addon_schedule_on_javascript_queue`.

## Return Promises to JS

```cpp
addon_deferred deferred = nullptr;
addon_value promise = nullptr;
UxpAddonApis.uxp_addon_create_promise(env, &deferred, &promise);
// start the async work; keep deferred and env
// later, ON THE JS THREAD:
HandlerScope scope(env);                        // necessary before JS-value work
UxpAddonApis.uxp_addon_resolve_deferred(env, deferred, result);  // or reject_deferred
```

On the JS side, `require("name.uxpaddon")` also returns a Promise. Use
`await` on it. See
[panel-javascript](panel-javascript.md#module-loading).

## Build facts (Windows)

- The output is a DLL with the extension `.uxpaddon`. Put it at
  `win/x64/<name>.uxpaddon` relative to the plugin root. The manifest
  gives the filename only.
- Use the same settings as the SDK sample project: x64, MSVC `v143`,
  C++17, `_UNICODE;UNICODE`, ConformanceMode.
- **Premiere memory-maps the DLL while Premiere runs.** When you close the
  project, Premiere keeps the file lock. Fully close Premiere before a
  rebuild. If you do not, the link step fails with an access error.

## Run external tools. Do not link them.

If you need ffmpeg (or another GPL tool), **run it as a subprocess**
(`CreateProcessA` on Windows). Do not link it. This prevents copyleft
linkage. The tool then becomes an optional run-time dependency. You can
detect it and show a warning, instead of a build-time dependency. Decode
the common formats natively where possible (for example, with the
public-domain single-header libraries `dr_wav` and `dr_mp3`). Then the
external tool is necessary only for the unusual formats.
