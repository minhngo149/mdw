# File Security

Path traversal, file uploads, downloads, and archive extraction. Extends step 12 of `SKILL.md`. The
data-flow method is the same as `injection.md`: is the path/content attacker-controlled, and is it
confined before the file operation?

## Contents

1. Path traversal
2. File uploads
3. Downloads and serving
4. Archive extraction

## 1. Path traversal

A user-controlled value used to build a filesystem path lets an attacker escape the intended
directory. Find the sinks: file open/read/write/delete, `ServeFile`/`sendFile`, template loading,
downloads, and includes.

```text
Vulnerable:  filepath.Join(baseDir, r.URL.Query().Get("name"))   // name = "../../etc/passwd"
             open(base + "/" + user_filename)
```

`filepath.Join`/`path.join` **clean** the path but do **not** confine it: `Join(base, "../../x")`
resolves outside `base`. Check for real confinement:

| Control | Effective? |
|---|---|
| Reject `..`, `/`, null bytes after decoding | partial; watch for encoded (`%2e%2e`), double-encoded, and backslash variants |
| `filepath.Base(name)` — strip to the last element | confines to the directory, if applied to the whole user part |
| Resolve then verify prefix: canonicalize (`filepath.Abs`/`EvalSymlinks`) and check it is still under base | strongest; also defeats symlink escape |
| An allowlist / an id mapped to a server-known path | strongest; the user never supplies a path |

**Framework caveat, verify before trusting:** some file servers guard against `..` only in the URL
*path*, not in a value you pass them. For example Go's `http.ServeFile` cleans `r.URL.Path` but will
happily serve whatever absolute/relative path you hand it as the `name` argument — so
`http.ServeFile(w, r, filepath.Join(dir, queryParam))` is **not** protected by ServeFile's own check.
State the confinement mechanism you actually see; do not assume the framework confines it.

Also watch: absolute paths supplied by the user (overriding the base), symlinks in the served
directory, and case-insensitive filesystems.

## 2. File uploads

For each upload, check the controls actually present:

| Control | Do not trust | Do |
|---|---|---|
| Size limit | — | a request/body size cap (e.g. `MaxBytesReader`) before reading fully |
| Type validation | `Content-Type` header, file extension, client filename | magic-byte / content sniffing of the actual bytes; still constrain to an allowlist of types |
| Filename | the client-provided filename | generate the stored name server-side (id-based); never build the path from the client name |
| Storage location | — | outside the web root and outside any directory that is executed/served as code |
| Execution | — | stored files must not be executable or interpretable (no upload into a PHP/JSP/CGI-served dir) |
| Post-processing | — | if an image/media processor runs on the file, note the processor and that untrusted input reaches it (its own vulns are a dependency/observation, not command injection when passed as an argument — `injection.md`) |
| Public access | — | are uploads served publicly and enumerable? |

A common **safe** pattern to recognize and not over-flag: size-limited upload + magic-byte type check
+ server-generated filename + storage outside web root. Confirm each part is present before calling
it safe.

## 3. Downloads and serving

Serving stored files back is where an upload weakness or a traversal becomes exploitable. Check that
the served path is confined (section 1), that authorization is applied (a user may only download
their own files — otherwise it is an IDOR, `authorization.md`), and that content-type and
`Content-Disposition` are set so a served file is not rendered as active content in the browser
(reflected file download / stored XSS via uploaded HTML/SVG).

## 4. Archive extraction

Extracting user-supplied archives (zip, tar) has two classic bugs:

- **Zip-slip / path traversal**: an entry name like `../../etc/cron.d/x` writes outside the extract
  directory. Confirm each entry's resolved path is verified to stay under the target dir before
  writing.
- **Zip bomb / resource exhaustion**: highly compressed entries or many entries exhaust disk/memory.
  Confirm limits on total size, per-entry size, and entry count.

Report these only when the repo extracts archives from an untrusted source; name the extraction call
and the missing check.
