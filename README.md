# Firefox YouTube 外掛安裝指南

這是一份給一般使用者看的 Firefox YouTube 外掛安裝指南。

本指南不包含自製外掛原始碼，主要協助你安裝已由 Mozilla Firefox Add-ons（AMO）簽署的獨立外掛。這種方式比把多個外掛硬合併成一個版本更容易更新，也比較不容易互相衝突。

## 可以做什麼

安裝下面的外掛後，可以取得這些功能：

| 功能 | 外掛 |
| --- | --- |
| 攔截 YouTube 廣告 | [uBlock Origin](https://addons.mozilla.org/firefox/addon/ublock-origin/) |
| 跳過影片業配、片頭、片尾 | [SponsorBlock](https://addons.mozilla.org/firefox/addon/sponsorblock/) |
| 顯示 YouTube 倒讚估算 | [Return YouTube Dislike](https://addons.mozilla.org/firefox/addon/return-youtube-dislikes/) |
| 調整 YouTube 播放速度 `0.1x～4x` | [Custom Speed Controller for YouTube](https://addons.mozilla.org/firefox/addon/custom-speed-control-youtube/) |
| YouTube 環境光 | [Ambient light for YouTube](https://addons.mozilla.org/firefox/addon/ambient-light-for-youtube/) |

### 功能限制

- Return YouTube Dislike 顯示的是估算值，不是 YouTube 官方數字。
- SponsorBlock 只能處理資料庫中已有標記的影片片段。
- uBlock Origin 不保證永久攔截所有 YouTube 廣告；YouTube 可能隨時更改網站程式。
- 環境光會增加 GPU 使用量，低階電腦或手機可以降低效果強度。
- 本指南不提供影片下載功能。
- iPhone／iPad 版 Firefox 不支援一般 Firefox 擴充套件安裝。

## 開始前：安裝 Firefox

- [Windows](https://www.mozilla.org/firefox/windows/)
- [macOS](https://www.mozilla.org/firefox/mac/)
- [Linux](https://www.mozilla.org/firefox/linux/)
- [Android](https://www.mozilla.org/firefox/browsers/mobile/android/)

如果已經安裝 Firefox，可以直接跳到下一節。

## 方法一：手動安裝（最推薦）

這是最安全、最容易移除的方法。每個外掛都從 Mozilla 官方 Add-ons 網站安裝。

### 電腦版 Windows、macOS、Linux

1. 開啟 Firefox。
2. 依序開啟上方五個外掛連結。
3. 在每個頁面按「新增至 Firefox」或「Add to Firefox」。
4. 閱讀權限提示後按「新增」。
5. 安裝完成後，可在網址列輸入 `about:addons` 查看外掛。
6. 重新開啟 YouTube 影片確認功能。

Firefox 會分別詢問每個外掛的權限，這是正常的安全機制。

### 如何確認已成功

在 Firefox 網址列輸入：

```text
about:addons
```

確認清單中出現：

- uBlock Origin
- SponsorBlock
- Return YouTube Dislike
- Custom Speed Controller for YouTube
- Ambient light for YouTube

接著開啟 YouTube 影片：

- 播放器附近應該能看到速度控制功能。
- 影片下方應該能看到倒讚估算。
- 有業配標記的影片會出現 SponsorBlock 控制。
- 開啟 Ambient light 後，影片周圍會產生環境光。

## 方法二：Windows PowerShell 一次安裝

這個方法會使用 Firefox 的 `policies.json`，讓 Firefox 啟動時安裝多個 AMO 外掛。

注意：

- 需要系統管理員權限。
- Firefox 必須完全關閉後重新開啟。
- 外掛可能會被 Firefox 視為政策管理，之後不能直接從一般介面移除。
- 只建議在自己的電腦使用。

請將以下內容儲存成 `install-firefox-youtube-addons.ps1`，再用 PowerShell 執行：

```powershell
$ErrorActionPreference = 'Stop'

$identity = [Security.Principal.WindowsIdentity]::GetCurrent()
$principal = New-Object Security.Principal.WindowsPrincipal($identity)
$adminRole = [Security.Principal.WindowsBuiltInRole]::Administrator

if (-not $principal.IsInRole($adminRole)) {
    Start-Process powershell.exe -Verb RunAs -ArgumentList @(
        '-NoProfile'
        '-ExecutionPolicy', 'Bypass'
        '-File', "`"$PSCommandPath`""
    ) -Wait
    exit $LASTEXITCODE
}

$firefoxExe = @(
    "$env:ProgramFiles\Mozilla Firefox\firefox.exe"
    "${env:ProgramFiles(x86)}\Mozilla Firefox\firefox.exe"
    "$env:LOCALAPPDATA\Mozilla Firefox\firefox.exe"
) | Where-Object { Test-Path -LiteralPath $_ } | Select-Object -First 1

if (-not $firefoxExe) {
    if (-not (Get-Command winget.exe -ErrorAction SilentlyContinue)) {
        throw '找不到 Firefox，也找不到 winget。'
    }

    winget install --id Mozilla.Firefox -e --accept-package-agreements --accept-source-agreements --silent

    $firefoxExe = @(
        "$env:ProgramFiles\Mozilla Firefox\firefox.exe"
        "${env:ProgramFiles(x86)}\Mozilla Firefox\firefox.exe"
        "$env:LOCALAPPDATA\Mozilla Firefox\firefox.exe"
    ) | Where-Object { Test-Path -LiteralPath $_ } | Select-Object -First 1
}

if (-not $firefoxExe) {
    throw 'Firefox 安裝後仍找不到 firefox.exe。'
}

$firefoxDir = Split-Path -Parent $firefoxExe
$policyDir = Join-Path $firefoxDir 'distribution'
$policyFile = Join-Path $policyDir 'policies.json'
New-Item -ItemType Directory -Force -Path $policyDir | Out-Null

$policy = @{
    policies = @{
        Extensions = @{
            Install = @(
                'https://addons.mozilla.org/firefox/downloads/latest/ublock-origin/latest.xpi'
                'https://addons.mozilla.org/firefox/downloads/latest/sponsorblock/latest.xpi'
                'https://addons.mozilla.org/firefox/downloads/latest/return-youtube-dislikes/latest.xpi'
                'https://addons.mozilla.org/firefox/downloads/latest/custom-speed-control-youtube/latest.xpi'
                'https://addons.mozilla.org/firefox/downloads/latest/ambient-light-for-youtube/latest.xpi'
            )
        }
    }
} | ConvertTo-Json -Depth 8

Set-Content -LiteralPath $policyFile -Value $policy -Encoding UTF8
if (-not (Test-Path -LiteralPath $policyFile)) { throw 'policies.json 建立失敗。' }
Write-Host "完成：$policyFile"
```

執行方式：

```powershell
powershell -ExecutionPolicy Bypass -File .\install-firefox-youtube-addons.ps1
```

看到 UAC 視窗時按「是」。

## 方法三：macOS Terminal 一次設定

macOS 的 Firefox 政策檔位於 Firefox 應用程式內，因此需要輸入登入密碼執行 `sudo`。

先完全關閉 Firefox，再開啟 Terminal，貼上：

```bash
FIREFOX_DIR="/Applications/Firefox.app/Contents/Resources"
POLICY_DIR="$FIREFOX_DIR/distribution"
POLICY_FILE="$POLICY_DIR/policies.json"

sudo mkdir -p "$POLICY_DIR"

sudo tee "$POLICY_FILE" >/dev/null <<'JSON'
{
  "policies": {
    "Extensions": {
      "Install": [
        "https://addons.mozilla.org/firefox/downloads/latest/ublock-origin/latest.xpi",
        "https://addons.mozilla.org/firefox/downloads/latest/sponsorblock/latest.xpi",
        "https://addons.mozilla.org/firefox/downloads/latest/return-youtube-dislikes/latest.xpi",
        "https://addons.mozilla.org/firefox/downloads/latest/custom-speed-control-youtube/latest.xpi",
        "https://addons.mozilla.org/firefox/downloads/latest/ambient-light-for-youtube/latest.xpi"
      ]
    }
  }
}
JSON

test -f "$POLICY_FILE" && echo "完成：$POLICY_FILE"
```

重新開啟 Firefox 後，在 `about:addons` 確認外掛。

## 方法四：Linux Terminal 一次設定

先完全關閉 Firefox。以下指令會自動尋找常見的 Firefox 安裝位置：

```bash
set -eu

for dir in \
  /usr/lib/firefox \
  /usr/lib/firefox-esr \
  /opt/firefox \
  /usr/local/lib/firefox
do
  if [ -x "$dir/firefox" ]; then
    FIREFOX_DIR="$dir"
    break
  fi
done

: "${FIREFOX_DIR:?找不到 Firefox 安裝目錄，請改用手動安裝方式}"

POLICY_DIR="$FIREFOX_DIR/distribution"
POLICY_FILE="$POLICY_DIR/policies.json"
sudo mkdir -p "$POLICY_DIR"

sudo tee "$POLICY_FILE" >/dev/null <<'JSON'
{
  "policies": {
    "Extensions": {
      "Install": [
        "https://addons.mozilla.org/firefox/downloads/latest/ublock-origin/latest.xpi",
        "https://addons.mozilla.org/firefox/downloads/latest/sponsorblock/latest.xpi",
        "https://addons.mozilla.org/firefox/downloads/latest/return-youtube-dislikes/latest.xpi",
        "https://addons.mozilla.org/firefox/downloads/latest/custom-speed-control-youtube/latest.xpi",
        "https://addons.mozilla.org/firefox/downloads/latest/ambient-light-for-youtube/latest.xpi"
      ]
    }
  }
}
JSON

test -f "$POLICY_FILE" && echo "完成：$POLICY_FILE"
```

如果使用 Snap、Flatpak 或發行版特殊套件，政策檔位置可能不同；此時建議使用「手動安裝」。

## Android Firefox 安裝方式

Android 不能使用 Windows、macOS 或 Linux 的 PowerShell／Terminal 指令。請直接在手機完成：

1. 安裝最新版 [Firefox for Android](https://www.mozilla.org/firefox/browsers/mobile/android/)。
2. 開啟 Firefox，點右上角「⋮」。
3. 進入「設定」。
4. 滑到最下方，開啟「關於 Firefox」。
5. 快速點擊 Firefox 圖示 5 次，啟用外掛安裝選項。
6. 回到「設定」。
7. 選擇「從檔案安裝擴充功能」。
8. 從 AMO 或手機檔案選擇已簽署的 XPI。
9. 按「新增」。

Android 只會顯示相容的 Firefox 外掛。如果找不到某個外掛，通常是該外掛尚未宣告支援 Android。

## 移除 Windows／macOS／Linux 的政策安裝

先完全關閉 Firefox，再刪除 `distribution/policies.json`：

### Windows（系統管理員 PowerShell）

```powershell
Remove-Item "C:\Program Files\Mozilla Firefox\distribution\policies.json"
```

### macOS

```bash
sudo rm "/Applications/Firefox.app/Contents/Resources/distribution/policies.json"
```

### Linux

```bash
sudo rm /usr/lib/firefox/distribution/policies.json
```

刪除後重新開啟 Firefox，再到 `about:addons` 手動移除外掛。

## 常見問題

### 顯示「附加元件尚未經過驗證，無法安裝」

這表示 XPI 沒有 Mozilla 簽署。一般 Firefox Release／Beta 不能直接安裝未簽署外掛。請改用 AMO 版本，或使用 Firefox Developer Edition、Nightly、ESR 進行開發測試。

### YouTube 播放器一直轉圈

先暫時停用其他 YouTube 修改類外掛，再重新整理頁面。不要同時使用兩個速度控制器、兩個環境光外掛或兩個 SponsorBlock 實作。

### 外掛已安裝但功能沒有出現

1. 開啟 `about:addons` 確認外掛沒有被停用。
2. 重新整理 YouTube。
3. 完全關閉 Firefox 後重新開啟。
4. 檢查外掛是否有 YouTube 網站權限。
5. 先停用其他可能修改 YouTube 的外掛。

### 為什麼不直接使用整合版？

多個完整外掛共用同一個 YouTube 頁面時，可能發生 CSS、事件監聽器、權限、設定儲存與更新衝突。分開安裝雖然需要多按幾次，但比較容易確認是哪個外掛發生問題。

## 隱私與權限

請在每個 AMO 頁面查看最新權限說明。Return YouTube Dislike 會查詢影片識別資料以取得倒讚統計；SponsorBlock 會查詢影片片段資料；Ambient Light 主要在本機處理影片畫面。

本指南不收集使用者資料，也不提供任何付費服務。

## 來源

- [Mozilla Firefox Add-ons](https://addons.mozilla.org/firefox/)
- [Firefox 官方安裝說明](https://support.mozilla.org/kb/find-and-install-add-ons-add-features-to-firefox)
- [Firefox 政策範本](https://mozilla.github.io/policy-templates/)
- [Firefox Extension Workshop：簽署與發布](https://extensionworkshop.com/documentation/publish/signing-and-distribution-overview/)
