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
  `/docs` (Swagger UI) renders BLANK in-browser here because it loads its JS/CSS from
  `cdn.jsdelivr.net`, which is egress-blocked — the API itself is fine; use
  `curl`/`openapi.json` to confirm endpoints.
- WebUI (Streamlit) on port 8501: `uv run streamlit run ./webui/Main.py --server.address=0.0.0.0 --server.port=8501 --server.headless=true --browser.gatherUsageStats=False`.
  Streamlit's first run prompts for an email on stdin and hangs; this is avoided by a
  `~/.streamlit/credentials.toml` containing `[general]\nemail = ""` (and
  `--server.headless=true`). Verify with `curl http://127.0.0.1:8501/_stcore/health`.

### Tests
- `uv run python -m unittest discover -s test` (see `test/README.md`).
- Several tests reach the network or need provider keys and will skip/fail offline:
  `test_azure_tts_v1` FAILS because the default edge-tts endpoint
  `speech.platform.bing.com:443` is egress-blocked; `test_siliconflow` and
  `test_azure_tts_v2` SKIP without keys. The rest pass.

### End-to-end video generation
- The full WebUI/API flow needs network + keys: an LLM provider (script), Pexels/Pixabay
  (materials), and TTS (`speech.platform.bing.com` for the default edge voice) — all
  currently egress-blocked or key-gated. The core video-assembly engine
  (`app.services.video.combine_videos` + `generate_video`, i.e. moviepy/ffmpeg/Pillow)
  runs fully offline against the local clips in `test/resources/*.png.mp4` and BGM in
  `resource/songs/`, producing a real subtitled MP4 under `storage/tasks/<id>/`.
