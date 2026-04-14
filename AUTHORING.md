# Authoring template rules

This document describes how to write **behavioral** (Onyx) YAML rules under `behavioral-templates/`. **Nuclei** rules use the standard Nuclei v3 format.

Implementation details live in **onyx**: `internal/templates/schema.go`, `internal/templates/loader.go`, and `internal/detector/engine.go`.

---

## Behavioral templates

### Where to put files

Use one subdirectory by area:

| Directory | Typical use |
|-----------|-------------|
| `behavioral-templates/session/` | Cross-event session patterns (duration, tags, exec/network ordering). |
| `behavioral-templates/file/` | File paths, extensions, mmap-related behavior. |
| `behavioral-templates/process/` | Process name, binary path, command line, AI process flag. |
| `behavioral-templates/network/` | Ports, IPs, HTTP method, protocol. |

Any `.yaml` / `.yml` under the directory passed to **`--behavioral-templates`** is loaded. Restart the monitor after changes. More sub-directories can be added if needed.

### Top-level structure

Each file defines one **template**:

```yaml
id: # should be unique and in snake_case format
info:
  name: # title
  author: # optional
  severity: # info | low | medium | high | critical
  description: >
    # a brief description about the template
  tags: [tag1, tag2]
  risk-score: 10   #  must be > 0 — added when this rule fires

matchers:
  - type: ...
    # fields depend on type (see below)

matchers-condition: and   # optional; default is and — all matchers must pass
                          # use or — any single matcher passing is enough
```

**Requirements (loader validation):**

- `id` — non-empty; becomes a tag on matching events/sessions.
- `info.name` — required.
- `info.risk-score` — required; integer **> 0**.
- At least one matcher; each matcher must have `type` set.

Optional **`negate: true`** on a matcher inverts that matcher’s result before combining with `matchers-condition`.

---

### Matcher type: `event-type`

Exact match on the event’s type string (OR across `values`).

```yaml
- type: event-type
  values: [file_open, file_rw, exec, net_connect]
```

---

### Matcher type: `process`

Use **`field`** to pick which process attribute to test:

| `field` | Meaning |
|---------|---------|
| `comm` | Process name (comparison uses lowercased `comm`). |
| `binary` | Full binary path. |
| `cmdline` | Full command line. |
| `is_ai_process` | Boolean; use `equals: true` or `equals: false`. |

Matching uses `values` (exact), `words` (substring, with `condition: and` / `or`), and/or `regex` — see [shared matching](#shared-matching-within-a-matcher).

---

### Matcher type: `filepath`

Checks `FilePath` on the event. If you configure **multiple** of `extensions`, `words`, `values`, or `regex`, **all** configured groups must pass (AND between groups).

- **`extensions`** — file suffix, e.g. `.gguf`, `.pt` (see loader for normalization).
- **`words`** — substrings; `condition: or` (default) or `and`.
- **`values`** — exact path match.
- **`regex`** — one pattern applied to the full path.

---

### Matcher type: `network`

Requires a network event with `Network` populated. Use **`field`**:

| `field` | Notes |
|---------|--------|
| `dst_port` | `values` for exact port strings, or **`lt`** / **`gt`** for numeric range (on the port number). |
| `dst_ip` | `values`, `words`, `regex` (via shared matching). |
| `http_method` | `values`; compared case-insensitively. |
| `protocol` | `values`; compared case-insensitively. |

For `dst_port` with `lt`/`gt`, see [numeric comparison](#numeric-lt--gt).

---

### Matcher type: `risk-flag`

Session/event risk bitmask. **`flags`** lists names; **all** listed flags must be set (AND).

Supported names: `sensitive`, `large_mmap`, `http`.

```yaml
- type: risk-flag
  flags: [sensitive, http]
```

---

### Matcher type: `session`

Uses **`field`** and the current **session** (nil session → matcher fails).

| `field` | Behavior |
|---------|----------|
| `exec_after_net` | `sess.ExecAfterNet`; optional `equals: true` / `false`. |
| `has_ai_process` | True if any event in the session has `IsAIProcess`; optional `equals:`. |
| `duration_minutes` | Minutes since session start; use **`gt`** / **`lt`** (see below). |
| `last_net_age_seconds` | Seconds since last `net_connect`; **`gt`** / **`lt`**. Fails if no net activity recorded. |
| `has_tag` | `contains: tagname` for one tag, or `values: [tag1, tag2]` if any tag exists in session. |
| `has_exec_comm` | True if any `exec` in session has `comm` in `values` (lowercased match). |
| `exec_binary_match_filepath` | True if some `exec` binary equals current event’s `FilePath`; use `equals: true` / `false` to require match or absence. |
| `other_file_rw` | True if session has another `file_rw` with a different non-empty path than the current event; optional `equals:`. |

---

### Matcher type: `tls-payload`

Inspects TLS payload text on the event. Uses **`words`** (with `condition`) and/or **`regex`**. All configured checks must pass.

---

## Shared matching within a matcher

For types that use `matchField`-style logic (`process` with comm/binary/cmdline, `network` with `dst_ip`, etc.):

- If **`values`**, **`words`**, and/or **`regex`** are all set, **each group that is present must pass** (AND between groups).
- **`words`**: substring match, case-insensitive. **`condition: or`** (default) = any word matches; **`and`** = all words must appear.
- **`regex`**: Go regular expression; compiled at load time — invalid regex fails template load.

---

## Numeric `lt` / `gt`

Some matchers support **`lt`** and **`gt`** (floating-point) on a single numeric value, for example:

```yaml
- type: session
  field: duration_minutes
  gt: 30

- type: network
  field: dst_port
  gt: 1024
  lt: 65535
```

If neither `lt` nor `gt` is set where numeric matching applies, the matcher may not match as intended (see engine behavior for that field).

---

## Nuclei templates

Files under `nuclei-templates/` follow the **Nuclei v3** YAML format (`id`, `info`, `http`, matchers, etc.). They are not loaded by the behavioral engine. Refer to [ProjectDiscovery nuclei-templates](https://github.com/projectdiscovery/nuclei-templates) and Nuclei documentation for authoring HTTP checks.

---

## Checklist before opening a PR

- [ ] Behavioral YAML validates: `id`, `info.name`, `info.risk-score` > 0, at least one matcher with `type`.
- [ ] Matcher `field` values match the engine (see tables above).
- [ ] Regexes are valid RE2/Go regex syntax.
- [ ] File lives under the appropriate `behavioral-templates/<category>/` or `nuclei-templates/<category>/` directory.
