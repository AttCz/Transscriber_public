# Transscriptor

Transscriptor is a Windows-focused transcription utility for meeting recordings. It uses OpenAI Whisper for speech-to-text, can optionally call an LLM backend to generate meeting minutes and file-name tags, and supports four operation modes:

- `cmd`: batch or interactive processing from the command line
- `gui`: drag-and-drop desktop UI for one-off transcription
- `watch`: unattended processing of a watched folder
- `dictate`: microphone dictation directly to clipboard

The main entry point is `create_transcript.py`. The application is designed to run either from source or as a PyInstaller-built `.exe` placed next to `settings.yaml`, `auth.yaml`, and `ffmpeg.exe`.

## Quick Start

If you just want to get the app running with the least amount of setup:

1. Install Python and the packages from `requirements.txt`.
2. Copy `settings.example.yaml` to `settings.yaml`.
3. Copy `auth.example.yaml` to `auth.yaml` if you want automatic meeting-minutes generation.
4. Put `ffmpeg.exe` next to `create_transcript.py`.
5. Open `settings.yaml` and choose one mode:
  - `cmd` for full processing from the console
  - `gui` for drag-and-drop transcription
  - `watch` for automatic folder monitoring
  - `dictate` for microphone-to-clipboard dictation
6. Run:

```powershell
python .\create_transcript.py
```

If you do not have valid `auth.yaml` credentials, `gui` and `dictate` are the easiest modes to start with because they do not depend on the LLM backend.

## Which Mode Should I Use?

- Use `dictate` if you want to press a button, speak, and paste the transcription somewhere else.
- Use `gui` if you have one recording file and want a simple desktop window instead of console prompts.
- Use `cmd` if you want the full pipeline: transcript, optional rename, optional meeting minutes, and HTML output.
- Use `watch` if another process or person drops files into a folder and you want Transscriptor to process them automatically.

## What The Application Does

Depending on the mode, Transscriptor can:

- transcribe `.mp4`, `.mp3`, and `.m4a` files with Whisper
- extract audio from `.mp4` via `ffmpeg.exe`
- build `.srt` subtitle output from Whisper text segments
- optionally ask an LLM to generate meeting minutes
- optionally ask an LLM to create a short file rename tag
- convert markdown notes into Outlook-friendly HTML
- watch a network or local folder for new recordings
- record microphone audio for immediate dictation
- copy either prompts or dictated text to the clipboard

## Repository Contents

Important files in this repository:

- `create_transcript.py`: main application
- `ask_db_functions.py`: helper for bearer token handling and LLM calls
- `ask_db.py`: small LLM helper usage example
- `ask_db_for_support.py`: support and attachment upload example script
- `settings.example.yaml`: example runtime settings
- `auth.example.yaml`: example authentication settings for the LLM backend
- `check_whisper_gpu.py`: environment check for Torch/Whisper GPU usage
- `create_transcript.spec`: PyInstaller build definition
- `requirements.txt`: Python package dependencies
- `ffmpeg.exe`: required local binary for extracting audio from `.mp4`

## Supported Inputs

Supported media extensions:

- `.mp4`
- `.mp3`
- `.m4a`

Default language handling:

- `hu` and `en` are the intended primary languages
- in `watch` mode, filenames containing `hun` force Hungarian
- in `watch` mode, filenames containing `eng` force English
- otherwise the default `language` from `settings.yaml` is used

## Requirements

Target environment:

- Windows
- Python 3.11 recommended
- local `ffmpeg.exe` available next to the script or bundled executable
- network access to the configured LLM backend if you use meeting-minutes generation or file renaming

Python dependencies are listed in `requirements.txt`:

- `openai-whisper`
- `torch`
- `tqdm`
- `numpy`
- `customtkinter`
- `tkinterdnd2`
- `sounddevice`
- `lameenc`
- `markdown`
- `bleach`
- `PyYAML`
- `requests`

## Setup On A New PC (Fast Path)

To clone and run the source on a fresh Windows machine:

```powershell
git clone https://github.com/AttCz/Transscriber.git "Transscriptor"
cd .\Transscriptor
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
Copy-Item settings.example.yaml settings.yaml
Copy-Item auth.example.yaml auth.yaml
# put ffmpeg.exe next to create_transcript.py
python .\create_transcript.py
```

If you need GPU acceleration and the local CUDA wheel is shipped in `whl\`:

```powershell
pip install .\whl\torch-2.10.0+cu128-cp311-cp311-win_amd64.whl
```

The `.venv` folder is path-bound after creation. If you copy a workspace that already contains a `.venv` from another path, do not reuse it; recreate it as shown above (otherwise launchers like `pyinstaller.exe` will fail with "Unable to create process").

## Installation From Source

### 1. Create and activate a virtual environment

PowerShell:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
```

### 2. Install dependencies

```powershell
pip install -r requirements.txt
```

If you want to use the local CUDA wheel shipped in `whl`, install it explicitly before or instead of a default Torch install, for example:

```powershell
pip install .\whl\torch-2.10.0+cu128-cp311-cp311-win_amd64.whl
```

If you change Torch manually, re-run Whisper/GPU verification afterward.

### 3. Prepare configuration files

Create local runtime files by copying the examples:

```powershell
Copy-Item settings.example.yaml settings.yaml
Copy-Item auth.example.yaml auth.yaml
```

Then edit both files with your real values.

If you only want local transcription and do not need automatic meeting-minutes generation, you can leave `auth.yaml` unconfigured and use `gui` or `dictate` mode.

### 4. Ensure `ffmpeg.exe` is present

`create_transcript.py` looks for `ffmpeg.exe`:

- first next to the script or bundled executable
- then in `C:\TOOLS\Transcriptor\ffmpeg.exe` as a fallback

Without `ffmpeg.exe`, `.mp4` inputs cannot be converted to audio for transcription.

## Configuration

### `settings.yaml`

Example:

```yaml
records_folder: "C:/path/to/local/records"
model_name: "large"
language: "hu"
q_char_limit: 99000
operation_mode: cmd
watchfolder: "//server/share/path/to/Transscriptor_Queue"
watch_interval_seconds: 20
mm_question_framing: "I have this transscript of a meeting: {X} Please do meeting minutes in the language used during the meeting."
file_rename_framing: "I have this transscript of a meeting: {X} Please summarize the meeting in max 140 characters."
```

Field reference:

- `records_folder`: folder used by `cmd` mode and the GUI file picker
- `model_name`: Whisper model name. Supported values include `tiny`, `base`, `small`, `medium`, `large`, and `turbo`
- `language`: default transcription language, typically `hu` or `en`
- `q_char_limit`: max size before prompts are reduced from full SRT to dialog-only text
- `operation_mode`: `cmd`, `gui`, `watch`, or `dictate`
- `watchfolder`: folder monitored in `watch` mode
- `watch_interval_seconds`: poll interval for `watch` mode
- `mm_question_framing`: prompt template for meeting-minutes generation
- `file_rename_framing`: prompt template for generating a short filename suffix

Prompt templates are expected to contain `{X}` where the transcript content should be injected.

### `auth.yaml`

Example:

```yaml
auth:
  tenant_id: "your-tenant-id"
  client_id: "your-client-id"
  client_secret: "your-client-secret"
  scope: "api://dia-brain/.default"
  grant_type: "client_credentials"
  proxy_url: "http://rb-proxy-de.bosch.com:8080"
```

What it is used for:

- acquiring an Azure AD bearer token
- calling the configured LLM endpoint in `ask_db_functions.py`
- using an explicit HTTP/HTTPS proxy when needed

Runtime behavior:

- bearer tokens are cached back into `auth.yaml`
- the code refreshes tokens automatically when needed
- on Windows, there is a PowerShell fallback for proxied POST requests if direct `requests` proxy usage fails

## Operation Modes

### `cmd` mode

This is the full end-to-end pipeline.

Behavior:

- scans `records_folder` for supported media files without an `.srt`
- if no command-line file is passed, shows a selection menu in the console
- extracts `.mp3` from `.mp4` when needed
- transcribes with Whisper
- creates `.srt`
- optionally renames the media set using `file_rename_framing`
- asks the LLM for meeting minutes using `mm_question_framing`
- writes Outlook-ready HTML from the returned notes
- deletes intermediate helper files after successful processing

Typical usage:

```powershell
python .\create_transcript.py
```

Or pass one or more files directly:

```powershell
python .\create_transcript.py "C:\Recordings\meeting1.mp4"
python .\create_transcript.py "C:\Recordings\meeting1.mp4" "C:\Recordings\meeting2.m4a"
```

Outputs in `cmd` mode:

- final media file, possibly renamed
- final `.srt`, possibly renamed
- final `_toSend.HTML`, possibly renamed

Intermediate files may be created during processing and then removed:

- `.mp3`
- `.txt`
- `_notes.txt`
- `_q.txt`

### `gui` mode

This mode is intended for manual, one-file-at-a-time use.

Behavior:

- opens a CustomTkinter window
- supports drag and drop when `tkinterdnd2` is available
- allows browsing for a media file if drag and drop is unavailable
- requires language selection before starting
- transcribes the selected file
- creates a transcript text file
- copies the meeting-minutes prompt text to the clipboard
- does not call the LLM backend itself

Important difference from `cmd` mode:

- `gui` mode is a transcription-and-prompt-preparation tool
- it does not generate `_toSend.HTML`
- it does not generate LLM notes automatically

Set in `settings.yaml`:

```yaml
operation_mode: gui
```

Then run:

```powershell
python .\create_transcript.py
```

GUI output:

- original media file
- final transcript `.txt`
- clipboard text containing the prompt built from the transcript

### `watch` mode

This mode runs continuously and is intended for unattended queue processing.

Behavior:

- polls `watchfolder` every `watch_interval_seconds`
- waits until a file looks stable before processing
- avoids retry loops on unchanged failed files
- infers language from filename hints: `hun` -> `hu`, `eng` -> `en`
- otherwise uses the default `language`
- performs the same main processing pipeline as `cmd` mode

Set in `settings.yaml`:

```yaml
operation_mode: watch
```

Then run:

```powershell
python .\create_transcript.py
```

This process is meant to stay open until you stop it.

### `dictate` mode

This mode is for immediate voice dictation from the microphone.

Behavior:

- opens a small GUI window
- requires language selection before recording is enabled
- supports English and Hungarian
- `F2` toggles the same start/stop action as the button
- records microphone input
- encodes the recording to temporary MP3 at 16 kHz, mono, 64 kbps
- transcribes immediately with Whisper
- copies the raw transcription to the clipboard
- shows recording duration and transcription duration in the UI

Set in `settings.yaml`:

```yaml
operation_mode: dictate
```

Then run:

```powershell
python .\create_transcript.py
```

Dependencies specific to dictate mode:

- `sounddevice`
- `lameenc`
- a working microphone input device

## Whisper Model Notes

Model loading behavior:

- the code prefers `cuda` when available
- falls back to `mps` on Apple systems if available
- otherwise uses `cpu`
- for English, `tiny`, `base`, `small`, and `medium` may automatically switch to `.en` variants
- `turbo` is treated as a standalone model name and is not changed to `.en`

Useful helper:

```powershell
python .\check_whisper_gpu.py
python .\check_whisper_gpu.py --try-load --model tiny
```

Use this script if you want to verify whether Torch and Whisper can actually use your GPU.

## Output Files And Lifecycle

### Full pipeline outputs (`cmd` and `watch`)

Typical end state after successful processing:

- `meeting.mp4` or renamed variant
- `meeting.srt` or renamed variant
- `meeting_toSend.HTML` or renamed variant

### GUI mode outputs

Typical end state after successful processing:

- original media file
- transcript `.txt`

### Rename behavior

If `file_rename_framing` is not empty:

- the application asks the LLM for a short descriptive tag
- invalid filename characters are removed
- if the target name already exists, a `_v1`, `_v2`, and so on suffix is added to avoid collisions
- associated files are renamed together

### Cleanup behavior

The tool is intentionally aggressive about deleting helper files after successful `cmd` and `watch` processing. If you need raw `.txt`, `_notes.txt`, or `_q.txt` artifacts preserved, you must change the code before running production processing.

## LLM Integration

`cmd` and `watch` modes depend on `ask_db_functions.py`.

That module currently:

- authenticates against Azure AD using values from `auth.yaml`
- calls the endpoint `https://ews-emea.api.bosch.com/it/application/dia-brain/v1/api/chat/pure-llm`
- sends the prompt with a fixed `knowledgeBaseId`
- returns only the final `result` text to the caller

This means:

- transcription works locally with Whisper
- meeting minutes and rename tags require valid backend credentials and network access
- if the LLM call fails, transcription can still succeed, but notes, HTML, or renaming may be skipped or partial

## Running The Application

The executable behavior is driven entirely by `settings.yaml`.

Typical pattern:

1. Set `operation_mode` in `settings.yaml`.
2. Ensure `settings.yaml`, `auth.yaml`, and `ffmpeg.exe` are next to the script or executable.
3. Launch `create_transcript.py` or the built `.exe`.

Examples:

```powershell
python .\create_transcript.py
```

Or, once built:

```powershell
.\dist\create_transcript\create_transcript.exe
```

## Common Workflows

### I want a transcript from one meeting file

1. Set `operation_mode: gui` in `settings.yaml`.
2. Run `python .\create_transcript.py`.
3. Drag an `.mp4`, `.mp3`, or `.m4a` file into the window.
4. Choose the meeting language.
5. Click the start button.
6. When it finishes, the transcript file is written next to the source file and a prompt is copied to the clipboard.

### I want full meeting minutes and an HTML file

1. Configure `auth.yaml` with working credentials.
2. Set `operation_mode: cmd` in `settings.yaml`.
3. Make sure `mm_question_framing` and `file_rename_framing` are set the way you want.
4. Run `python .\create_transcript.py`.
5. Select a waiting file from the menu, or pass the file path on the command line.
6. After processing, look for the `.srt` and `_toSend.HTML` files next to the original media.

### I want the app to process a shared queue automatically

1. Set `operation_mode: watch` in `settings.yaml`.
2. Set `watchfolder` to the network or local folder to monitor.
3. Run `python .\create_transcript.py` and leave it open.
4. Drop files into the watched folder and let the tool process them when they become stable.

### I want fast speech-to-text for notes or chat messages

1. Set `operation_mode: dictate` in `settings.yaml`.
2. Run `python .\create_transcript.py`.
3. Choose `English` or `Hungarian`.
4. Click `Start Dictate` or press `F2`.
5. Speak.
6. Press `F2` again or click the button to stop.
7. Paste the clipboard text into Teams, Outlook, Word, or any other target app.

## Building The PyInstaller Executable

The repository already contains `create_transcript.spec`, including hidden imports and bundled data for Whisper and dictate-mode dependencies.

Recommended build command:

```powershell
pyinstaller --clean --noconfirm create_transcript.spec
```

Build result:

- executable output under `dist\create_transcript\`

The spec currently bundles:

- `ffmpeg.exe`
- `settings.yaml`
- `auth.yaml`
- Whisper package data
- `sounddevice` runtime data

### Important: do not strip the spec

`create_transcript.spec` uses `collect_data_files('whisper')` and `collect_submodules('whisper')`. Removing those will produce an exe that crashes at transcription time with a missing `mel_filters.npz` (or similar) inside the PyInstaller `_MEIPASS` Temp folder. Always keep them.

### About the exe size

The built exe is roughly 3 GB. Whisper model weights are NOT bundled. The size comes almost entirely from the PyTorch CUDA runtime DLLs (`torch_cuda.dll`, `cublasLt64_*.dll`, `cudnn_*.dll`, etc.) that the `+cu128` Torch wheel ships.

Whisper model files (`tiny`, `base`, `small`, `medium`, `large`, `turbo`) are downloaded on first use into `%USERPROFILE%\.cache\whisper\` and reused on subsequent runs.

Runtime portability:

- the same CUDA-enabled exe runs on any Windows x64 PC, with or without an NVIDIA GPU
- without a GPU, transcription falls back to CPU automatically (see `get_preferred_whisper_device`) and is slower, especially for `large` / `turbo`
- if you need a smaller exe and accept CPU-only speeds, install CPU Torch in the build environment before building:

```powershell
pip uninstall -y torch torchaudio torchvision
pip install torch --index-url https://download.pytorch.org/whl/cpu
pyinstaller --clean --noconfirm create_transcript.spec
```

## Troubleshooting

### `settings.yaml not found next to executable`

Cause:

- the script or executable cannot find `settings.yaml` in its own folder

Fix:

- place `settings.yaml` next to `create_transcript.py` or next to the built `.exe`

### Whisper does not use the GPU

Check:

```powershell
python .\check_whisper_gpu.py --try-load
```

Common causes:

- CPU-only Torch wheel installed
- CUDA-enabled Torch installed into a different environment
- missing or incompatible NVIDIA driver or CUDA runtime

### Drag and drop does not work in GUI mode

Cause:

- `tkinterdnd2` is missing or unavailable in the runtime

Fix:

- install `tkinterdnd2`
- use the click-to-browse fallback in the GUI

### Dictate mode cannot start recording

Check:

- microphone permissions
- working default input device
- `sounddevice` installation
- whether PortAudio dependencies are included in the environment or PyInstaller build

### `.mp4` files fail immediately

Cause:

- `ffmpeg.exe` is missing

Fix:

- place `ffmpeg.exe` next to the script or built executable

### LLM calls fail behind a proxy

The code can use:

- `auth.proxy_url` from `auth.yaml`
- `LLM_PROXY_URL`
- standard `HTTP_PROXY` or `HTTPS_PROXY` environment variables

If direct proxied requests fail on Windows, the helper code attempts a PowerShell POST fallback.

## Notes About The Helper Scripts

- `ask_db.py` is a tiny example that calls `ask()` with a test prompt
- `ask_db_for_support.py` is a separate support script for attachment upload and follow-up queries
- these are not the main application entry points

## Recommended Operational Practice

- use `gui` mode for manual, supervised transcription when you want to review before sending anything onward
- use `cmd` mode for full local batch processing
- use `watch` mode only when the watched folder and LLM connectivity are stable
- use `dictate` mode for quick speech-to-clipboard capture
- test new Whisper or Torch setups with `check_whisper_gpu.py` before packaging

## License

See `license.txt`.
