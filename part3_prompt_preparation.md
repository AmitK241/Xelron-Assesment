# Part 3: Prompt Preparation

## Selected PR: beets #3877 — Web readonly mode

---

## 3.1.1 Repository Context

beets is a music library manager written in Python. The project's main job is keeping a music collection organized — it can import music files, query MusicBrainz to identify albums and tracks, write corrected tags back to the files, and maintain an SQLite database that acts as an index of everything in the library.

The people who use beets are mostly technical music collectors. They want something more controllable than iTunes or a streaming service — they have large collections of FLAC or MP3 files on disk, and they want those files properly tagged and organized without fighting a GUI. beets is run from the command line and configured with a YAML file.

The codebase is designed around a plugin system. Almost every feature beyond the basic importer is a plugin: the web interface, the MPD-compatible server (BPD), the various metadata fetchers, the duplicate finder, and so on. Each plugin inherits from `BeetsPlugin` and can add CLI commands, hook into library events, or expose a configuration schema.

The `web` plugin specifically exposes the beets library over HTTP using Flask. It is a small REST API that lets you query items and albums, stream audio files, and — before this PR — also modify or delete entries. People use it to build simple web frontends or mobile apps that browse their music collection without touching the command line.

The problem domain is personal media management, specifically for people who care deeply about metadata accuracy and want automation rather than manual curation.

---

## 3.1.2 Pull Request Description

This PR adds a `readonly` configuration option to the beets `web` plugin.

Before this change, the web plugin's HTTP API exposed all HTTP methods without restriction. A GET request could read library data and stream audio. A DELETE request could remove an item from the library. A PATCH request could modify item attributes. There was no way to say "I want to browse my music over the network but I do not want anyone — including myself by accident — to be able to delete things through the API."

The PR introduces a `readonly` key in the web plugin's config section. The default is `true`. When readonly is enabled, any incoming request that is not a GET returns HTTP 405 Method Not Allowed. When a user explicitly sets `readonly: false`, DELETE and PATCH requests are allowed through as before.

The key design decision here is that the default changed. Previously the implicit default was "allow everything." After this PR the default is "allow reads only." That is a breaking change for anyone who was using write operations without being aware they needed to configure anything — but it is the right call from a security standpoint. A beets web server running on a home network or a shared machine should not silently allow destructive operations by default.

The behavior change is: GET requests behave identically before and after. DELETE and PATCH requests that previously returned 200 or 404 now return 405 unless the user has set `readonly: false`.

---

## 3.1.3 Acceptance Criteria

✓ When `readonly` is not set in the config, it defaults to `true` and the server rejects non-GET requests with HTTP 405.

✓ When `readonly: false` is set, DELETE and PATCH requests are processed normally as they were before this PR.

✓ When a request is rejected due to readonly mode, the response includes an `Allow` header listing the permitted methods (GET).

✓ GET requests — including item queries, album queries, and audio streaming — work identically regardless of the `readonly` setting.

✓ The `readonly` option is documented in `docs/plugins/web.rst` with its type, default value, and a description of what enabling/disabling it does.

✓ Existing tests for GET-based functionality continue to pass without modification.

✓ New tests cover: (a) readonly mode blocking DELETE, (b) readonly mode blocking PATCH, (c) non-readonly mode allowing DELETE, (d) non-readonly mode allowing PATCH.

---

## 3.1.4 Edge Cases

**1. Requests with custom or unusual HTTP methods (e.g., PUT, OPTIONS, HEAD)**
The implementation must decide whether to block only DELETE and PATCH, or block everything that is not GET. OPTIONS and HEAD are sometimes used by browsers for CORS preflight or caching and are not destructive. The PR should clarify whether these are blocked in readonly mode and what the expected response is.

**2. The `readonly` config key is missing entirely vs. explicitly set to `true`**
These should behave the same way, but only if the default is correctly registered in the plugin's config defaults. If the defaults dictionary is missing the key, accessing `config['web']['readonly'].get(bool)` might raise a `ConfigError` rather than falling back to `True`. This needs to be tested with a config file that has a `[web]` section but no `readonly` key.

**3. Concurrent requests when readonly is toggled at runtime**
beets does not support live config reloading, so this is mostly a non-issue — but if someone sends a PATCH request at the exact moment the server is starting up and the config has not been fully parsed yet, there is a potential window where the readonly check might not be in place. The before-request hook must be registered before the Flask app starts accepting connections.

---

## 3.1.5 Initial Prompt

You are implementing a pull request for the beets project. beets is an open-source music library manager written in Python. It is organized around a plugin system where each feature is a `BeetsPlugin` subclass. The web plugin (`beetsplug/web.py`) exposes the beets music library as an HTTP REST API using Flask.

**What you need to implement:**

Add a `readonly` boolean configuration option to the web plugin. When `readonly` is `true` (and this should be the default), the web server must reject any HTTP request that is not a GET by returning HTTP 405 Method Not Allowed. When `readonly` is `false`, write operations (DELETE, PATCH) should be allowed through as they were before this change.

**Context on the existing code:**

The web plugin is in `beetsplug/web.py`. It uses Flask to define routes. Look at how the plugin registers its config defaults — other plugins use `self.config_defaults` or a `config_defaults` dict. Flask provides a `before_request` decorator on the app object that lets you register a function that runs before every request; this is the right place to add the readonly check rather than modifying each individual route handler.

**Implementation steps:**

1. Add `readonly` with a default of `True` to the web plugin's config defaults.
2. Register a `before_request` function on the Flask app that reads `config['web']['readonly'].get(bool)`. If it is `True` and the request method is not `'GET'`, abort with 405.
3. Make sure the 405 response is clean — Flask's `abort(405)` handles the response format.
4. Update `docs/plugins/web.rst` to document the new option: its name, type (`bool`), default (`yes`), and what it controls.

**Acceptance criteria to satisfy:**

- Readonly mode is on by default.
- GET requests work in both readonly and non-readonly modes.
- DELETE and PATCH return 405 in readonly mode and succeed (or return 404 if the item doesn't exist) in non-readonly mode.
- A config file with no `readonly` key behaves the same as `readonly: yes`.
- The `Allow` header in the 405 response lists `GET`.
- Documentation is updated.

**Edge cases to handle:**

- Non-standard HTTP methods like PUT or OPTIONS: decide what to do with these in readonly mode (most likely block everything except GET).
- Config parsing: if the `readonly` key is absent from the config section, the default must kick in — do not let a missing key cause a `ConfigError`.
- The before-request hook must be registered before the Flask development server starts, not after.

**Testing:**

Add tests to `test/test_web.py`. The existing test setup creates a Flask test client. Add at minimum: a test that sends DELETE to a known item URL in readonly mode and asserts a 405 response, and a test that sends DELETE in non-readonly mode and asserts it is processed. Mirror these for PATCH. Confirm all existing GET-based tests still pass.

Do not modify the behavior of any routes other than adding this gate. The goal is a minimal, clean change: one config key, one before-request hook, updated documentation, and new tests.
