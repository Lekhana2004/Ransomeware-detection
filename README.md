# Ransomware Behaviour Detection - Gradient-Descent Anomaly Model

One project that satisfies **both** course requirements:

| Requirement | Where it is met |
|---|---|
| **CIE3 Mini-Project** - *create and simulate a cyberattack with preventive technologies* | `simulation/sandbox.py` (safe ransomware simulator) -> `engine.py` + `monitoring/` (detection) -> `response/alert_manager.py` (stop + isolate + log) |
| **Optimisation Project** - *real-world application of Gradient Descent / Ascent* (problem, maths, code, results, appendix) | `models/gd_logistic.py` (logistic regression trained by gradient descent, written from scratch in NumPy), `analysis.py` (convergence / learning-rate / path figures), report appendix |

## Safety
* **No real malware, no real encryption.** The "attack" overwrites *dummy files created by this project* with random bytes (same entropy signature as ciphertext) and renames them `*.locked`.
* All simulator and response code calls `assert_sandbox()`: the folder must be inside `./sandbox/` **and** contain the `.SANDBOX_OK` marker. Anywhere else it refuses to run (`tests/test_safety.py`).
* A shared `threading.Event` is the kill switch.
* Run it only on your own machine / VM. Do not point it at real folders.

## Install & run
```bash
python -m venv .venv && source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt

python -m models.train_model     # 1. generate data, train 4 detectors, print metrics
python analysis.py               # 2. write figures to reports/figures/
python run_demo.py               # 3. headless live demo (normal -> attack -> detect -> respond)
python experiments.py            # 4. repeat live trials (attack-speed sweep + false-alarm runs)
streamlit run app.py             # 5. interactive dashboard
python -m pytest -q              # 14 tests (gradient check, safety guard, features, monitor)
python selftest.py               # health check for YOUR machine: PASS/FAIL per check
```

## Verify on your own machine (recommended first step)
```bash
python selftest.py     # expect: 9/9 checks passed
```
It checks that the monitor sees every write/rename of a fast attack, that attacks at 5, 30 and 60 files/s
are detected and stopped, that all response actions complete, and that normal editing raises no alert.

## Monitoring backend
`config.MONITOR_BACKEND = "snapshot"` (default) lists the folder every 50 ms and tracks each file by inode, so
a write followed by an immediate rename cannot be hidden and behaviour is identical on Linux, macOS and Windows.
`"watchdog"` uses the OS event API (inotify / FSEvents / ReadDirectoryChangesW); delivery timing and event
coalescing differ per OS, which is why it is not the default. Set `MONITOR_BACKEND = "watchdog"` to compare.

## How it works
```
 sandbox/test_folder  --monitor-->  events (created/modified/moved/deleted + entropy)
        |                                  |
        |                       1-second sliding window -> 10 behavioural features
        |                                  |
        |            +---------------------+----------------------+
        |            |  GD logistic (core) | RF | IsolationForest | mu+3sigma
        |            +---------------------+----------------------+
        |                                  | risk = 100 * P(ransomware)
        |                    P >= 0.7 for 2 consecutive 0.25 s ticks
        v                                  v
   kill switch  <------  alert_manager: stop -> make folder read-only -> log (logs/alerts.jsonl)
```
Features (`config.FEATURE_NAMES`): modify/rename/create/delete rate, extension-change rate, distinct files touched, mean write entropy, fraction of high-entropy writes, KB written, CPU %.

## Honest limitations
* Training data is **synthetic** (documented scenario generator); results show the method works on those assumptions, not that it catches real malware.
* The monitor works on file events inside one folder; real ransomware also uses shadow-copy deletion, network shares, etc.
* On Linux as `root`, `chmod` quarantine is not enforced - the kill switch is the primary stop; on Windows `chmod` only toggles read-only flags.
