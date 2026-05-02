# Dev Environment Guard Proposal

Use this note when applying the dev-mode hardening later.

## Goal

Keep the local watchdog/auto-reload workflow, but prevent Flask debug/watchdog mode from accidentally activating in production if this code is merged to `main`.

## Current Risk

Current dev mode is enabled by:

```python
dev_mode = _env_flag("NEWGAMESAVE_DEV") or _env_flag("FLASK_DEBUG")
```

That is convenient locally, but if a production environment accidentally sets either variable, Flask debug mode and the reloader could run in production.

## Applied Change

Gate dev mode behind an explicit local environment check.

Implemented helper:

```python
def _is_local_environment():
    hosted_environment = any(
        os.environ.get(name)
        for name in ("RAILWAY_ENVIRONMENT", "RENDER_SERVICE_NAME", "FLY_APP_NAME")
    )
    if hosted_environment:
        return False

    env = os.environ.get("APP_ENV", "").strip().lower()
    return env in {"", "local", "development", "dev"}
```

Then replace:

```python
dev_mode = _env_flag("NEWGAMESAVE_DEV") or _env_flag("FLASK_DEBUG")
```

with:

```python
dev_mode = _is_local_environment() and (
    _env_flag("NEWGAMESAVE_DEV") or _env_flag("FLASK_DEBUG")
)
```

Also update `create_app()` to use the same guarded condition:

```python
if _is_local_environment() and (_env_flag("NEWGAMESAVE_DEV") or _env_flag("FLASK_DEBUG")):
    app.config["TEMPLATES_AUTO_RELOAD"] = True
    app.jinja_env.auto_reload = True
```

## Expected Behavior

- Local terminal with `NEWGAMESAVE_DEV=1`: debug/watchdog enabled.
- `dev.bat`: debug/watchdog enabled locally.
- Production on Railway/Render/Fly: debug/watchdog disabled, even if `APP_ENV`, `NEWGAMESAVE_DEV`, or `FLASK_DEBUG` are accidentally set.
- Gunicorn importing `app:app`: unchanged.

## Optional Cleanup

To avoid repeating the condition, create one helper:

```python
def _dev_mode_enabled():
    return _is_local_environment() and (
        _env_flag("NEWGAMESAVE_DEV") or _env_flag("FLASK_DEBUG")
    )
```

Then use:

```python
if _dev_mode_enabled():
    ...
```

and:

```python
dev_mode = _dev_mode_enabled()
```
