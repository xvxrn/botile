# Writing plugins for the exteraGram host family

One `.plugin` file is the whole plugin: plain Python, loaded by Chaquopy at
runtime, no build step. The same file is expected to load in exteraGram,
exteraless and AyuGram; a client-specific API is the exception, not the rule.
Behaviour is verified on exteraGram only.

- **Entry point and metadata.** Exactly one `BasePlugin` subclass plus
  module-level literals: `__id__`, `__name__`, `__description__`, `__author__`,
  `__version__`, `__icon__`, `__app_version__`, `__sdk_version__`,
  `__permissions__`. The host reads them; never compute them.
- **Lifecycle.** `on_plugin_load()` installs, `on_plugin_unload()` unhooks and
  restores. Assume the plugin is reloaded in a live process: no state may leak
  between loads.
- **Hooks.** `self.hook_method(method, handler, priority=50,
  before_filters=[...], after_filters=[...])` returns the unhook handle. Get the
  exact `java.lang.reflect.Method` through `find_class()` +
  `getDeclaredMethods()`, match parameter type names, and call
  `setAccessible(True)`; do not go by argument count.
- **Handlers.** Subclass `MethodHook` and implement `before_hooked_method` /
  `after_hooked_method`; `param.thisObject`, `param.args`, `param.getResult()`.
  Catch every exception inside the handler — a raised Python error can break the
  client's draw path.
- **Filters.** `HookFilter.Condition`, `HookFilter.Or`,
  `HookFilter.ArgumentIsInstanceOf` run in the dispatcher. Anything on a hot path
  (`draw()` and avatar setters run constantly) needs a filter, not a Python-side
  check.
- **Settings.** `create_settings()` returns `ui.settings` entries — `Header`,
  `Switch`, `Custom` with `SimpleSettingFactory(create_view=..., bind_view=...)`;
  persist with `self.get_setting(key, default)` / `self.set_setting(key, value)`.
- **UI from Python.** Native widgets via Chaquopy (`android.widget`,
  `org.telegram.ui.ActionBar.Theme`, `SeekBarView`), `dynamic_proxy(JavaInterface)`
  for callbacks, `android_utils.run_on_ui_thread(fn)` for anything touching views
  off the UI thread.
- **Java interop.** Objects are Chaquopy proxies: normalize with
  `_as_java_class()` before reflection. Colors crossing into Java are signed
  int32 ARGB (`_signed_color`). Keep `int`/`float` widths explicit.
- **Permissions.** Declare `__permissions__ = ["hooks"]` and also verify at
  runtime (`PythonBridge.hasPermission`); if the bridge is missing on a fork,
  fall back to granted rather than staying silent.
- **Host differences.** `app.exteraless.plugins` and similar paths exist on some
  forks only. Import them inside `try/except` so the plugin degrades instead of
  failing to load.
