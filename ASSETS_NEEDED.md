# Assets needed

**Short answer: no media assets. Clone, `npm install`, `npm start` — it boots and
serves with zero art, audio, or fonts supplied.**

VideoTub tracks **24 files, all text**. There is not one image, font, audio, or video
file in the repo (proven by an extension sweep of the whole tree excluding
`node_modules/`, which returns 0 matches; `git ls-files node_modules` is empty). The UI
is styled entirely with CSS and a system-font stack, and it is **CDN-free** — every
`fetch()` is a same-origin `/api/*` path, and the CSP at `server.js:345-348`
(`default-src 'self'; img-src 'self' data:; script-src 'self'`) would block an external
asset even if one were added.

**There are no unguarded load sites.** Every `fs.read*` in first-party code is either
guarded or targets a file the app just created:

- `server.js:130` `fs.readFileSync(file, "utf8")` — inside `loadList()`, in a
  `try/catch` returning `[]`; targets optional denylists that don't ship
- `media.js:79` `fs.readFileSync(tmp)` — guarded by `media.js:78`
  (`if (r.code !== 0 || !fs.existsSync(tmp)) return "";`)
- `server.js:256-258` `fs.openSync`/`readSync` for magic-byte sniffing — `try/catch`,
  `finally` closes the fd; target is the just-uploaded temp file
- `server.js:590` `res.sendFile(public/index.html)` — tracked and present

No `require`/`import` of any media or asset file exists anywhere.

| path/pattern | type | format | dimensions | used for | required/optional | fallback behavior |
|---|---|---|---|---|---|---|
| — | — | — | — | *no repo media assets exist* | — | — |

The distinction that matters for this project is between three categories that are easy
to confuse. Only the first would ever live in git.

---

## (a) Repo assets — none

Notably absent, and deliberately so:

- **No favicon.** No `favicon.ico`/`.png` and no `<link rel="icon">` in any of
  `public/index.html`, `public/watch.html`, `public/admin.html`. Side effect worth
  knowing: a browser's automatic `GET /favicon.ico` misses `express.static`
  (`server.js:353`) and falls through to the SPA catch-all at `server.js:590`, so it
  returns **200 with `index.html`** rather than 404. Cosmetic only.
- **No logo.** The brand is text: `<h1>VideoTub</h1>` (`public/index.html:11`).
- **No placeholder/poster image.** The no-thumbnail fallback is CSS plus a Unicode
  glyph — see (c).
- **No webfonts.** System stack only: `public/style.css:17`
  `font-family: "Segoe UI", Tahoma, Geneva, Verdana, sans-serif;` and
  `public/style.css:184` `font-family: monospace;`. No `@font-face` anywhere.
- **No CSS images.** Zero `url(` in `public/style.css`; zero `background-image`.
  Backgrounds are colours and one gradient (`public/style.css:18`
  `radial-gradient(circle at top, #1c2331 0%, #0d1117 42%)`).

The only `<img>` tags in the codebase point at **runtime-generated** thumbnails, never
a repo file: `public/app.js:130` and `public/admin.js:45`.

*(The four PNGs that exist under `node_modules/` belong to `jest` and
`istanbul-reports` HTML output — third-party test tooling, gitignored, not served.)*

---

## (b) Runtime binaries and services — install these, don't look for files

| dependency | how it's found | required? | failure mode if absent |
|---|---|---|---|
| **Node 18+** | — | **required** | won't run |
| **ffmpeg** | bare name on `PATH` (`media.js:13` `spawnSync("ffmpeg", ...)`, spawns at `media.js:42, 53, 73`). No bundled path, **no env var to override** | optional | **silent degradation, no crash**: thumbnails `return null` (`media.js:41`), transcode `return null` and the original file is kept (`media.js:52`, `server.js:465-471`), perceptual hash `return ""` so near-duplicate blocking is off (`media.js:71`) |
| **ffprobe** | bare name on `PATH` (`server.js:276`) | optional | **fails open** — `server.js:270-279` resolves `{ ok: true, available: false }`, skipping decode validation and the `MAX_DURATION_SEC` cap. Magic-byte validation (`server.js:255-265`) still applies. `FFPROBE_DISABLED=1` forces the same skip |
| **ClamAV** (`clamdscan` or `clamscan`) | `PATH` probe in `resolveScanner()`, or an explicit `CLAMSCAN_PATH` | preferred scanner; one scanner required unless you opt out | see below |
| **Windows Defender** `MpCmdRun.exe` | **hardcoded absolute paths**, not `PATH` — `absoluteDefenderPath()` checks `C:\Program Files\Windows Defender\`, `C:\Program Files\Microsoft Defender\`, and versioned `C:\ProgramData\...\Platform` dirs | fallback scanner (ClamAV wins if both are present) | see below |
| **Python + `nudenet`** | via `CONTENT_SCAN_CMD` env var (`server.js:52`), reference impl `tools/nsfw_scan.py` | optional, off by default | if the var is empty the scan is skipped (`server.js:291-300`); if set, spawn error or non-zero exit **rejects the upload** (`server.js:452-453`) |
| **SQLite** (`better-sqlite3`) | npm native addon; the `.db` **file is created, not shipped** — `db.js:11-13` `mkdirSync` then `new Database(data/videotub.db)`, schema via `CREATE TABLE IF NOT EXISTS` | required | needs a prebuild or a toolchain; `Dockerfile:4` installs `python3 make g++` for this |

**With no malware scanner present, every upload is rejected** — the server still boots
and browses fine, so this used to be a silent surprise: a bare `HTTP 400` per upload and
a boot line that claimed `malware scanner: Windows Defender (if present)` either way.

That is now explicit. `resolveScanner()` runs once at boot and the outcome is printed in
the startup banner; when nothing is available the banner carries a `!!` warning to stderr
saying every upload will be refused, plus how to fix it. The refusal itself is
**HTTP 503** (a server-side capability gap, not a bad request) and its body carries
`scanner`, `details`, and a `remedy` string, which the upload UI shows to the user.

Fail-closed is the default and is never relaxed implicitly. To accept uploads without a
scanner you must say so: `MALWARE_SCAN=off` skips scanning entirely, or
`ALLOW_UNSCANNED_UPLOADS=true` stores files the scanner could not check. Each prints its
own `!!` boot warning. See the **Malware scanning** table in the README.

`compose.yml:21-27` documents a commented-out `clamav/clamav:latest` sidecar;
`Dockerfile:13` installs ffmpeg, so the Docker path gets media handling for free.
Neither Python nor `nudenet` is in the image (`compose.yml:12` notes this).

No Redis, no external DB server, no queue, no cloud SDK. Runtime deps are only
`better-sqlite3`, `express`, `multer`.

---

## (c) User-uploaded content and its derivatives — gitignored, created at boot

`.gitignore` (7 lines):

```
node_modules/
data/
tmp/
videos/
thumbs/
__pycache__/
*.pyc
```

Note the four runtime directories are **siblings**, not all under `data/`
(`server.js:25-28`). All four are auto-created at startup — `server.js:61-63`
`for (const dir of [DATA_DIR, TMP_DIR, VIDEO_DIR, THUMB_DIR]) fs.mkdirSync(dir, { recursive: true })`,
then `db.js` init at `server.js:64`. None exist in a fresh clone; you never create them
by hand. `.dockerignore` mirrors the list, and `compose.yml:16-18` maps `./data`,
`./videos`, `./thumbs` as volumes.

| directory | contents | shipped? |
|---|---|---|
| `videos/` | uploaded video files (served by `express.static`, `server.js:356`) | no — user content |
| `thumbs/` | `<id>.jpg` poster frames **generated by ffmpeg** (served at `server.js:357`) | no — derived at upload |
| `tmp/` | in-flight upload staging | no |
| `data/` | `videotub.db` (SQLite, WAL), optional denylists, `moderation.log` | no — created at boot |

**Thumbnails are generated, never supplied.** `media.js:40-47` `makeThumbnail()` runs
`ffmpeg -ss 1 -i <video> -frames:v 1 -vf scale=320:-1 -y <out>`, i.e. a poster frame
~1 s in, scaled to **320 px wide, height auto**. Called per upload at
`server.js:477-480`; the name is recorded only if the file materialised.

**The no-thumbnail fallback is not an image** — so there is no placeholder file to hunt
for and nothing to flag as missing:

- `public/app.js:131` — `<div class="thumb noThumb">▶</div>` (U+25B6, 32 px via
  `public/style.css:172`, black box via `:171`)
- `public/admin.js:45` — `<div class="noThumb">no thumb</div>`
- `public/watch.js:47` — `if (video.thumbUrl) player.poster = video.thumbUrl;` — the
  poster is simply not set; the `<video>` shows CSS `background: black`
  (`public/style.css:146`)
- embed player `server.js:402, 406` — the `poster` attribute is omitted entirely when
  `thumb_name` is empty

Every image reference in the app is conditional on a runtime-generated file existing.

Optional denylist files (`HASH_BLOCKLIST_FILE`, `WORD_BLOCKLIST_FILE`,
`IP_BANLIST_FILE`, `PHASH_BLOCKLIST_FILE`, `server.js:42-45`) are read defensively by
`loadList()` (`server.js:128-131`) — a missing file yields an empty list.

---

## Configuration, not assets

- `PORT` — `server.js:16` `Number(process.env.PORT || 3000)`, bound at `server.js:596`
  behind `if (require.main === module)` (`:595`) so tests import without listening.
  `Dockerfile:20-21` sets 3000; `compose.yml:4-5` publishes `3000:3000`. Caveat: the
  value isn't validated — a non-numeric `PORT` becomes `NaN`, which Node treats as
  port 0 (random port).
- `ADMIN_KEY` — must be set to use `/admin.html`.
- `VIDEOTUB_RUNTIME_DIR` — relocates the runtime root (`server.js:24`).

The README's prerequisite list contains **no "download or place a file" step of any
kind**, and the README itself has no embedded images or badges.
