# Handoff: LineageOS 17.1 build attempt for suez

Target device: Amazon Fire HD 10, 2017, 7th generation. Codename: suez.
Current state: the device runs LineageOS 16.0 (Android 9) and is rooted.
Goal: build LineageOS 17.1 (Android 10) for suez, then test it on the real device.

Build machine for this attempt: a different machine, not the one used before.
Build disk: `/mnt/m2_disk/` (120GB).

## 1. Why this project exists

The device WebView is old and unmaintained (Bromite, frozen since 2022).
Current WebView options (Cromite, current Chrome) need Android 10 or higher.
The device is on Android 9. A WebView upgrade needs an OS upgrade first.

## 2. What is already known

No finished LineageOS 17.x or 18.x build exists for suez. Checked XDA in September 2026.
The most recent relevant post is from November 2025. It still points to LineageOS 16.0 as the newest working build.

The device tree and kernel repos for suez are frozen. Last commit: February 2023.
They have only two branches: `cm-12.1` and `lineage-16.0`. No `lineage-17.1` branch exists.

One person tried to build LineageOS 17.1 for suez before. The attempt is not finished and not released.
It is documented in a blog post, not a real ROM release. See link list at the end of this file.
That attempt hit three specific build failures. List them in section 4. Expect to hit the same ones.

## 3. What we already tested, and the result

We ran a test build on GitHub Actions CI, using a fork of the current LineageOS 16.0 build repo.

- Fork: `https://github.com/domgregori/lineage16-suez-build`
- Branch: `lineage-17.1-attempt`
- Workflow file: `.github/workflows/build.yml` on that branch

The manifest points `repo init` at `lineage-17.1` (the newer LineageOS source tree).
It keeps the device tree, kernel, and vendor blobs at their existing `lineage-16.0` revision, since no newer version exists.

Result:
- `repo sync` finished with no errors.
- `lunch lineage_suez-userdebug` finished with no errors. It printed a full Android 10 config banner:
  `PLATFORM_VERSION=10`, `LINEAGE_VERSION=17.1-...-UNOFFICIAL-suez`, `BUILD_ID=QQ3A.200805.001`.
  This is a good sign. It means the old device tree is not rejected outright by the newer build system.
- The build then started Soong (the build-system bootstrap step) and reached 138 of 139 steps there.
- At that point the CI runner failed. GitHub's own message: "The hosted runner lost communication with the server."
  This is a resource failure, not a code failure. Disk use was already 67GB of 93GB right after the source sync,
  before any real compiling started. Only 26GB was free.

**Conclusion: the config stage works. We have not yet tested a real compile of any device, kernel, or HAL code.**
That real compile is the actual unknown. Find out what happens there on the new machine.

## 4. Known failure points to expect

These come from the one earlier, unfinished attempt at LineageOS 17.1 for suez (see link list).
That build needed these fixes, and the person who tried never fully solved all of them:

1. **Bluetooth and AudioFX blocked the build.** The build only completed after both were removed.
2. **The WiFi HAL failed to build.** File: `libwifi-hal-mt66xx`. Cause: undefined symbols.
3. **OpenSSL/BoringSSL function calls did not match.** This happens when an old vendor tree meets a newer crypto library.

Expect one or more of these three problems. Plan for real source-level fixes, in C or C++ code,
not just config changes. Do not remove Bluetooth, audio, or WiFi as a first response.
Removing a working radio defeats the goal of this project. Try a real fix first.

## 5. Build environment setup, on the new machine

Use `/mnt/m2_disk/` as the build root. Example: `/mnt/m2_disk/lineage`.

**Disk space warning:** 120GB may not be enough. The CI test used 67GB for source alone, with no
real compile output yet. A full Android build often needs 60-100GB more for the `out/` directory.
Watch free space from the start. Run `df -h /mnt/m2_disk` often during the build. Have a plan ready:
free ccache space, remove unused synced projects, or clear old `out/` files if space runs low.

Packages needed (names shown are for Arch/Manjaro; check the new machine's package manager for equivalents):

```
jdk8-openjdk repo ccache flex bison gperf schedtool zip unzip rsync bc lzop git python
```

**Java version warning:** LineageOS 17.1's own build docs call for OpenJDK 9, not 8.
The current CI workflow still installs Java 8 (this worked for LineageOS 16.0).
Test this early. If the build stops with a Java version error, install OpenJDK 9 and try again,
or search for a documented patch that relaxes the version check.

## 6. Commands to reproduce the build

```bash
mkdir -p /mnt/m2_disk/lineage
cd /mnt/m2_disk/lineage

repo init \
  -u https://github.com/LineageOS/android.git \
  -b lineage-17.1 \
  --depth=1 \
  --no-repo-verify \
  --no-clone-bundle

mkdir -p .repo/local_manifests
cat > .repo/local_manifests/roomservice.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<manifest>
  <project name="lineage16-suez/device_amazon_suez"
           path="device/amazon/suez"
           remote="github"
           revision="lineage-16.0" />
  <project name="lineage16-suez/kernel_amazon_suez"
           path="kernel/amazon/suez"
           remote="github"
           revision="lineage-16.0" />
  <project name="ivanlopezmolina/vendor_amazon_suez"
           path="vendor/amazon/suez"
           remote="github"
           revision="main" />
</manifest>
EOF

repo sync -c --no-tags --no-clone-bundle --force-sync -j<N>
```

Then, before the first build, apply the same host-environment fixes the CI workflow already
proved necessary (these are not suez-specific; they come from the shallow sync and host setup,
so apply them up front instead of waiting for the same errors to appear again):

- Patch `external/v8/Android.bp` (a `torque` build rule needs a `srcs:` field added — see the
  workflow file step named "Apply external/v8 Android.bp fix" for the exact patch).
- Create a stub file at `cts/error_prone_rules.mk` (needed because CTS is not synced in a
  device-only build).
- Create an empty file at
  `out/target/product/suez/obj_arm/SHARED_LIBRARIES/libnvram_intermediates/export_includes`
  (works around a missing vendor blob reference).

Full copy-pasteable versions of all three fixes:
`https://github.com/domgregori/lineage16-suez-build/blob/lineage-17.1-attempt/.github/workflows/build.yml`

Then build:

```bash
source build/envsetup.sh
lunch lineage_suez-userdebug
mka bacon -j<cores>
```

Use a job count matched to the new machine's CPU core count and RAM.
More cores need more RAM. Watch memory use during the first build. Lower `-j` if the system swaps heavily.

## 7. Step-by-step plan

1. Install packages. Confirm `repo`, `java`, `ccache` all run.
2. Run the `repo init` and `repo sync` commands above. Confirm the sync finishes with no errors.
3. Apply the three known host-environment fixes from section 6, before the first build attempt.
4. Run `lunch lineage_suez-userdebug`. Confirm it prints the config banner shown in section 3.
5. Run `mka bacon`. Watch disk space and memory use the whole time, in a separate terminal:
   `watch -n 30 'df -h /mnt/m2_disk; free -h'`
6. When a real compile error appears (expect this — see section 4), read the exact failing file
   and error message. Do not assume it is one of the three known problems until the log confirms it.
7. Fix the specific file at the source level. Rebuild. Ninja and ccache should only rebuild
   what changed, so this is faster than the first build.
8. Repeat steps 6 and 7 until `mka bacon` finishes and produces a ROM zip in
   `out/target/product/suez/`.
9. Stop. Do not flash the device yet. Go to section 8 first.

## 8. Testing plan, before touching the real device

**Back up first.** In TWRP, take a full backup of the current, working LineageOS 16.0 install
(boot, system, data). Save the backup off the tablet's own storage — copy it to the build machine
or another drive. This is the rollback path if the new build breaks something.

**GApps gap:** the GApps package already downloaded (`BiTGApps-arm64-9.0.0-v3.8-ROAR.zip`) targets
Android 9. It will not work on Android 10. Find and download a BiTGApps (or equivalent) package
built for Android 10 (API level 29), arm64, before testing Google apps on the new build.

**Keep the rollback files ready**, both already present at
`/home/domgr/Downloads/Fire HD 10 7th Root/`:
- `lineage-16.0-20220705-UNOFFICIAL-suez.zip`
- `BiTGApps-arm64-9.0.0-v3.8-ROAR[1].zip`

### Flash and test order

1. Sideload the new build through TWRP, the same way LineageOS 16.0 was installed:
   `adb sideload lineage-17.1-....zip`
2. Watch the first boot. Android 10's first boot is normally slower than Android 9's.
   A boot loop past 10-15 minutes is a real failure, not a slow boot.
3. If it boots, test each of these in order. These are the parts known to break, from section 4:
   - **WiFi.** Connect to a network. Reboot once. Confirm it reconnects on its own.
   - **Bluetooth.** Pair a device. Confirm audio or data actually moves, not just a successful pair.
   - **Audio.** Play sound through the speaker. Test any equalizer or AudioFX-style setting.
   - **Touchscreen and general UI.** Confirm normal response, no lag or dead zones.
   - **adb/USB**, from the build machine.
   - **Camera and sensors**, if the tablet has working ones to test.
   - **Reboot stability.** Reboot two or three times in a row. Confirm no boot loop appears.
4. If GApps are wanted, sideload the correct Android 10 GApps package next, then repeat the
   Play Store and app-install checks.
5. If a check in step 3 fails and there is no quick fix: restore the TWRP backup from before
   flashing. A tablet with no WiFi or no Bluetooth is worse than the current working LineageOS 16.0.
   Do not leave the device on a broken build.

## 9. Reference links

- Fork used for the CI test: `https://github.com/domgregori/lineage16-suez-build`
  (branch: `lineage-17.1-attempt`)
- Upstream LineageOS 16.0 build source: `https://github.com/ivanlopezmolina/lineage16-suez-build`
- Device tree: `https://github.com/lineage16-suez/device_amazon_suez`
  (branches: `cm-12.1`, `lineage-16.0` only)
- Kernel: `https://github.com/lineage16-suez/kernel_amazon_suez`
  (branches: `lineage-16.0-pre-upstream`, `lineage-16.0`, `master` only)
- Vendor blobs: `https://github.com/ivanlopezmolina/vendor_amazon_suez`
- XDA thread confirming no working 17.x/18.x build exists for suez, as of November 2025:
  `https://xdaforums.com/t/closed-fire-hd-10-7th-suez-custom-rom-we-are-looking-for-a-new-developer-to-create-a-custom-rom-for-fire-hd-10-2017-suez-help-us.4649464/`
- Blog post describing the earlier, unfinished LineageOS 17.1 attempt (source of section 4):
  `https://toaru-web.net/2024/04/17/fire-hd-10-2017-suez-unofficial-lineageos-17-1/`
- CI run that proved `lunch` works, and found the CI disk-space limit:
  `https://github.com/domgregori/lineage16-suez-build/actions/runs/35955880353`

## 10. Bottom line

The config stage works. No one has finished this port before. The real risk sits at the
device, kernel, and HAL compile stage. Expect WiFi, Bluetooth, AudioFX, or OpenSSL problems there.
Treat this as real porting work. Budget real time for source-level fixes, not a quick rebuild.
