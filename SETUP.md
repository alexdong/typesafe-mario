# Setup and tracing

This guide sets up TypeSafe Mario on Linux and shows how to trace the emulator, state
parser, and Jev decision path. The project requires Python 3.13 or newer.

## Prerequisites

- Python 3.13 or newer
- [`uv`](https://docs.astral.sh/uv/)
- A TypeSafe API key for Jev-backed runs
- VS Code with the Python and Python Debugger extensions for the supplied debug profiles

The repository contains no separately supplied Nintendo ROM. Make sure any game data used
on your machine is obtained and used lawfully.

## Install the project

From the repository root:

```bash
uv venv --python 3.13 .venv
source .venv/bin/activate
uv pip install -e ".[mario,dev]"
```

The editable install maps debugger breakpoints to the Python files under `src/`.

## Configure the TypeSafe key

Create `.env` in the repository root:

```dotenv
TYPESAFE_API_KEY=replace-with-your-key
```

Protect the file on a shared Linux machine:

```bash
chmod 600 .env
```

`.env` is ignored by Git. The VS Code Jev profiles load it automatically. The command-line
program does not load `.env` itself, so export it before a Jev run:

```bash
set -a
source .env
set +a
```

## Verify the installation

Start with the parser-only demo. It does not launch Mario or call TypeSafe:

```bash
source .venv/bin/activate
typesafe-mario state-demo
```

Then run five emulator decisions with the offline heuristic policy:

```bash
SDL_VIDEODRIVER=dummy typesafe-mario play \
  --policy heuristic \
  --display none \
  --max-decisions 5 \
  --frames-per-decision 8
```

Run the project checks:

```bash
python -m pytest -q
ruff check src tests
ruff format --check src tests
```

## Trace it in VS Code

Open the repository:

```bash
code .
```

Open **Run and Debug** and choose one of the supplied profiles:

1. **Trace: state parser demo (no API)** traces the synthetic RAM example.
2. **Trace: Mario + heuristic (5 decisions)** starts the emulator without API requests.
3. **Trace: Mario + Jev (1 API request)** traces one synchronous Jev decision.
4. **Run: Mario + Jev dashboard (50-request cap)** opens the game and telemetry window.

Start with the single-request profile when tracing Jev. In dashboard mode the Jev call runs
on a background thread so the emulator window remains responsive.

Useful first breakpoints:

- `src/typesafe_mario/cli.py`, `main()`: command parsing and policy selection
- `src/typesafe_mario/runner.py`, `create_mario_env()`: emulator construction
- `src/typesafe_mario/runner.py`, `run_episode()`: state, decision, and controller loop
- `src/typesafe_mario/state.py`, `MarioStateParser.parse()`: RAM and telemetry parsing
- `src/typesafe_mario/policy.py`, `TypeSafePolicy.choose()`: questions, API call, and response
- `src/typesafe_mario/runner.py`, `_record_decision()`: JSONL artifact writing

The launch profiles set `justMyCode` to `false`, so **Step Into** can follow calls into
`typesafe-sdk` and the emulator packages.

## Run Mario with Jev

After activating `.venv` and exporting `.env`:

```bash
typesafe-mario play \
  --policy typesafe \
  --display dashboard \
  --max-decisions 50 \
  --frames-per-decision 8
```

Press `Q` or `Esc` to close the dashboard. Press `R` to restart an episode.

Each TypeSafe-policy decision makes one API request containing the Choice, Noul, and Score
judgements. `--max-decisions` caps those requests, so keep it small while debugging.

Completed decisions are written to `artifacts/run-<timestamp>.jsonl`. Each line contains the
model-facing state, debug state, action probabilities, latency, reward, and game outcome.
The `artifacts/` directory is ignored by Git.

## Fork remotes

This checkout uses the following remotes:

```text
origin    https://github.com/alexdong/typesafe-mario.git
upstream  https://github.com/fhshaik/typesafe-mario.git
```

To bring new upstream commits into the fork:

```bash
git fetch upstream
git switch main
git merge --ff-only upstream/main
git push origin main
```
