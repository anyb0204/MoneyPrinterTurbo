# AGENTS.md

## Cursor Cloud specific instructions

MoneyPrinterTurbo generates short videos from a topic. It runs as **two services**: a FastAPI API and a Streamlit WebUI.

### Environment
- Python is managed by **uv** (installed globally). `uv sync` creates a `.venv` using Python 3.11 (per `.python-version`; `pyproject.toml` allows `>=3.11,<3.13`). The startup update script already runs `uv sync`.
- `ffmpeg` (system) and `ImageMagick` (`convert`) are installed; `moviepy`/`imageio-ffmpeg` also bundle an ffmpeg. None are needed to *start* the services — only to actually render videos.
- `config.toml` is auto-created from `config.example.toml` on first import and is gitignored. No API keys are required to start; `pexels_api_keys`/`pixabay_api_keys` + an LLM provider key are only needed to actually generate a video.

### Run (development)
- API (port 8080, docs at `/docs`): `uv run python main.py`
- WebUI (port 8501): `uv run streamlit run ./webui/Main.py --server.address=0.0.0.0 --server.port=8501 --browser.gatherUsageStats=False --server.headless=true`

### Test
- `uv run python -m unittest discover -s test`
- Caveat: some tests hit the network (edge-tts → Microsoft, Pexels/Pixabay) and will **time out** under restricted egress. The offline subset passes, e.g. `uv run python -m unittest test.services.test_video` (exercises the ffmpeg image→video pipeline).
- There is no lint config in this repo.
