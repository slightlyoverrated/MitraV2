# Mitra V2

Mitra V2 is a privacy-conscious native Windows assistant built with C#, WinUI 3, .NET 10, SQLite, and a bundled Python 3.12 sidecar. It can launch applications, search the web and local known folders, manage notes and reminders, run timers, focus intervals, and safe routines, provide attributed weather, control selected Windows features, understand local speech, and stay available through the compact Mitra Island.

> Renovated September 2026. The self-contained x64 release is in `D:\MitraV2\release`. See [Renovation and validation](docs/RENOVATION.md) for current evidence and unfinished features. Older release-gate reports describe earlier builds.

![Mitra companion](docs/images/renovated-home.png)

## Run the finished app

Open **`D:\MitraV2\release\Mitra.exe`**, keeping its accompanying files and `Agent` folder together. Or open **`D:\MitraV2\release\MitraSetup.exe`** to install into a new, empty writable folder. This folder installer bundles .NET, the Windows App SDK, Python and the speech engine. No development tools or terminal commands are needed to run it. It is unsigned and does not register a Windows uninstall entry; close Mitra and remove its folder to uninstall, keeping a backup of `UserData` if needed.

The portable release keeps settings, notes and optional models in `UserData` beside the executable. Existing development data is left alone. The first run offers theme, microphone, Windows voice, a calm speaking speed, optional Ollama connection and privacy controls. Text commands work immediately; speech recognition needs an optional Whisper model. A trained **Hey Mitra** wake-word model is not bundled.

Try `Calculate 12 * (3 + 4)`, `Convert 25 C to F`, `Search YouTube for circular motion`, or `Remind me to submit this tonight` (Mitra asks for the time). Use Settings to choose your browser and weather location. The floating panel supports text, microphone, cancel, mute and expansion; Ctrl+Space and the tray keep it accessible.

For explanations, writing and document questions, connect a locally installed model through **Connect local AI** in setup or **Settings → Intelligence**. Ollama and its models are optional and are not bundled. Drop up to two files into Home or choose **Use clipboard** with clipboard permission enabled, then ask Mitra to summarize, rewrite, translate or review. Copy, save as note and read aloud are available on the answer. Sources stay in session memory and attachment requests go only to loopback Ollama. Clear the source to resume desktop commands.

Local document input supports text/code/CSV, DOCX, PDF and image text through Windows OCR. Limits are 20 MB per file, 24,000 source characters and the first 12 PDF pages. OCR is text extraction, not visual reasoning. Browser searches open a search page; `open the first result` is unfinished.

Windows speech now applies speed once, defaults to 0.95, avoids duplicate acknowledgements and reads a short version of long answers. **Read aloud** requests the full answer. New installations keep command history and conversation retention off by default; deliberately saved notes and memories remain persistent.

## Rebuild this release

Using PowerShell 7 from the repository root, run `./scripts/release.ps1`. It publishes the self-contained host, includes the frozen sidecar from `artifacts/agent/win-x64`, and builds `MitraSetup.exe`. Use `scripts/build-agent.ps1` first only when changing the Python sidecar. The older MSIX workflow below remains available but was not rebuilt for this renovation.

## Requirements

For development:

- Windows 11 x64 is the primary target; Windows 10 build 19041 or later is supported where Windows APIs permit.
- .NET 10 SDK.
- Windows App SDK dependencies restored by the build (the release bundles its runtime).
- PowerShell 7 or Windows PowerShell 5.1.
- Python 3.12 only when rebuilding the frozen voice sidecar. Ordinary host development and installed releases do not require a system Python.
- A current Visual Studio WinUI workload is recommended for XAML editing, but the command-line build is supported.

The repository can use its local `.dotnet` SDK, NuGet cache, Python 3.12 toolchain, and wheelhouse. These development directories are not installer prerequisites.

## Repository map

```text
src/                  WinUI host, domain, infrastructure, skills, and tests
agent/                Authenticated local Python sidecar and locked dependencies
assets/               Original Mitra visual and audio assets
installer/            Reviewed MSIX install and uninstall helpers
scripts/              Build, test, security, accessibility, recovery, and release gates
docs/                 Architecture, privacy, operations, test evidence, and status
artifacts/             Generated sidecar, package, and validation reports
```

## Build and test

From the repository root:

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass `
  -File scripts/test.ps1 -Configuration Release -Platform x64 -Offline
```

Omit `-Offline` when dependencies are not already cached. The test script builds the unpackaged WinUI target, runs all .NET tests, and then runs the Python tests.

Additional release gates:

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -File scripts/check-security.ps1 -Offline
powershell.exe -NoProfile -ExecutionPolicy Bypass -File scripts/check-accessibility.ps1
powershell.exe -NoProfile -ExecutionPolicy Bypass -File scripts/first-run-smoke.ps1 -Configuration Release -NoBuild -Offline
powershell.exe -NoProfile -ExecutionPolicy Bypass -File scripts/accessibility-smoke.ps1 -Configuration Release -NoBuild -Offline
powershell.exe -NoProfile -ExecutionPolicy Bypass -File scripts/profile-startup.ps1 -Configuration Release -NoBuild -Offline
powershell.exe -NoProfile -ExecutionPolicy Bypass -File scripts/recovery-smoke.ps1 -Configuration Release -NoBuild -Offline
```

`MITRA_DATA_ROOT` can point a development or smoke-test process at an isolated absolute data directory.

## Run from source

The unpackaged source build includes the Windows App SDK runtime:

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -File scripts/run.ps1
```

The first launch opens a ten-section setup flow. Optional permissions remain off unless selected, and every optional section can be skipped. Text commands remain available without a microphone, speech model, wake-word model, cloud provider, clipboard access, screen awareness, or startup registration.

The host starts the sidecar itself, authenticates it through a random user-scoped named pipe, and owns its shutdown. Do not start an unauthenticated listener manually.

## Voice runtime and models

Release packaging uses [scripts/build-agent.ps1](scripts/build-agent.ps1) to create `artifacts/agent/win-x64/mitra-agent.exe` from hash-locked Python 3.12 dependencies. Installed users do not need Python.

No large speech or wake-word model is downloaded automatically. Voice settings offer pinned multilingual faster-whisper Tiny, Base, and Small models with explicit size display, confirmation, progress, SHA-256 verification, corruption detection, and deletion. A compatible, appropriately licensed wake-word ONNX model must be imported separately; push-to-talk and text input are always available.

## Build and install the MSIX

Create the frozen sidecar and a development-signed x64 package:

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass `
  -File scripts/build-agent.ps1 -Offline

powershell.exe -NoProfile -ExecutionPolicy Bypass `
  -File scripts/package.ps1 -Version 0.1.0.0 `
  -DevelopmentSigning -SkipAgent -Offline
```

The package, public development certificate, lifecycle scripts, and machine-readable manifest are written to `artifacts/installer`.

A development certificate must be trusted in the Local Computer `TrustedPeople` store, so development installation requires an elevated PowerShell session:

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass `
  -File installer/Install-Mitra.ps1 `
  -PackagePath artifacts/installer/Mitra-0.1.0.0-x64-dev.msix `
  -DevelopmentCertificatePath artifacts/installer/Mitra-0.1.0.0-development.cer
```

Production packaging uses `-PfxPath` and reads the password only from `MITRA_SIGNING_PASSWORD`. The certificate subject must exactly match the manifest publisher. An unsigned inspection package is produced when neither signing option is supplied.

MSIX upgrades preserve `%LOCALAPPDATA%\Mitra`. Normal uninstall preserves it too; use Settings > Privacy > Clear data first, or explicitly pass `-RemoveUserData` to the uninstall helper. See [Packaging](docs/PACKAGING.md) and [Release checklist](docs/RELEASE_CHECKLIST.md).

## Privacy and safety

Deterministic local routing is the default. Microphone, location, file contents, clipboard, focus history, screen awareness, cloud conversation text, cloud files, cloud screen context, and long-term memory have separate controls. Raw microphone recordings and screenshots are not retained. Clipboard and OCR content is excluded from command and conversation persistence.

Optional unmatched-command routing can use loopback-only Ollama or a user-configured OpenAI-compatible HTTPS endpoint. Model output can select only a registered skill and typed arguments; Mitra independently validates permissions, risk, provenance, and confirmation immediately before execution. Credentials are protected with user-scoped Windows DPAPI and excluded from diagnostics and ordinary export.

## Troubleshooting

See [Troubleshooting](docs/TROUBLESHOOTING.md). Common development failures are an incompatible .NET SDK, a missing Windows App Runtime, an empty NuGet cache during an offline build, or a development certificate that is not trusted in the Local Computer store.

## Documentation

- [Acceptance tests](docs/ACCEPTANCE_TESTS.md)
- [Accessibility](docs/ACCESSIBILITY.md)
- [Architecture](docs/ARCHITECTURE.md)
- [Database and recovery](docs/DATABASE.md)
- [Design system](docs/DESIGN_SYSTEM.md)
- [Implementation status](docs/IMPLEMENTATION_STATUS.md)
- [Intelligence providers](docs/INTELLIGENCE.md)
- [Known limitations](docs/KNOWN_LIMITATIONS.md)
- [Legacy audit](docs/LEGACY_AUDIT.md)
- [Packaging](docs/PACKAGING.md)
- [Performance](docs/PERFORMANCE.md)
- [Privacy](docs/PRIVACY.md)
- [Productivity services](docs/PRODUCTIVITY.md)
- [Release checklist](docs/RELEASE_CHECKLIST.md)
- [Security](docs/SECURITY.md)
- [Skill system](docs/SKILL_SYSTEM.md)
- [Troubleshooting](docs/TROUBLESHOOTING.md)
- [Voice pipeline](docs/VOICE_PIPELINE.md)
- [Wake-word model workflow](docs/WAKE_WORD_MODEL.md)
