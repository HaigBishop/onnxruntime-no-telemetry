# ONNX Runtime without telemetry

Microsoft's [ONNX Runtime](https://github.com/microsoft/onnxruntime) for Windows x64, built from Microsoft's source with Microsoft's release flags, without its telemetry. [PetriScan](https://petriscan.com) Desktop ships it in place of Microsoft's build, so that nothing ONNX Runtime does reaches Windows' diagnostic data.

Each [release](https://github.com/HaigBishop/onnxruntime-no-telemetry/releases) is one ONNX Runtime version, tagged as Microsoft tags it (`v1.28.3`), with one archive named and laid out as Microsoft's (`onnxruntime-win-x64-1.28.3.zip`, holding `lib/onnxruntime.dll` and its PDB), so it stands in wherever Microsoft's does. Its notes give the SHA-256 of the archive and of the library. A release is never replaced.

## Why a build of its own

Microsoft's library for Windows registers an Event Tracing for Windows provider, `Microsoft.ML.ONNXRuntime` (`{3a26b1ff-7484-7484-7484-15261f42614d}`), as an app makes its environment, and writes its telemetry events to it. Windows' diagnostic data listener, `Diagtrack-Listener`, names that provider by its ID, so Windows subscribes to it in every process that does this. What Windows then receives depends on the computer's diagnostic data setting and on Windows' own rules, which no app can see.

ONNX Runtime's API cannot undo that. `DisableTelemetryEvents` sets a flag of ONNX Runtime's own, which ONNX Runtime sets back whenever Windows enables the provider again, and the first event is written as the environment is made, whenever Windows asks for it, before an app can call anything.

Microsoft's release pipeline turns its telemetry on with `--use_telemetry`, which puts the provider in Microsoft's telemetry provider group. A build from source without the flag leaves the provider out of the group but still registers it under the same ID, which is all Windows' listener needs. So leaving out the flag is not enough.

Checked on Windows 11 Pro on 2026-11-17: the listener names the provider, and in a process that made an environment with Microsoft's 1.28.3 library, Windows' session enabled it at once and kept it enabled after `DisableTelemetryEvents`.

## What differs from Microsoft's release

- **[windows.patch](https://github.com/HaigBishop/onnxruntime-no-telemetry/blob/main/windows.patch)** holds the changes to Microsoft's source:
  - Windows' environment keeps ONNX Runtime's no-op telemetry, the `Telemetry` base class its Linux and macOS builds use, rather than `WindowsTelemetry`. The provider is never registered, so there is nothing for Windows to subscribe to and no telemetry event can be written. Nothing else changes, and inference is untouched.
  - The version information says the library is PetriScan's build, made by PetriScan Limited as the product ONNX Runtime, since Microsoft neither built nor signed it. The copyright stays Microsoft's.
- **The flags** are Microsoft's, the command of the x64 job of its Windows CPU release (`tools/ci_build/github/azure-pipelines/templates/c-api-cpu.yml` through `win-ci.yml`, at the release's tag), less four:
  - `--use_telemetry`, the point of this build.
  - `--use_vcpkg_ms_internal_asset_cache`, Microsoft's internal mirror for vcpkg's downloads. ONNX Runtime pins every dependency itself, in `cmake/vcpkg-configuration.json` and its overlay ports, and vcpkg checks each download's hash.
  - `--build_java`, the Java bindings, which are a library of their own.
  - `--enable_onnx_tests`, with the tests themselves, since the one target built is the `onnxruntime` library. The configuration stays Microsoft's, and the tests do not change the library.
- **The tools** are Microsoft's: Python 3.12 with the packages its build installs, CMake and vcpkg through Microsoft's own `setup-build-tools` action with the versions and hashes its packaging pipeline passes (`tools/ci_build/run_gh_action.py`), and the `windows-2022` runner, whose Visual Studio 2022 (17.14, MSVC 14.44) and Windows SDK (10.0.26100.0) are what Microsoft's library was built with. `windows-2025` has moved to Visual Studio 2026.
- **The archive** holds the library and its PDB, ONNX Runtime's `LICENSE` and `ThirdPartyNotices.txt`, its `VERSION_NUMBER`, the `GIT_COMMIT_ID` it was built from, and this README. It leaves out the headers, the import libraries and `onnxruntime_providers_shared.dll`, which an app that loads the library as it runs does not need.
- **No signature.** Microsoft signs its library, and this one is not signed.

## How each build is checked

Before a release is published, the workflow holds the library to Microsoft's own release of the version, which it fetches:

- **The same source:** Microsoft's archive is validly signed by Microsoft, and its `GIT_COMMIT_ID` names the commit this build checked out.
- **No telemetry provider:** [check-telemetry.py](https://github.com/HaigBishop/onnxruntime-no-telemetry/blob/main/check-telemetry.py) loads the library into a process of its own, makes an environment and asks Windows whether the provider is registered in that process. In this build it must not be, and in Microsoft's it must be, which shows the check would see it.
- **Built as Microsoft's is:** the library was linked by the same MSVC and imports the same libraries as Microsoft's, which a change of toolchain or configuration would show, and its version information names PetriScan Limited.

The release notes name the run that built it, its MSVC and the runner image.

## Building a version

1. At Microsoft's tag for the version, compare the command in `win-ci.yml` and `c-api-cpu.yml`, the tool versions in `tools/ci_build/run_gh_action.py`, and the toolchain Microsoft's library names (its linker version, and the MSVC and Windows SDK paths in its PDB) with this workflow's, and bring the workflow in line.
2. Run `gh workflow run build.yml -R HaigBishop/onnxruntime-no-telemetry -f version=<version>`. It refuses a version that already has a release, and fails if `windows.patch` no longer applies.
3. PetriScan Desktop then pins the SHA-256 of the archive and of the library from the release notes in its `src-tauri/build.rs`, once its own checks agree.

**Past 1.28,** Microsoft's builds on every platform carry a 1DS telemetry client that uploads over HTTPS, and a build from source includes it unless `--no_telemetry` is passed. A version past 1.28 adds that flag, takes a fresh look at `windows.patch` and `check-telemetry.py`, and builds for macOS and Linux as well.

## Licence

ONNX Runtime is Microsoft's, under the MIT License, and every archive carries its `LICENSE` and `ThirdPartyNotices.txt`. This repository's own files are under the MIT License too ([LICENSE](https://github.com/HaigBishop/onnxruntime-no-telemetry/blob/main/LICENSE)).
