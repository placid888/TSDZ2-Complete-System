# TSDZ2 Complete System

整合 TSDZ2 開源系統的單一開發倉庫（monorepo）。目標是讓 STM8 馬達控制器、TSDZ2 ESP32 Gateway、Android App，以及 ESP32-S3 Fake STM8 測試器可以在同一個專案中共同開發、驗證與追蹤。

## 專案架構

```text
TSDZ2-Complete-System/
├── esp32-gateway/    # 原作者 TSDZ2-ESP32 firmware，目標硬體為原作者 ESP32 PCB
├── android-app/      # 原作者 TSDZ2 Android App
├── fake-stm8/        # ESP32-S3 Fake STM8 / Motor Controller Simulator
└── docs/             # Protocol、硬體、OTA、測試與架構文件
```

## 系統資料流

```text
ESP32-S3 Fake STM8
        |
        | TSDZ2 UART protocol
        v
TSDZ2 ESP32 Gateway / 原作者 PCB
        |
        | BLE
        v
TSDZ2 Android App
```

## 開發原則

1. `esp32-gateway/` 與 `android-app/` 必須先從原作者官方 repository 的乾淨版本匯入。
2. 匯入時記錄 upstream repository、branch/tag 與 commit SHA，作為可驗證的 baseline。
3. 在 baseline 建立完成前，不修改原作者 production source。
4. `fake-stm8/` 為獨立測試工具，不宣稱執行或模擬 STM8 CPU；它只模擬 TSDZ2 STM8 UART protocol 與馬達控制器行為。
5. 未經原始碼驗證的 UART byte、CRC、UUID、OTA/bootloader 行為不得猜測。
6. 所有後續修改應保持可追蹤，方便與 upstream 原作者版本比較。

## 三個子專案

### esp32-gateway

正式車載 Gateway。最終目標硬體是依原作者設計製作的 TSDZ2 ESP32 PCB，而不是一般市售 ESP32 DevKit。負責 STM8 UART、BLE、Android 通訊，以及後續 OTA、診斷與擴充功能。

### android-app

以原作者 TSDZ2 Android App 為乾淨基線。作為主要設定、監控、診斷與未來 OTA 操作介面。

### fake-stm8

使用一般 ESP32-S3 開發板建立桌上測試設備。透過 UART 模擬 STM8/TSDZ2 Motor Controller，讓真正的 TSDZ2 ESP32 PCB 與 Android App 可以在沒有馬達與實車的情況下進行整合測試。

預定由電腦 USB Serial 控制，例如設定速度、踏頻、扭力、電流、電壓、溫度與故障狀態；同時記錄 ESP32 Gateway 傳來的控制封包。

## 第一階段

- 鎖定原作者 ESP32 upstream baseline
- 鎖定原作者 Android upstream baseline
- 完整核對 STM8 ↔ ESP32 UART protocol
- 找出並驗證 CRC16 實作
- 建立 UART protocol contract
- 建立 ESP32-S3 Fake STM8 v1
- 使用真正的 TSDZ2 ESP32 PCB + Android App 完成桌上整合測試

## Upstream

Baseline 匯入完成後，會在 `docs/UPSTREAM_BASELINE.md` 記錄原作者 repository 與精確 commit SHA。這份紀錄將作為後續判斷「原作者行為」與「本專案修改」的依據。

## Status

Repository scaffold established. Upstream source import尚未開始。
