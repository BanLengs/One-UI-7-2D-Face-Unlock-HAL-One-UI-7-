[README.md](https://github.com/user-attachments/files/35011452/README.md)
# Exynos7420 One UI 7 Face HAL（獵戶座 7420 系列 One UI 7 移植版人臉解鎖）

[English](README.en.md) | [繁體中文](README.md)

為「第三方移植 One UI 7（Android 15）到 Exynos 7420 Galaxy S6 / Note 5 系列」製作的 **2D 人臉解鎖 HAL**。
讓鎖屏、App 鎖、亮屏解鎖可以用前鏡頭的臉解鎖，註冊流程走系統原本的「設定 → 生物辨識 → 臉部辨識」。

> **⚠️ 先看這裡**
> - 這是 **2D 人臉辨識（弱安全等級）**，**用照片可能騙過**。只適合方便解鎖，**不要用在支付、銀行 App、密碼管理器**。
> - 需要 **已解鎖 Bootloader、Magisk root、SELinux Permissive** 的手機。會寫入 `/vendor` 兩個小檔案。
> - **請先備份 vendor 分割區**（教學裡有指令）。一切風險自負，作者不提供任何保證。
> - 本專案與 Samsung 無關。

目前版本：**v1.0**（分享版，內部 HAL 編號 v0.16：連續失敗鎖定預設關閉）

---

## 1. 支援機型表

我們需要你的回報來補完這張表。測過請開 Issue（模板在第 9 節）。

| 機型 | SoC | 系統 | 狀態 | 回報者 | 備註 |
|---|---|---|---|---|---|
| Galaxy S6 edge（SM-G9250） | Exynos 7420 | 移植 One UI 7（Eylem ROM，Android 15） | ✅ 可用 | 作者 | 開發機，註冊、解鎖、預覽、鎖定提示皆正常 |
| Galaxy Note 5 | Exynos 7420 | 同上 ROM | ✅ 可用 | 作者朋友 | 以分享版 v1.0 安裝成功 |
| Galaxy S6 / S6 edge+ 其他型號 | Exynos 7420 | 同上 ROM | ❓ 未測 | — | 預期可用，歡迎回報 |
| Galaxy S7 / S7 edge / Note 7 FE（Exynos 8890/8895） | Exynos 8xxx | 移植 One UI 7 | ❓ 未測 | — | 前鏡頭方向、`VENDOR` 分割區名稱可能不同 |
| 其他第三方 One UI 7 移植 | — | Android 15 | ❓ 未測 | — | 先跑第 5 節的相容性檢查 |
| One UI 8.x（Android 16）原廠或移植 | — | Android 16 | ❌ 預期不可用（未測） | — | 人臉 AIDL 與 Samsung 內部介面可能已改動；原廠機通常 SELinux Enforcing 且已有官方人臉 HAL |
| One UI 6 / Android 14 以下 | — | — | ❌ 不支援 | — | 人臉介面版本不同 |

狀態圖例：✅ 實測可用　⚠️ 部分可用　❌ 不可用　❓ 未測

---

## 2. 這個 HAL 做了什麼

```
設定 / 鎖屏 (Samsung keyguard、SemFaceService)
        │  AIDL (android.hardware.biometrics.face.IFace v4)
        ▼
 face-service.facehal  ←  本專案（原生 C++，跑在 /system_ext）
   ├─ Samsung 擴充層：ISehFace / ISehSession（Samsung 鎖屏走這條）
   ├─ 前鏡頭：NDK Camera2 + AImageReader（YUV_420_888 640x480）
   ├─ 偵測：YuNet（5 點特徵點）
   ├─ 姿勢檢查：用特徵點判斷是否正臉、是否上下顛倒
   ├─ 對齊：5 點相似變換（ArcFace 範本）→ 112x112
   ├─ 特徵：GhostFaceNet（512 維，L2 正規化）
   └─ 比對：與註冊範本做 cosine 相似度（預設門檻 0.45）
```

註冊時收集 5 張正臉樣本（可調），存成特徵向量（不存照片）。
預覽畫面走 `enrollWithOptions` 的 `surfacePreview`（所以 HAL 介面必須宣告為 v4）。

### 檔案會放在哪裡

| 檔案 | 位置 | 方式 |
|---|---|---|
| HAL 執行檔 | `/system_ext/bin/hw/android.hardware.biometrics.face-service.facehal` | Magisk 模組（systemless 疊加） |
| 4 個模型檔 | `/system_ext/etc/facehal/` | 同上 |
| 啟動腳本 `face_facehal.rc` | `/vendor/etc/init/` | **寫入 vendor（開機最早期就要讀）** |
| HAL 宣告 `face_facehal.xml` | `/vendor/etc/vintf/manifest/` | **寫入 vendor** |
| 註冊的臉特徵、設定檔 | `/data/vendor/face/` | 手機本機，不會被打包分享 |

為什麼不能只靠 Magisk 模組：`rc` 與 VINTF manifest 在 Magisk 掛載之前就會被系統讀取，所以這兩個小檔必須真的寫進 `/vendor`。

---

## 3. 使用的元件與授權

| 元件 | 用途 | 授權 | 來源 |
|---|---|---|---|
| [ncnn](https://github.com/Tencent/ncnn)（Tencent） | 神經網路推論（靜態連結，無 OpenMP） | BSD 3-Clause | 自行以 NDK r27c 編譯 |
| [YuNet](https://github.com/ShiqiYu/libfacedetection)（`yunet_2023mar`） | 人臉偵測 + 5 點特徵點 | MIT（OpenCV Zoo）／BSD 3-Clause（libfacedetection），以來源專案為準 | 轉換成 ncnn 格式 |
| [GhostFaceNet](https://github.com/HamadYA/GhostFaceNets)（`w1.3 s2`） | 人臉特徵（512 維） | MIT | 轉換成 ncnn 格式 |
| Android NDK r27c | 編譯工具鏈 | Apache 2.0 | Google |
| AOSP `aidl` 工具與 Face AIDL 介面 | 產生 binder 介面程式碼（`--version 4`） | Apache 2.0 | AOSP android-14 |
| Samsung `ISehFace` / `ISehSession` AIDL | 與 Samsung 鎖屏溝通 | — | 由 One UI 7 的 `SemFaceService` 反推（非官方，僅供互通） |
| Magisk | systemless 疊加 `/system_ext` | GPL-3.0 | topjohnwu/Magisk |

> **授權聲明**：完整的第三方授權文字請見 `THIRD_PARTY.txt`。
> **模型授權提醒**：模型權重的授權為上表所列，但部分人臉模型的**訓練資料集**可能有研究／非商業用途限制。本專案定位為個人、非商業使用，若你要做商用請自行查證。
> 本專案原始碼的授權：**MIT License**，Copyright (c) 2026 BanLeng（全文見 repo 根目錄的 `LICENSE`）。

---

## 4. 需求

- 已移植的 **One UI 7（Android 15、sdk 35）**，CPU 為 **arm64-v8a**
- **已解鎖 Bootloader + Magisk root**
- **SELinux Permissive**（`adb shell getenforce` 要顯示 `Permissive`）
- 系統內含 Samsung 人臉框架（`face` 服務與 `com.samsung.android.smartface.overlay`）
- 前鏡頭可以被 Android 相機正常開啟
- 電腦：adb（Linux 直接用；Windows 請用 WSL 或 Git Bash 執行 `.sh`）

---

## 5. 先做相容性檢查（只讀，不會改動手機）

手機開啟「USB 偵錯」接上電腦，依序執行：

**A. 基本資訊**
```bash
adb shell "getprop ro.product.model; getprop ro.build.version.release; getprop ro.build.version.sdk; getprop ro.product.cpu.abi; getprop ro.build.version.oneui; getenforce; ls -d /system_ext /system/system_ext 2>&1"
```

**B. Samsung 人臉框架與前鏡頭**
```bash
adb shell "service list | grep -i -E 'face|biometric'; echo ---; pm list packages | grep -i -E 'face|biometric'; echo ---; dumpsys media.camera | grep -i -E 'device@|facing' | head -12"
adb shell "dumpsys media.camera | grep -A1 'android.sensor.orientation' | head -6"
```

**C. root、Magisk、vendor 分割區**
```bash
adb shell "su -c 'id; ls /data/adb/modules; ls -l /dev/block/platform/15570000.ufs/by-name/ | grep -i -E vendor; ls /vendor/etc/vintf/manifest | head -5'"
```

### 判讀

| 項目 | 可以裝的情況 |
|---|---|
| `ro.build.version.sdk` | `35`（Android 15） |
| `ro.product.cpu.abi` | `arm64-v8a` |
| `getenforce` | `Permissive` |
| `service list` | 有 `face: [android.hardware.face.IFaceService]` |
| `pm list packages` | 有 `com.samsung.android.smartface.overlay` |
| 相機 | 有一個 `Facing: Front`；記下前鏡頭 `orientation`（S6 系列是 270） |
| `su -c id` | `uid=0(root)` |
| `by-name` | 有 `VENDOR`（你的路徑若不是 `15570000.ufs`，請在 Issue 回報） |

只要有一項不符合，**先不要裝**，把輸出貼到 Issue。

---

## 6. 安裝（adb 方式）

### 步驟 1：確認連線與 root
```bash
adb devices
adb shell "su -c id"
```
要看到一台 `device`，以及 `uid=0(root)`（手機若跳出 root 授權視窗請允許）。

### 步驟 2：備份 vendor（一定要做）
```bash
adb exec-out "su -c 'dd if=/dev/block/platform/15570000.ufs/by-name/VENDOR bs=4M'" | gzip -1 > vendor_backup.img.gz
ls -la vendor_backup.img.gz
```
`ls` 要看到幾百 MB。只有幾 KB 就是失敗了，不要繼續。

### 步驟 3：安裝
從 [Releases](../../releases) 下載 `facehal_v1.0.zip`，並對照 Release 說明頁上的 SHA256（`sha256sum facehal_v1.0.zip`），確認沒有下載中斷。

```bash
unzip facehal_v1.0.zip
cd facehal_v1.0
./install.sh
```
最後要看到 `INSTALL_OK`。若出現 `MISMATCH`，代表複製出錯，**不要重開機**，貼到 Issue。
如果你之前裝過檔名含 `g9250` 的測試版，`install.sh` 會自動清掉舊的模組和 vendor 檔，不需要先手動移除。

### 步驟 4：重開機
```bash
adb reboot
```

### 步驟 5：確認服務有啟動
```bash
adb shell "su -c 'pidof android.hardware.biometrics.face-service.facehal'"
```
要印出一個數字。

### 步驟 6：註冊臉
**設定 → 生物辨識 → 臉部辨識**，臉正對鏡頭、用平常拿手機的距離、光線均勻。進度條會走完；低頭、抬頭、太遠或太近、側臉會暫停等你調整。

### 步驟 7：測試
鎖屏後對著前鏡頭，約 1～2 秒內應該解鎖。

---

## 7. 測試清單（建議都試一輪再回報）

- [ ] 註冊能走到 100%，沒有卡在 99%
- [ ] 預覽畫面有出現；手機橫放或倒置時註冊仍能完成
- [ ] 鎖屏用自己的臉可以解鎖（記下大概幾秒）
- [ ] 別人的臉、側臉、遮臉**不會**解鎖
- [ ] 重開機後還能用（不用重新註冊）
- [ ] 日常使用幾小時沒有當機、發熱、耗電異常

---

## 8. 設定檔（可選）

設定放在手機的 `/data/vendor/face/cfg`（每行 `key=value`，每次操作時讀取，不用重開機）。

寫入範例：
```bash
adb shell "su -c 'sed -i /^thr=/d /data/vendor/face/cfg; echo thr=0.5 >> /data/vendor/face/cfg'"
```

| 鍵 | 預設 | 說明 |
|---|---|---|
| `thr` | 0.45 | 解鎖相似度門檻。調高更嚴格、更不容易被別人解開 |
| `samples` | 5 | 註冊時收集的樣本數 |
| `rot` | 3 | 前鏡頭畫面初始順時針旋轉（×90°）。S6 系列是 3 |
| `auth_ms` | 7000 | 單次解鎖嘗試的最長時間（毫秒）。搭配鎖定功能請設 3500（Samsung 約 4 秒會自己取消） |
| `min_face` / `max_face` | 120 / 230 | 註冊時臉寬（像素）範圍 |
| `auth_min_face` | 70 | 解鎖時最小臉寬 |
| `pitch_min` / `pitch_max` | 0.47 / 0.64 | 低頭／抬頭容許範圍（正臉約 0.55） |
| `vr_min` / `vr_max` | 0.95 / 1.30 | 臉的縱橫比容許範圍 |
| `enroll_sim_min` | 0.5 | 新樣本與第一張的最低相似度 |
| `enroll_fail` | 3 | 不合格樣本累積幾次後，註冊重新從 0 開始 |
| `stall_ms` | 12000 | 註冊停滯多久後重新開始 |
| `max_fails` | 0 | **連續失敗幾次後鎖定臉部解鎖**（0＝關閉，分享版預設）。例如 `3` |
| `lockout_ms` | 600000 | 鎖定逾時自動解除（毫秒，0＝不逾時）。用密碼/圖案解鎖進系統後也會自動解除 |
| `dbg` | 1 | log 詳細程度 |

清除失敗計數：`adb shell "su -c 'echo 0 > /data/vendor/face/fails'"`

---

## 9. 回報測試結果（開 Issue）

請附上：

```
機型：
ROM 名稱與版本：
Android / One UI 版本：
SoC：
getenforce：
前鏡頭 orientation：
安裝是否出現 INSTALL_OK：
pidof 結果（有/無）：
註冊：成功 / 失敗（卡在哪）
解鎖：成功 / 失敗（約幾秒）
其他現象：
```

以及這兩個輸出：

```bash
adb shell "su -c 'ls /data/adb/modules; pidof android.hardware.biometrics.face-service.facehal'"
adb logcat -b all -d -v threadtime | grep FaceG9250 | tail -80
```

第一次安裝註冊前，建議先 `adb logcat -c` 清 log，重現問題後再抓，內容會比較乾淨。

---

## 10. 疑難排解

| 現象 | 可能原因與處理 |
|---|---|
| 設定裡沒有「臉部辨識」或註冊一開始就失敗 | 服務沒起來。檢查 `pidof`，沒有數字就看 log 裡 `addService` 那行；確認 `/vendor/etc/vintf/manifest/face_facehal.xml` 與 `/vendor/etc/init/face_facehal.rc` 存在 |
| 預覽沒畫面 | 前鏡頭被其他程序占用，或相機模組沒修好；先確認系統相機前鏡頭能用 |
| 註冊時一直說臉不正／方向錯 | 前鏡頭方向和 S6 不同，把 `rot` 試 0～3；log 看 `cam: front id=... sensorOrientation=...` |
| 解鎖很慢（>3 秒） | 這台 CPU 偵測一張約 340 ms；開相機約 1.1 秒是固有延遲 |
| 自己的臉常解不開 | 在平常拿手機的距離**重新註冊**；log 看 `auth: sim=`，若常在 0.5～0.6，註冊時的距離和使用時差太多 |
| 失敗鎖定之後臉不恢復 | 確認 `max_fails` 有設；用圖案／密碼解鎖進系統後約 2～3 秒會自動清除，也可手動清 `fails` |
| 重開機後服務沒了 | 確認 Magisk 模組 `facehal` 沒被停用，且 `/vendor` 的兩個檔案還在（有些 ROM 更新會覆蓋 vendor） |

---

## 11. 還原

```bash
cd facehal_v1.0
./uninstall.sh
adb reboot
```
會移除 Magisk 模組並刪除寫入 `/vendor` 的兩個檔案。你的臉資料在 `/data/vendor/face/`，需要的話可手動清除：
```bash
adb shell "su -c 'rm -rf /data/vendor/face'"
```
如果連開機都出問題：用 TWRP 把步驟 2 備份的 `vendor_backup.img.gz` 還原回 vendor 分割區。

---

## 12. 已知限制

- 2D 辨識，**照片、螢幕上的臉可能解鎖**（目前沒有活體偵測）
- 暗處、逆光表現會變差；沒有紅外線補光
- 多使用者／工作資料夾（非 user 0）未測試
- 只在 Exynos 7420 的 S6 / Note 5 系列驗證過
- 「臉部辨識」只作為弱生物辨識（WEAK），Samsung 會限制它不能用於需要強生物辨識的場景

---

## 13. 給開發者

- Face AIDL 以 `--version 4` 編譯，框架才會呼叫 `enrollWithOptions`（帶 `surfacePreview`）
- Samsung 鎖屏的解鎖呼叫的是 `ISehSession` 的 **transaction 1**：`authenticate(long operationId, int securityMode, byte[] fido) → ICancellationSignal`，不是標準的 `ISession.authenticate`
- `ISehSession` 擴充必須同時暴露標準 `ISession`（用 `AIBinder_setExtension`）
- 註冊進度：Samsung 介面把 `remaining` 當作 `100 − percent`，所以進度要以百分比反算
- 失敗鎖定由 HAL 負責（`onLockoutPermanent` / `onLockoutCleared`）；Samsung 不會呼叫 `resetLockout`，所以 HAL 自己輪詢螢幕鎖狀態（`dumpsys window policy`）來判斷「已解鎖」
- 建置：NDK r27c、aidl 工具（AOSP android-14）、靜態 ncnn（`NCNN_OPENMP=OFF`、`NCNN_SIMPLEOCV=ON`、`NCNN_PLATFORM_API=OFF`）；編譯旗標 `-DBINDER_STABILITY_SUPPORT -DSYSTEM_PROC`

---

## 14. 致謝

- Tencent ncnn、ShiqiYu/libfacedetection（YuNet）、HamadYA/GhostFaceNets
- 所有幫忙測試與回報的人
