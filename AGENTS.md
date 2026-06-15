# AGENTS.md

## Cursor Cloud specific instructions

MoneyPrinterTurbo is a Python app that generates short videos from a subject/script
(LLM script → stock material → TTS narration → subtitles → final MP4). It ships two
runnable services plus a `unittest` test suite. Standard run/test commands live in
`README.md` / `README-en.md` and `test/README.md`; only the non-obvious caveats are
captured here.

### Environment / tooling
- Dependencies are managed with `uv` (lockfile `uv.lock`, Python pinned to `3.11` via
  `.python-version`). The update script installs `uv` and runs `uv sync --frozen`,
  which creates `.venv` (Python 3.11). Run everything through `uv run ...`.
- `uv` is installed via `pip install --user uv` (the official `astral.sh` installer
  script is egress-blocked here; PyPI is reachable). It lives at `~/.local/bin/uv`, so
  ensure `~/.local/bin` is on `PATH` (e.g. `export PATH="$HOME/.local/bin:$PATH"`).
- `ffmpeg` is preinstalled and auto-detected. ImageMagick is NOT required: moviepy 2.x
  renders subtitle `TextClip`s with Pillow + bundled fonts in `resource/fonts/`.
- `config.toml` is auto-created from `config.example.toml` on first config load (and is
  gitignored). No external API keys are needed just to boot the services or run tests.

### Running the services (dev)
- API (FastAPI/uvicorn) on port 8080: `uv run python main.py`. Verify with
  `curl http://127.0.0.1:8080/openapi.json` (or the homepage `/`). The interactive
  `/docs` (Swagger UI) loads its JS/CSS from `cdn.jsdelivr.net`; that domain has been
  added to the egress allowlist, so `/docs` renders. If it ever shows blank again,
  re-check that `cdn.jsdelivr.net` is allowlisted (the API itself works regardless —
  confirm via `curl`/`openapi.json`).
- WebUI (Streamlit) on port 8501: `uv run streamlit run ./webui/Main.py --server.address=0.0.0.0 --server.port=8501 --server.headless=true --browser.gatherUsageStats=False`.
  Streamlit's first run prompts for an email on stdin and hangs; this is avoided by a
  `~/.streamlit/credentials.toml` containing `[general]\nemail = ""` (and
  `--server.headless=true`). Verify with `curl http://127.0.0.1:8501/_stcore/health`.

### Tests
- `uv run python -m unittest discover -s test` (see `test/README.md`).
- `test_siliconflow` and `test_azure_tts_v2` SKIP without provider keys.
- `test_azure_tts_v1` uses the default edge-tts endpoint `speech.platform.bing.com`
  (now egress-allowlisted) and passes. Caveat: edge-tts rate-limits rapid repeated
  calls — the app's sync path then hits its 30s timeout and retries 3x. If TTS starts
  timing out, wait ~1-2 min for the throttle to clear and retry; it is NOT an egress
  block (a single async `edge_tts` stream still connects).

### End-to-end video generation
- TTS (default edge voice via `speech.platform.bing.com`) now works (allowlisted), so the
  narration + subtitle + video-assembly chain runs end to end: `app.services.voice.tts`
  → `create_subtitle` → `app.services.video.combine_videos` → `generate_video`
  (moviepy/ffmpeg/Pillow), producing a real subtitled MP4 under `storage/tasks/<id>/`.
- The remaining two steps of the *full* product flow still need user-supplied API keys:
  an LLM provider (AI script generation) and Pexels/Pixabay (stock materials). Without
  them, supply the script manually and use local clips (`test/resources/*.png.mp4`) +
  BGM (`resource/songs/`) as materials.
