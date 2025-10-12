# Snap AI

Step-by-step guide to build Local Dream APK on my machine.

## My System Requirements

**Operating System:**
- Linux (recommended)
- OR Windows with WSL2 installed
- macOS might work but not verified

**Required Tools:**
- Rust
- Ninja build system
- CMake
- Android Studio
- Git

## Step 1: Install Prerequisites

### Install Rust
```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env
rustup default stable
rustup target add aarch64-linux-android
```

### Install Ninja and CMake

**On Ubuntu/Debian:**
```bash
sudo apt update
sudo apt install ninja-build cmake git
```

**On macOS:**
```bash
brew install ninja cmake git
```

**On Windows (WSL):**
```bash
sudo apt update
sudo apt install ninja-build cmake git
```

### Install Android Studio
- Download from: https://developer.android.com/studio
- Install and complete the setup wizard

## Step 2: Download Required SDKs

### Download QNN SDK 2.39
1. Go to: https://qpm.qualcomm.com/#/main/tools/details/qualcomm_ai_engine_direct
2. Create/login to Qualcomm account (required)
3. Download QNN SDK version 2.39
4. Extract to a location I'll remember (e.g., `~/SDKs/QNN_SDK_2.39`)

### Download Android NDK
1. Go to: https://developer.android.com/ndk/downloads
2. Download latest NDK
3. Extract to a location (e.g., `~/SDKs/android-ndk-r26d`)

**Note:** Write down these paths - I'll need them in Step 4!

My SDK paths:
- QNN SDK: `_____________________________`
- Android NDK: `_____________________________`

## Step 3: Clone the Repository

```bash
cd ~/Projects  # or wherever I keep my code
git clone --recursive https://github.com/xororz/local-dream.git
cd local-dream
```

**Important:** The `--recursive` flag downloads all submodules. Don't skip it!

## Step 4: Configure SDK Paths

### Edit CMakeLists.txt
```bash
nano app/src/main/cpp/CMakeLists.txt
```

Find the line with `QNN_SDK_ROOT` and update it to my QNN SDK path:
```cmake
set(QNN_SDK_ROOT "/home/myuser/SDKs/QNN_SDK_2.39")
```

Save and exit (Ctrl+X, then Y, then Enter)

### Edit CMakePresets.json
```bash
nano app/src/main/cpp/CMakePresets.json
```

Find the line with `ANDROID_NDK_ROOT` and update it to my NDK path:
```json
"ANDROID_NDK_ROOT": "/home/myuser/SDKs/android-ndk-r26d"
```

Save and exit

## Step 5: Build Native Libraries

Look for build scripts in the project:
```bash
ls *.sh  # Check for build scripts
```

If there's a `build.sh` or similar:
```bash
chmod +x build.sh
./build.sh
```

If no build script exists, I'll need to build manually with CMake:
```bash
cd app/src/main/cpp
cmake -B build -G Ninja
cmake --build build
cd ../../..
```

## Step 6: Build APK in Android Studio

1. Open Android Studio
2. Click "Open an Existing Project"
3. Navigate to my `local-dream` folder and select it
4. Wait for Gradle sync to complete (this might take several minutes)
5. Go to: **Build → Generate Signed Bundle / APK**
6. Select **APK**
7. Create a new keystore or use an existing one
8. Click **Next** → **Release** → **Finish**

The APK will be in: `app/release/app-release.apk`

## Step 7: Install on My Phone

### Enable Developer Options on Phone:
1. Settings → About Phone
2. Tap "Build Number" 7 times
3. Go back → Developer Options
4. Enable "USB Debugging"

### Install via USB:
```bash
adb install app/release/app-release.apk
```

OR transfer the APK to my phone and install manually.

## Troubleshooting My Build

**CMake can't find NDK:**
- Double-check the path in `CMakePresets.json`
- Make sure there are no trailing slashes
- Use absolute paths, not relative ones

**QNN SDK not found:**
- Verify the path in `CMakeLists.txt`
- Make sure I downloaded the correct version (2.39)

**Gradle sync fails:**
- Check internet connection
- Try: File → Invalidate Caches → Invalidate and Restart

**Native library build fails:**
- Make sure all prerequisites are installed
- Check if submodules were cloned: `git submodule update --init --recursive`

**Out of disk space:**
- This project needs ~10GB for build files
- Clean up space and try again

**Rust target missing:**
- Run: `rustup target add aarch64-linux-android`

## Build Notes

**Build times on my machine:**
- First build: ~15-30 minutes (downloads dependencies)
