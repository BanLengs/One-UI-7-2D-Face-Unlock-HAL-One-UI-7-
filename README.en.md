[README.en.md](https://github.com/user-attachments/files/35010246/README.en.md)
# Exynos7420 One UI 7 Face HAL

[English](README.en.md) | [繁體中文](README.md)

A **2D face unlock HAL** for third-party **One UI 7 (Android 15) ports on Exynos 7420 Galaxy S6 / Note 5 family devices**.
It enables face unlock on the lock screen, in app lock, and for wake-to-unlock using the front camera. Enrollment goes through the stock **Settings → Biometrics → Face recognition** flow.

> **⚠️ Read this first**
> - This is **2D face recognition (weak security class)**. **A photo may fool it.** Use it for convenience only. **Do not use it for payments, banking apps, or password managers.**
> - Requires a phone with an **unlocked bootloader, Magisk root, and SELinux Permissive**. It writes two small files into `/vendor`.
> - **Back up your vendor partition first** (commands below). Use at your own risk; no warranty of any kind.
> - This project is not affiliated with Samsung.

Current version: **v0.16** (shared edition: lockout after repeated failures is **off** by default)

---

## 1. Compatibility table

Your reports help complete this table. If you tested it, please open an Issue (template in section 9).

| Device | SoC | System | Status | Reporter | Notes |
|---|---|---|---|---|---|
| Galaxy S6 edge (SM-G9250) | Exynos 7420 | Ported One UI 7 (Eylem ROM, Android 15) | ✅ Works | Author | Dev device. Enrollment, unlock, preview and lockout message all work |
| Galaxy Note 5 | Exynos 7420 | Same ROM | ✅ Works | Author's friend | Installed successfully with shared edition v0.16 |
| Other Galaxy S6 / S6 edge+ models | Exynos 7420 | Same ROM | ❓ Untested | — | Expected to work, reports welcome |
| Galaxy S7 / S7 edge / Note 7 FE (Exynos 8890/8895) | Exynos 8xxx | Ported One UI 7 | ❓ Untested | — | Front camera orientation and `VENDOR` partition name may differ |
| Other third-party One UI 7 ports | — | Android 15 | ❓ Untested | — | Run the compatibility check in section 5 first |
| One UI 8.x (Android 16), stock or ported | — | Android 16 | ❌ Expected not to work (untested) | — | Face AIDL and Samsung internal interfaces may have changed. Stock devices usually have SELinux Enforcing and an official face HAL |
| One UI 6 / Android 14 and older | — | — | ❌ Not supported | — | Different face interface version |

Legend: ✅ works　⚠️ partially works　❌ does not work　❓ untested

---

## 2. What this HAL does

```
Settings / Lock screen (Samsung keyguard, SemFaceService)
        │  AIDL (android.hardware.biometrics.face.IFace v4)
        ▼
 face-service.g9250  ←  this project (native C++, runs from /system_ext)
   ├─ Samsung extension layer: ISehFace / ISehSession (the Samsung lock screen uses this path)
   ├─ Front camera: NDK Camera2 + AImageReader (YUV_420_888 640x480)
   ├─ Detection: YuNet (5 facial landmarks)
   ├─ Pose check: landmarks decide whether the face is frontal and upright
   ├─ Alignment: 5-point similarity transform (ArcFace template) → 112x112
   ├─ Embedding: GhostFaceNet (512-d, L2-normalized)
   └─ Matching: cosine similarity against enrolled templates (default threshold 0.45)
```

Enrollment collects 5 frontal samples (configurable) and stores **feature vectors, not photos**.
The enrollment preview uses the `surfacePreview` field of `enrollWithOptions`, which is why the HAL interface must be declared as **v4**.

### Where files go

| File | Location | Method |
|---|---|---|
| HAL executable | `/system_ext/bin/hw/android.hardware.biometrics.face-service.g9250` | Magisk module (systemless overlay) |
| 4 model files | `/system_ext/etc/facehal/` | Same |
| Service script `face_g9250.rc` | `/vendor/etc/init/` | **Written into vendor (read very early in boot)** |
| HAL declaration `face_g9250.xml` | `/vendor/etc/vintf/manifest/` | **Written into vendor** |
| Enrolled face features, config | `/data/vendor/face/` | Stays on the phone, never packaged or shared |

Why a Magisk module alone is not enough: the `rc` file and the VINTF manifest are read before Magisk mounts its overlays, so those two small files have to really live in `/vendor`.

---

## 3. Components and licenses

| Component | Purpose | License | Source |
|---|---|---|---|
| [ncnn](https://github.com/Tencent/ncnn) (Tencent) | Neural network inference (statically linked, no OpenMP) | BSD 3-Clause | Built with NDK r27c |
| [YuNet](https://github.com/ShiqiYu/libfacedetection) (`yunet_2023mar`) | Face detection + 5 landmarks | MIT | Converted to ncnn format |
| [GhostFaceNet](https://github.com/HamadYA/GhostFaceNets) (`w1.3 s2`) | Face embedding (512-d) | MIT | Converted to ncnn format |
| Android NDK r27c | Toolchain | Apache 2.0 | Google |
| AOSP `aidl` tool and Face AIDL interfaces | Generates binder code (`--version 4`) | Apache 2.0 | AOSP android-14 |
| Samsung `ISehFace` / `ISehSession` AIDL | Talks to the Samsung lock screen | — | Reverse-engineered from One UI 7 `SemFaceService` (unofficial, for interoperability only) |
| Magisk | Systemless overlay on `/system_ext` | GPL-3.0 | topjohnwu/Magisk |

> **Model license note**: the model weights are under the licenses listed above, but the **training datasets** of some face models may carry research / non-commercial restrictions. This project is intended for personal, non-commercial use. If you plan commercial use, check this yourself.
> Source code license: **(add the license you choose to this repo, for example MIT)**

---

## 4. Requirements

- A ported **One UI 7 (Android 15, sdk 35)** system on **arm64-v8a**
- **Unlocked bootloader + Magisk root**
- **SELinux Permissive** (`adb shell getenforce` must print `Permissive`)
- Samsung's face framework in the system (the `face` service and `com.samsung.android.smartface.overlay`)
- A front camera that opens normally in Android
- On the PC: adb (Linux works directly; on Windows use WSL or Git Bash to run the `.sh` scripts)

---

## 5. Run the compatibility check first (read-only, changes nothing)

Enable USB debugging, connect the phone, then run in order:

**A. Basic info**
```bash
adb shell "getprop ro.product.model; getprop ro.build.version.release; getprop ro.build.version.sdk; getprop ro.product.cpu.abi; getprop ro.build.version.oneui; getenforce; ls -d /system_ext /system/system_ext 2>&1"
```

**B. Samsung face framework and front camera**
```bash
adb shell "service list | grep -i -E 'face|biometric'; echo ---; pm list packages | grep -i -E 'face|biometric'; echo ---; dumpsys media.camera | grep -i -E 'device@|facing' | head -12"
adb shell "dumpsys media.camera | grep -A1 'android.sensor.orientation' | head -6"
```

**C. root, Magisk, vendor partition**
```bash
adb shell "su -c 'id; ls /data/adb/modules; ls -l /dev/block/platform/15570000.ufs/by-name/ | grep -i -E vendor; ls /vendor/etc/vintf/manifest | head -5'"
```

### How to read the results

| Item | OK to install when |
|---|---|
| `ro.build.version.sdk` | `35` (Android 15) |
| `ro.product.cpu.abi` | `arm64-v8a` |
| `getenforce` | `Permissive` |
| `service list` | has `face: [android.hardware.face.IFaceService]` |
| `pm list packages` | has `com.samsung.android.smartface.overlay` |
| Camera | one device with `Facing: Front`; note the front `orientation` (270 on the S6 family) |
| `su -c id` | `uid=0(root)` |
| `by-name` | contains `VENDOR` (if your path is not under `15570000.ufs`, please report it in an Issue) |

If any item does not match, **do not install yet**. Paste the output into an Issue.

---

## 6. Installation (adb)

### Step 1: Check the connection and root
```bash
adb devices
adb shell "su -c id"
```
You should see one `device` and `uid=0(root)`. If the phone shows a root permission prompt, allow it.

### Step 2: Back up vendor (mandatory)
```bash
adb exec-out "su -c 'dd if=/dev/block/platform/15570000.ufs/by-name/VENDOR bs=4M'" | gzip -1 > vendor_backup.img.gz
ls -la vendor_backup.img.gz
```
`ls` should show hundreds of MB. If it shows only a few KB, the backup failed. Do not continue.

### Step 3: Install
Download `G9250_facehal_share_v0.16.zip` from [Releases](../../releases) and make sure the download is complete (file size 16938735 bytes).

```bash
unzip G9250_facehal_share_v0.16.zip
cd G9250_facehal/facehal
./install.sh
```
The last line must be `INSTALL_OK`. If you see `MISMATCH`, a copy failed. **Do not reboot** and report it in an Issue.

### Step 4: Reboot
```bash
adb reboot
```

### Step 5: Check that the service is running
```bash
adb shell "su -c 'pidof android.hardware.biometrics.face-service.g9250'"
```
It must print a number.

### Step 6: Enroll your face
Go to **Settings → Biometrics → Face recognition**. Face the camera directly, hold the phone at your usual distance, and use even lighting. The progress bar runs to the end. Looking down or up, being too close or too far, or turning sideways pauses enrollment until you adjust.

### Step 7: Test
Lock the screen and look at the front camera. It should unlock within about 1–2 seconds.

---

## 7. Test checklist (please try everything once before reporting)

- [ ] Enrollment reaches 100% and does not stall at 99%
- [ ] The preview shows; enrollment still completes with the phone held sideways or upside down
- [ ] Your own face unlocks the lock screen (note roughly how many seconds)
- [ ] Other faces, side views, or a covered face do **not** unlock
- [ ] Still works after a reboot (no re-enrollment needed)
- [ ] No crashes, heat, or abnormal battery drain after a few hours of normal use

---

## 8. Configuration (optional)

Settings live in `/data/vendor/face/cfg` on the phone (one `key=value` per line, read on every operation, no reboot needed).

Example:
```bash
adb shell "su -c 'sed -i /^thr=/d /data/vendor/face/cfg; echo thr=0.5 >> /data/vendor/face/cfg'"
```

| Key | Default | Description |
|---|---|---|
| `thr` | 0.45 | Unlock similarity threshold. Higher is stricter and harder for other people to unlock |
| `samples` | 5 | Samples collected during enrollment |
| `rot` | 3 | Initial clockwise rotation of the front camera frame (×90°). 3 on the S6 family |
| `auth_ms` | 7000 | Maximum time of one unlock attempt (ms). If you use lockout, set 3500 (Samsung cancels after about 4 s on its own) |
| `min_face` / `max_face` | 120 / 230 | Face width range in pixels during enrollment |
| `auth_min_face` | 70 | Minimum face width when unlocking |
| `pitch_min` / `pitch_max` | 0.47 / 0.64 | Allowed look-down / look-up range (frontal is about 0.55) |
| `vr_min` / `vr_max` | 0.95 / 1.30 | Allowed face aspect range |
| `enroll_sim_min` | 0.5 | Minimum similarity of a new sample to the first one |
| `enroll_fail` | 3 | Bad samples before enrollment restarts from 0 |
| `stall_ms` | 12000 | Enrollment restarts after this long without progress |
| `max_fails` | 0 | **Consecutive failures before face unlock is locked out** (0 = off, the shared default). For example `3` |
| `lockout_ms` | 600000 | Lockout expires on its own after this time (ms, 0 = never). Unlocking with PIN or pattern also clears it |
| `dbg` | 1 | Log verbosity |

Clear the failure counter: `adb shell "su -c 'echo 0 > /data/vendor/face/fails'"`

---

## 9. Reporting results (open an Issue)

Please include:

```
Device:
ROM name and version:
Android / One UI version:
SoC:
getenforce:
Front camera orientation:
Did the install print INSTALL_OK:
pidof result (yes/no):
Enrollment: success / failure (where it got stuck)
Unlock: success / failure (about how many seconds)
Other observations:
```

and these two outputs:

```bash
adb shell "su -c 'ls /data/adb/modules; pidof android.hardware.biometrics.face-service.g9250'"
adb logcat -b all -d -v threadtime | grep FaceG9250 | tail -80
```

Before enrolling for the first time, run `adb logcat -c` to clear the log, then capture it after reproducing the problem. The output will be much cleaner.

---

## 10. Troubleshooting

| Symptom | Likely cause and fix |
|---|---|
| No "Face recognition" in Settings, or enrollment fails right away | The service did not start. Check `pidof`; if there is no number, look at the `addService` line in the log. Confirm `/vendor/etc/vintf/manifest/face_g9250.xml` and `/vendor/etc/init/face_g9250.rc` exist |
| No preview | The front camera is held by another process or the camera fix is missing. First confirm the stock camera's front camera works |
| Enrollment keeps saying the face is not frontal / wrong direction | Your front camera orientation differs from the S6. Try `rot` 0–3 and check `cam: front id=... sensorOrientation=...` in the log |
| Unlock is slow (>3 s) | Detection takes about 340 ms per frame on this CPU, and opening the camera takes about 1.1 s regardless |
| Your own face often fails | **Re-enroll** at the distance you normally hold the phone. If `auth: sim=` in the log is often 0.5–0.6, your enrollment distance differs too much from your usage distance |
| Face does not recover after a lockout | Confirm `max_fails` is set. After you unlock with pattern or password, it clears itself in about 2–3 s. You can also clear `fails` manually |
| Service gone after reboot | Make sure the Magisk module `facehal_g9250` is not disabled and the two `/vendor` files are still there (some ROM updates overwrite vendor) |

---

## 11. Uninstall

```bash
cd G9250_facehal/facehal
./uninstall.sh
adb reboot
```
This removes the Magisk module and deletes the two files written to `/vendor`. Your face data is in `/data/vendor/face/`; remove it manually if you want:
```bash
adb shell "su -c 'rm -rf /data/vendor/face'"
```
If the phone does not even boot: restore the `vendor_backup.img.gz` from step 2 to the vendor partition with TWRP.

---

## 12. Known limitations

- 2D recognition: **a photo or a face on a screen may unlock it** (no liveness detection yet)
- Worse in the dark or against backlight; there is no infrared illuminator
- Multi-user / work profiles (non user 0) are untested
- Only verified on the Exynos 7420 S6 / Note 5 family
- Face unlock is a weak biometric class; Samsung restricts it from scenarios that require strong biometrics

---

## 13. For developers

- The Face AIDL is compiled with `--version 4` so the framework calls `enrollWithOptions` (which carries `surfacePreview`)
- The Samsung lock screen unlocks through **transaction 1** of `ISehSession`: `authenticate(long operationId, int securityMode, byte[] fido) → ICancellationSignal`, not the standard `ISession.authenticate`
- The `ISehSession` extension must also expose the standard `ISession` (via `AIBinder_setExtension`)
- Enrollment progress: the Samsung UI treats `remaining` as `100 − percent`, so progress is reported accordingly
- Lockout is the HAL's job (`onLockoutPermanent` / `onLockoutCleared`). Samsung never calls `resetLockout`, so the HAL polls the keyguard state (`dumpsys window policy`) to notice that the device was unlocked
- Build: NDK r27c, the aidl tool (AOSP android-14), static ncnn (`NCNN_OPENMP=OFF`, `NCNN_SIMPLEOCV=ON`, `NCNN_PLATFORM_API=OFF`); compile flags `-DBINDER_STABILITY_SUPPORT -DSYSTEM_PROC`

---

## 14. Credits

- Tencent ncnn, ShiqiYu/libfacedetection (YuNet), HamadYA/GhostFaceNets
- Everyone who tested and reported results
