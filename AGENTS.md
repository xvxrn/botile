# AGENTS.md — botile

Single-file plugin for the Telegram client family sharing one Python plugin host:
**exteraGram**, **exteraless**, **AyuGram**. It gives bot avatars a rounded-square
shape with an optional theme-aware frame; other avatars are left alone. Python
loaded by the client's plugin host (Chaquopy), not a standalone app.

Tested on exteraGram only. exteraless and AyuGram use the same plugin API and are
supported targets, but nobody has run Botile on them; treat their support as
unverified.

## Layout

| Path | Contents |
|------|----------|
| `plugin/botile.plugin` | The whole plugin: hooks, settings, drawing. Single file by contract. |
| `scripts/package_release.py` | Validates metadata, builds `dist/Botile-<version>.plugin` + `SHA256SUMS`. |
| `tests/test_state_preservation.py` | Host-side mocks for state restore, settings, drawing, reflection. |
| `.github/workflows/` | CI on `main`, release on `v*.*.*` tags. |
| `.github/preview.webp` | Screenshot shown at the top of both READMEs. |
| `ru/` | Russian README and license; English originals stay at the root. |

## Commands

```sh
python3 -m py_compile plugin/botile.plugin
python3 scripts/package_release.py --check
python3 tests/test_state_preservation.py
python3 scripts/package_release.py --output-dir dist
(cd dist && sha256sum -c SHA256SUMS)
```

## Writing plugins for this host family

One `.plugin` file is the whole plugin: plain Python, loaded by Chaquopy at
runtime, no build step. The same file is expected to load in exteraGram,
exteraless and AyuGram; a client-specific API is the exception, not the rule.

- **Entry point and metadata.** Exactly one `BasePlugin` subclass
  (`BotilePlugin`), plus module-level literals: `__id__`, `__name__`,
  `__description__`, `__author__`, `__version__`, `__icon__`, `__app_version__`,
  `__sdk_version__`, `__permissions__`. The host reads them; never compute them.
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

## Rules

- The `.plugin` file stays at `plugin/botile.plugin`; CI, packaging and tests read
  that exact path.
- Metadata (`__id__`, `__version__`, `__app_version__`, `__sdk_version__`,
  `__permissions__`) must stay literal assignments; `package_release.py` parses
  them with `ast`.
- Hook only the paths listed in the module docstring. Widen a hook only with a
  test proving regular users, groups, channels and custom images are untouched.
- Every applied radius/mark stores its baseline and is restored on clear and
  unload; do not clobber marks set by other code.
- Colors crossing into Java must be signed 32-bit ARGB.
- Keep host-side tests runnable without the client or Android: use the mocks,
  they do not replace a device test.
- Document client support honestly: name every client you claim, and say which
  ones were never run.

## Language

Code, comments, commits and English docs in English. `ru/` is Russian product
documentation; keep it in sync with the English files.
