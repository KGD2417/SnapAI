# Snap AI

On-device AI art generation for Android. Stable Diffusion runs entirely on the
phone — the model lives locally, prompts and images never leave the device, and
generation works with the network off.

NPU inference is powered by **Qualcomm AI Engine Direct (QNN / QIDK)**, so on a
supported Snapdragon the UNet runs on the Hexagon NPU instead of the CPU, which
is what makes on-device generation fast enough to be usable.

**Why:** cloud image generation means your prompts and images sit on someone
else's server, you pay per image, and you need a connection. Running the model
on-device gives you privacy, low latency, and offline capability.

## How it works

```
Compose UI (Kotlin)
      │  HTTP on localhost:8081
      ▼
libstable_diffusion_core.so   ← native binary, launched by BackendService
      │
      ├── QNN / QIDK  → UNet + VAE on the Hexagon NPU   (NPU models, .bin)
      └── MNN         → CPU/GPU fallback                (CPU models, .mnn)
```

| Piece | Where |
|---|---|
| UI, navigation, model download | `app/src/main/java/com/example/snapai/` |
| Backend process launcher | `service/BackendService.kt` |
| Native inference server | `app/src/main/cpp/src/main.cpp` |
| QNN graph execution | `app/src/main/cpp/src/QnnModel.hpp` |
| Sampler | `app/src/main/cpp/src/DPMSolverMultistepScheduler.hpp` |
| Model catalog / download URLs | `data/Model.kt` |

Models are downloaded on first use, not shipped in the APK.

## Build status

| | |
|---|---|
| Kotlin / Compose app | builds — `./gradlew assembleBasicDebug` produces `SnapAI_armv8a_2.0.0.apk` (67 MB), label `Snap AI`, launcher `com.example.snapai.MainActivity` |
| Native core (`build.sh`) | **not built in this tree** — needs the QNN SDK + NDK, see Step 4 |
| Release APK | needs the `RELEASE_*` signing properties, see Step 5 |

A fresh clone builds an installable APK, but **image generation will not work
until Step 4 has been run** — `jniLibs/` and `assets/qnnlibs/` are build outputs
and are gitignored, so they are not in the repo.

## Supported NPUs

Snapdragon 8 Gen 1 / Gen 2 / Gen 3 / 8 Elite / 8 Elite Gen 5. Other Qualcomm
chips from 2020 onward are experimental. Anything else falls back to CPU/GPU.

---

# Building it

## My system requirements

- Linux, or Windows with WSL2 (macOS unverified)
- Rust, Ninja, CMake, Android Studio, Git, ccache

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env
rustup default stable
rustup target add aarch64-linux-android

sudo apt update && sudo apt install ninja-build cmake ccache git   # Ubuntu/WSL
brew install ninja cmake ccache git                                # macOS
```

Rust is needed because `tokenizers-cpp` builds the HuggingFace tokenizer from
Rust source.

## Step 1: SDKs

**QNN SDK 2.39** — https://qpm.qualcomm.com/#/main/tools/details/qualcomm_ai_engine_direct
(Qualcomm account required). Extract it somewhere I'll remember.

**Android NDK** — https://developer.android.com/ndk/downloads (r27c is what I used).

My paths:
- QNN SDK: `_____________________________` (the `qairt/<version>` directory)
- Android NDK: `_____________________________`

## Step 2: Clone

```bash
git clone https://github.com/KGD2417/SnapAI.git
cd SnapAI
```

No `--recursive` needed — MNN, tokenizers-cpp, zstd and the rest are vendored
in `app/src/main/cpp/3rdparty/`.

## Step 3: Point the build at my SDKs

Both paths come from environment variables now, so I don't have to edit any
files:

```bash
export QNN_SDK_ROOT=~/SDKs/qairt/2.39.0.250926
export ANDROID_NDK_ROOT=~/SDKs/android-ndk-r27c
```

CMake fails immediately with a clear message if `QNN_SDK_ROOT` is wrong, rather
than spewing copy errors. There's still a hardcoded fallback path in
`app/src/main/cpp/CMakeLists.txt` from my old machine — the env var wins.

## Step 4: Build the native libraries first

This has to happen **before** the APK build — Gradle just packages whatever is
already in `jniLibs/` and `assets/`, it does not run CMake.

```bash
cd app/src/main/cpp
./build.sh
```

That does three things:
1. builds `libstable_diffusion_core.so` for arm64-v8a
2. copies the QNN runtime `.so` files to `app/src/main/assets/qnnlibs/`
3. copies the executable to `app/src/main/jniLibs/arm64-v8a/`

On Windows use `build.bat` instead. First build is 15–30 minutes (MNN is big);
ccache makes rebuilds much faster.

Both output directories are gitignored, so this step is required on every fresh
clone. Skipping it still produces a working-looking APK — it opens, lists and
downloads models, then fails at generation with "Backend start failed".

## Step 5: Build the APK

Open the project in Android Studio, let Gradle sync, then
**Build → Generate Signed Bundle / APK → APK**.

Or from the command line:

```bash
./gradlew assembleBasicDebug      # no signing setup needed
./gradlew assembleBasicRelease    # needs the RELEASE_* properties below
```

Two flavors: `basic`, and `filter` which additionally runs a NSFW safety
checker model.

The release build reads `RELEASE_STORE_FILE`, `RELEASE_STORE_PASSWORD`,
`RELEASE_KEY_ALIAS` and `RELEASE_KEY_PASSWORD` as Gradle properties and fails
without them. They are deliberately **not** in the repo's `gradle.properties` —
put them in `~/.gradle/gradle.properties` or pass them with `-P`, so the
keystore password never gets committed.

Output: `app/build/outputs/apk/basic/<debug|release>/SnapAI_armv8a_2.0.0.apk`

## Step 6: Install

Phone: Settings → About Phone → tap Build Number 7× → Developer Options →
enable USB Debugging.

```bash
adb install app/build/outputs/apk/basic/debug/SnapAI_armv8a_2.0.0.apk
```

## Troubleshooting

**`./gradlew` fails with just a version number as the error (e.g. `* What went
wrong: 25.0.3`)** — my default JDK is too new. Gradle 8.14.3 supports Java 24 at
most, and I have JDK 25 on `PATH`. Android Studio is fine (it uses its own
bundled JDK 21); only the command line breaks. Fix it per-invocation:

```bash
./gradlew assembleBasicRelease "-Dorg.gradle.java.home=C:/Program Files/Android/Android Studio/jbr"
```

or point `JAVA_HOME` at a JDK 17–21 for the shell. Not putting the path in
`gradle.properties` — it's specific to this machine.

**"QNN SDK not found at ..."** — `QNN_SDK_ROOT` isn't set or points at the wrong
level. It must be the `qairt/<version>` directory, the one containing `lib/` and
`include/`.

**CMake can't find the NDK** — `ANDROID_NDK_ROOT` unset. `CMakePresets.json`
builds the toolchain path from it; there is no default.

**`ccache: not found`** — install it, or drop the two
`CMAKE_*_COMPILER_LAUNCHER` lines from `CMakePresets.json`.

**App installs but generation fails instantly** — the native binary didn't get
packaged. Check `app/src/main/jniLibs/arm64-v8a/libstable_diffusion_core.so`
exists and rerun Step 4.

**"Backend start failed"** on a real device — usually an unsupported NPU. Try a
CPU model from the model list to confirm the rest of the app works.

**Out of disk space** — the build needs roughly 10GB.

## Credit

Built on the open source [Local Dream](https://github.com/xororz/local-dream)
project, which provides the QNN inference core and the converted models hosted
on HuggingFace.
