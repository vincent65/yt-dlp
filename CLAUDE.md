# CLAUDE.md

yt-dlp is a CLI and Python library that downloads video/audio from ~1700 sites.
Almost all work is in **extractors**: one class per site that turns a URL into an "info dict" (metadata plus a list of formats).

## Upstream policy (read first)

yt-dlp bans AI-assisted contributions (`.NO_AI/README.md`, `CONTRIBUTING.md:33`).
This is a personal fork (`origin` = vincent65/yt-dlp); work stays here.
Never open, comment on, or push to issues/PRs at `yt-dlp/yt-dlp`; the `upstream` remote is fetch-only.

## Architecture

- CLI: `yt_dlp/__init__.py:main` parses `options.py` (every flag lives there) into a params dict for `YoutubeDL`.
- `YoutubeDL.py` orchestrates: picks the first extractor whose `_VALID_URL` matches, runs it, selects formats, hands off to `downloader/` (one class per protocol, `PROTOCOL_MAP`), then `postprocessor/` (mostly `ffmpeg.py`).
- `yt_dlp/extractor/`: one module per site. `common.py` has the `InfoExtractor` base; its docstring (lines ~105-579) is the spec for every info-dict field. `generic.py` is the last-resort fallback.
- `extractor/youtube/` is a package (`_video.py`, `_tab.py`, `_base.py`), with challenge solving in `jsc/` and PO tokens in `pot/`.
- `networking/`: request handlers (`_urllib`, `_requests`, `_curlcffi`, `_websockets`). All HTTP goes through `self._download_*` / `ydl.urlopen`, never `requests` directly.
- Helpers: `utils/_utils.py` (`int_or_none`, `url_or_none`, `parse_iso8601`, ...) and `utils/traversal.py` (`traverse_obj`, `require`).

## Commands

Run from the repo root. `hatch` is the project tool; if it is not installed, prefix with `uvx` (`uvx hatch test core`).

- `hatch test core`: offline core suite (~1300 tests, ~3 min). This is CI's required check.
- `hatch test test/test_utils.py`: one file (an argument containing `/` is a path). `hatch test core -k traverse` filters by name.
- `hatch test test/test_all_urls.py`: offline; fails if any `_TESTS` URL matches more than one extractor. Run after touching any `_VALID_URL`.
- `hatch test FooIE`, `hatch test Foo_1`, `hatch test Foo_all`: download tests for `FooIE._TESTS[0]`, `[1]`, all. These hit the live site.
- `hatch fmt --check`: CI lint (`ruff check` + `autopep8 --diff`). `hatch fmt yt_dlp/extractor/foo.py` applies fixes to one file; bare `hatch fmt` rewrites the whole tree. Ignore its deprecation notice.
- `hatch run yt-dlp -v -s URL`: run from source with `-Werror -Xdev`, simulate only.

## Extractor conventions

- New site: `class FooIE(InfoExtractor)` in `yt_dlp/extractor/foo.py`, then add the import to `extractor/_extractors.py` (alphabetical by module). Without that import it never loads.
- `_VALID_URL` must have a `(?P<id>...)` group; get it with `self._match_id(url)`.
- Mandatory fields: `id` (a `str`), `title`, and `url` or `formats`. Everything else must be non-fatal.
- Download tests turn every `report_warning` into a failure, so optional lookups use `default=None` (silent), not `fatal=False` (warns).
- Every non-None field returned must appear in the test's `info_dict`; the failure output prints the missing ones to copy. Values can be literals, a type (`int`), `'md5:...'`, `'re:...'`, or `'count:N'`. URL-only tests use `'only_matching': True`.
- Nested data: `traverse_obj(data, {'title': ('meta', 'title', {str}), 'duration': ('len', {int_or_none})})`, imported from `..utils.traversal`. Not `.get()` chains, `try_get`, or `dict_get`.
- Lint rules that only apply under `yt_dlp/extractor/`: no `from re import ...` for stdlib modules (write `import urllib.parse`, `import datetime as dt`), no `print`. `compat_*` helpers are banned everywhere. No `# noqa`.
- Commit titles: `[ie/foo] Fix extraction`, `[ie/foo] Add extractor`, `[rh:requests] ...`, `[pp/FFmpegMetadata] ...`, `[test] ...`. The `[foo]` form shown in CONTRIBUTING is outdated.

## Guardrails

- Never run `hatch check fmt`, `ruff format`, or `black`: the formatter is autopep8, and these reformat ~1100 files.
- Never run plain `pytest` or `make test`: they collect 7132 tests, 5840 of them network download tests, without `-Werror`. Use `hatch test core`.
- `pip install -e ".[default,dev]"` from CONTRIBUTING silently skips the dev tools (`dev` is a dependency group, not an extra). Use hatch.
- `yt_dlp/extractor/lazy_extractors.py` is generated and gitignored. If it exists, it replaces `_extractors.py` and new or changed extractors are invisible: delete it or set `YTDLP_NO_LAZY_EXTRACTORS=1`.
- Owned by the release bot, do not edit: `yt_dlp/version.py`, `Changelog.md`, `CONTRIBUTORS`, `supportedsites.md`, the README options block (generated from `options.py` help text), `.github/ISSUE_TEMPLATE/` (edit `ISSUE_TEMPLATE_tmpl/`), `uv.lock`, `bundle/requirements/`.
- `extractor/youtube/jsc/_builtin/vendor/` is vendored from yt-dlp/ejs and hash-checked at runtime; edits break YouTube.
- Don't run `make`, `make doc`, or `make clean`: they rewrite committed docs or `rm -rf` artifacts.
- Download tests write media and `.info.json` files into the working directory; under `test/` those are not gitignored. Credentials go only in the gitignored `test/local_parameters.json`.
- On macOS, two core tests fail for environment reasons: `test_get_netrc_login_info` (hatch env path contains a space) and `test_cookie_behavior[CurlFD]` (needs `127.0.0.2`). Not regressions.
