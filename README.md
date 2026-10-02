# NeuroFlow

An EEG-driven desktop prototype for **meditation and focus feedback**, developed around a Muse S headband. NeuroFlow connects live signals to adaptive videos, a visual state indicator, focus monitoring and an optional Unity experience, while recording session history locally.

The project combines Python desktop development, physiological signal processing and machine learning. Bernardo's contribution included the classification model, desktop application work and connecting predictions to the feedback shown to the user.

## Application features

- **Meditation and focus sessions** with adaptive nature-video scenes and a visual feedback indicator.
- **Live EEG input through Lab Streaming Layer (LSL)**, with optional accelerometer input for movement checks.
- **Baseline calibration**, signal-quality feedback and recalibration controls.
- **Hybrid feedback:** model output is combined with baseline-relative band-power rules and prediction smoothing.
- **Session history in SQLite**, including predictions, target-state summaries, band metrics, stored EEG and session notes.
- **Unity integration over OSC/UDP**, plus a desktop focus-monitoring mode.
- **Replay/simulation utilities** for developing without an active headset stream.

This is an experimental neurofeedback application. Its states are model/rule outputs associated with recorded task conditions, not clinical measurements of stress, attention or mental health.

## System overview

```text
Muse S / compatible LSL stream
            ↓
EEG worker → signal quality → calibration → filtering / features
            ↓
Saved classifier + baseline-relative band-power rules + smoothing
            ↓
PyQt5 videos / focus feedback / OSC to Unity
            ↓
Local SQLite session history
```

The interface is built with **PyQt5** and **qtmodern**. NumPy/SciPy and BrainFlow support signal processing, scikit-learn/XGBoost support offline model comparison, OpenCV handles video playback, and Matplotlib displays analysis/history plots.

## Signals and classification

### Offline training

`classifier.py` reads session CSVs whose filenames encode subject and condition, using four EEG columns corresponding to **TP9, AF7, AF8 and TP10**, with a nominal **256 Hz** sampling rate.

The current training code uses a fourth-order **0.5–50 Hz band-pass**, a **50 Hz notch**, and **six-second windows with 75% overlap**. It extracts band-power, spatial, spectral and temporal statistics, ratios, changes between windows and moving summaries.

It compares random forest, XGBoost, gradient boosting, RBF SVM and an MLP using **leave-one-subject-out cross-validation** (`LeaveOneGroupOut`, grouped by subject). Scaling for SVM/MLP sits inside the model pipeline. Selection uses mean macro F1, and the chosen model is fitted to the complete dataset and saved with metadata.

The recorded target classes are **`concentrating`, `neutral` and `relaxed`**. The interface's tense/distracted levels are derived feedback states, not additional labelled training classes.

### Desktop feedback

The default model is `dataset_results/gradient_boosting_model.joblib`. The worker resolves an LSL stream of type `EEG`, reads the first four channels and optionally looks for an `Accelerometer` stream (or configured accelerometer channels within a combined stream).

The live path uses a **0.5–30 Hz band-pass**, six-second PSD configuration, a one-second processing timer and a nominal **20-second baseline calibration**. Signal-quality logic considers movement, band powers and electrode-contact indicators. Baseline-relative metrics and model confidence are converted to feedback levels and smoothed before UI updates.

Scenes map to `forest.mp4`, `beach.mp4`, `waterfall.mp4` and occasional thunder feedback. Scene changes have a two-second minimum interval. Unity receives scaled feedback on localhost port **9000**, using `/muse/relaxation` or `/neuroflow/focus` and a scene address `/muse/scene`.

## Recorded evaluation

The repository preserves multiple experiments; they must not be combined into a single benchmark:

| Artifact | What it records |
| --- | --- |
| `dataset_results/classifier_metadata.json` | Gradient boosting: mean accuracy **0.8464**, macro F1 **0.8466**, F1 standard deviation **0.0724**; the training code computes these across held-out-subject folds |
| `dataset_results/complete_classifier_report.txt` | A different report: four subjects, 375 windows, 68 features and 50% overlap; random forest is best at **0.837 ± 0.099** accuracy |
| `dataset_results/windowed_classifier_summary.txt` | A separate 24-window relaxation diagnostic: **0.292** accuracy under a different label setup |

These are historical saved outputs, not a new reproduction or evidence of reliable live mental-state recognition. The current source's 75% overlap differs from the older report's 50%. The separate diagnostic also illustrates why offline model scores cannot substitute for checking the full feedback pipeline.

`calculate_session_level_f1()` predicts on the full feature dataset after fitting the model to it. Its session-level scores are therefore **in-sample**, not held-out-subject validation. Use the window-level grouped CV results when discussing generalisation.

## Run locally

No dependency lockfile or packaged installer is provided. A starting environment based on the application's imports is:

```bash
git clone https://github.com/Bernardo-JG/NeuroFlow.git
cd NeuroFlow
python -m venv .venv
```

Activate the environment, then install:

```bash
python -m pip install PyQt5 qtmodern numpy scipy pandas matplotlib seaborn scikit-learn xgboost joblib pylsl brainflow opencv-python python-osc
python app.py
```

Run from the repository root so model, video and storage paths resolve correctly. Qt multimedia, LSL and BrainFlow require platform-compatible native libraries. Saved scikit-learn models can require the original package versions; this repository does not pin them. Only load model files you trust.

For a fresh local profile, back up and move the existing `app_data` directory outside the checkout before first launch. The database manager creates the directory/schema automatically. Register a local account rather than reusing any saved login/session state.

### Live headset

Start a compatible Muse-to-LSL bridge separately; the application consumes LSL streams and does not itself provide a complete Bluetooth pairing workflow. Confirm stream channel order, units and sampling rate before calibration. Keep only the intended EEG stream active, since the worker uses the first matching stream.

### Replay without a headset

In one terminal, stream an existing recording:

```bash
python lsl_eeg_simulator.py --file test_session_data/eeg_data_raw.npy
```

In another, launch `python app.py`, then choose a session. Replay helps exercise integration; it does not reproduce a real user's response to the feedback. The simulator has recording-duration/sample-rate assumptions that should be checked for other files. `lsl_multistream_simulator.py` provides a separate EEG/accelerometer simulation utility.

The optional Unity path includes a **Windows build** at `game/NeuroFlow.exe`; do not assume that executable works on other operating systems or that a Unity editor project is supplied.

## Retrain / inspect

Extract the available training data to a local directory and update the machine-specific `data_path` inside `classifier.py` before running `python classifier.py`. The loader expects names such as `subjecta-relaxed-1.csv` and uses CSV column positions 1–4 for EEG (zero-based positions, after the first column).

Retraining writes plots, metadata and a joblib model under `dataset_results/`. If another model wins, update the desktop worker's model path and verify feature order, filtering and label decoding before using it for live feedback. The training code label-encodes targets, while the runtime passes model predictions into its feedback logic; the saved artifact and mapping need to remain consistent.

## Code map

| File / directory | Role |
| --- | --- |
| `app.py` | Application shell, navigation and login flow |
| `backend/eeg_processing_worker.py` | LSL acquisition, calibration, features, prediction and session accumulation |
| `backend/signal_quality_validator.py` | Movement, contact and spectral-quality heuristics |
| `backend/database_manager.py` | Local users, sessions, metrics, EEG and notes |
| `ui/unified_page_widget.py` | Meditation/focus orchestration and Unity OSC feedback |
| `ui/video_player_window.py` | Video selection, transitions and feedback indicator |
| `ui/history_page_widget.py` | Session review and plots |
| `classifier.py` | Offline features, model comparison and saved classifier |
| `muse_real_time_gui.py`, `latency_move_test.py` | Separate acquisition/evaluation utilities |
| `assets/`, `game/` | Media and optional Unity Windows build |

## Current limitations

Offline and live filtering differ, so model features are not generated under identical conditions. Model loading/prediction failures fall back to rules; an active interface alone does not prove the ML model is running. The worker enables a testing mode with reduced stability requirements. Quality scores and calibration thresholds are heuristics, and no therapeutic benefit is established by the saved experiments.

Local session storage and remembered login state are part of the prototype, not a production authentication/privacy design. Use fresh test data when demonstrating the project.
