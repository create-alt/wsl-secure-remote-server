# 最終課題：発展的な環境構築を伴う遠隔操作サーバ構築手順書

## 1. 目的と完成条件の定義
本手順書は、ノートPC上のWSL(Ubuntu)をサーバとして構築し、同一ネットワーク上のスマートフォンから安全に遠隔操作を行う環境を構築することを目的とする。

**完成条件：**
* スマートフォン(iOS)から公開鍵認証を用いてのみWSLへSSH接続できること（パスワード認証の完全無効化）。
* 複数回の認証失敗に対して、fail2banが適切にIPをブロックすること。
* 接続後、コマンド一つでWSL上のカメラ制御プログラム（OpenCV）が起動し、画像が保存されること。

## 2. 前提条件
* **対象OS:** Ubuntu 22.04 LTS (Windows 11 WSL2 上で稼働)
* **実行環境:** ローカルVM (WSL2)
* **クライアント環境:** iOS 18.5 (アプリ: Termius を使用)
* **利用ツール・パッケージ:** `apt`, `ufw`, `fail2ban`, `python3`, `pip`, `usbipd-win` (Windows側)
* **ネットワーク条件:** ローカルネットワーク（Wi-Fi）。Windowsホスト経由のポートフォワーディングを利用。

## 3. 全体構成図とポート設計


* **通信経路:** iPhone(Termius) → [Wi-Fi] → Windows 11 (Port: 2222) → [Port Proxy] → WSL2 Ubuntu (Port: 22)
* **使用ポート:**
  * 外部待受ポート: TCP 2222 (Windows側で開放)
  * 内部サーバポート: TCP 22 (WSLのSSHデーモン)

## 4. 事前準備 (Windows側の設定)
WSL2はWindowsとは異なる仮想IPを持つため、Windows側で通信を中継する設定を行う。

**1. IPアドレスの確認**
Windowsのコマンドプロンプト、およびWSLのターミナルで以下を実行しIPを控える。
* WindowsのIP (`ipconfig` IPv4アドレス): 例 `192.168.1.10`
* WSLのIP (`ip a` eth0のinetアドレス): 例 `172.18.0.5`

**2. ポートフォワーディングとファイアウォール許可**
Windows PowerShellを**管理者権限**で実行し、以下のコマンドを入力する。

```powershell
# Windowsの2222番ポートへのアクセスをWSLの22番ポートへ転送
netsh interface portproxy add v4tov4 listenport=2222 listenaddress=0.0.0.0 connectport=22 connectaddress=172.18.0.5

# Windowsファイアウォールで2222番ポートの外部からの受信を許可
New-NetFirewallRule -DisplayName "WSL SSH" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 2222
```

**3. USBカメラの接続準備 (usbipd)**
PowerShell(管理者)でカメラデバイスをWSLにアタッチする。
```powershell
usbipd list
# カメラをWSLに接続 (例: BUSIDが 2-1 の場合)
usbipd bind --busid 2-1
usbipd attach --wsl --busid 2-1
```

## 5. 構築手順 (WSL/Ubuntu側の設定)

### 5.1 パッケージの更新とSSH導入
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install openssh-server -y
sudo service ssh start
```

### 5.2 発展要素：ユーザー管理強化 (SSH公開鍵認証)
```bash
# 1. SSH鍵ペアの生成 (パスフレーズは空とする)
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""

# 2. 公開鍵を認可リストに登録
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys

# 3. 権限の厳格化
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

### 5.3 発展要素：ハードニング (UFW と fail2ban)
```bash
sudo apt install ufw -y
sudo ufw allow 22/tcp
sudo ufw --force enable
```

```bash
sudo apt install fail2ban -y
sudo nano /etc/fail2ban/jail.local
```

**【設定ファイル完全版】 `/etc/fail2ban/jail.local`** (権限: 644)
```ini
[DEFAULT]
ignoreip = 127.0.0.1/8 ::1
bantime  = 3600
findtime  = 600
maxretry = 3

[sshd]
enabled = true
port    = ssh
filter  = sshd
logpath = /var/log/auth.log
maxretry = 3
```
```bash
sudo service fail2ban restart
```

### 5.4 SSH設定のハードニング (パスワード認証の無効化)
```bash
sudo nano /etc/ssh/sshd_config
```

**【設定ファイル完全版】 `/etc/ssh/sshd_config`** (権限: 644)
※主要な設定箇所のみ抜粋。
```text
Include /etc/ssh/sshd_config.d/*.conf
Port 22
AddressFamily any
ListenAddress 0.0.0.0
Protocol 2
LoginGraceTime 2m
PermitRootLogin no
StrictModes yes
MaxAuthTries 3
MaxSessions 10
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
PasswordAuthentication no  # パスワード認証を無効化
PermitEmptyPasswords no
ChallengeResponseAuthentication no
UsePAM yes
X11Forwarding yes
PrintMotd no
AcceptEnv LANG LC_*
Subsystem sftp /usr/lib/openssh/sftp-server
```
```bash
sudo service ssh restart
```

### 5.5 遠隔実行用プログラム (OpenCVカメラ制御) の配置
```bash
sudo apt install python3-opencv python3-pip -y
nano ~/capture.py
```

**【ソースコード完全版】 `~/capture.py`** (権限: 644)
```python
import cv2
import datetime
import os
import sys

def main():
    print(f"[{datetime.datetime.now()}] カメラ処理を開始します...")
    cap = cv2.VideoCapture(0)
    
    if not cap.isOpened():
        print("エラー: カメラを開けませんでした。")
        sys.exit(1)

    ret, frame = cap.read()
    if ret:
        filename = f"remote_capture_{datetime.datetime.now().strftime('%Y%m%d_%H%M%S')}.jpg"
        filepath = os.path.join(os.path.expanduser("~"), filename)
        cv2.imwrite(filepath, frame)
        print(f"成功: 画像を保存しました -> {filepath}")
    else:
        print("エラー: 画像フレームの取得に失敗しました。")

    cap.release()

if __name__ == "__main__":
    main()
```

### 5.6 スマートフォン (iOS 18.5) 側の接続準備
iPhoneから安全に接続するための鍵転送とアプリ設定を行う。

**1. 秘密鍵のiOSへの転送**
WSL上で以下のコマンドを実行し、Windowsのエクスプローラーを起動する。
```bash
explorer.exe ~/.ssh
```
開いたフォルダ内にある `id_ed25519`（拡張子のない秘密鍵ファイル）をコピーし、iCloud Drive等を経由してiPhoneの「ファイル」アプリに保存する。
*(※セキュリティ配慮：転送完了後、iCloud上の中間ファイルは削除すること)*

**2. Termius (iOSアプリ) の設定**
1. App Storeから **Termius** をインストールして起動する。
2. **鍵のインポート:**
   * 左上のメニュー（三本線）から `Keychain`（または `Keys`）を開く。
   * 右上の `+` ボタンをタップし、`Import Key` を選択。
   * iPhoneの「ファイル」アプリが起動するので、先ほど保存した `id_ed25519` を選択してインポートする。
3. **接続先 (Host) の作成:**
   * `Hosts` 画面に戻り、右上の `+` ボタンから `New Host` を選択。
   * 以下のように入力する。
     * **Alias:** 任意の名前 (例: WSL Camera Server)
     * **Hostname or IP Address:** WindowsのIPアドレス (例: `192.168.1.10`)
     * **Port:** `2222`
     * **Username:** WSLのユーザー名
     * **Key:** 先ほどインポートした `id_ed25519` を選択する。
   * 右上の `Save` をタップして保存。

> [画像挿入：TermiusのHost設定画面（IP、Port、Username、Keyが設定されている状態）]

## 6. 動作確認と検証

**1. 疎通確認と公開鍵認証のテスト (iOS側)**
* Termiusの `Hosts` 一覧から作成したホストをタップする。
* 初回接続時の警告 (Fingerprintの確認) が出たら `Continue` をタップする。
* パスワードを入力することなく、WSLのターミナル画面が表示されることを確認する。
> [画像挿入：iPhone(Termius)上でWSLにログイン成功したターミナル画面のスクリーンショット]

**2. プログラムの遠隔実行 (iOS側)**
* iPhoneのTermiusの画面で、以下のコマンドをソフトウェアキーボードで入力して実行する。
```bash
python3 ~/capture.py
```
* 「成功: 画像を保存しました」と表示された後、`ls -l *.jpg` を実行し、画像ファイルが生成されていることを確認する。
> [画像挿入：iPhone上でコマンドを実行し、成功メッセージと画像ファイルが表示されたスクリーンショット]

**3. セキュリティ動作確認 (fail2ban) (サーバ側)**
* 意図的に鍵の設定を外すなどしてパスワード認証を試み、数回ログインを失敗させる。
* サーバ（WSL）側で `sudo fail2ban-client status sshd` を実行し、BanされたIP（iPhoneのIPアドレス）がリストに追加されていることを確認する。
> [画像挿入：fail2banによってBanされたIPリストの表示]

## 7. トラブルシューティング
* **iPhoneからConnection RefusedやTimeoutになる**
  * Windowsファイアウォールで2222番が許可されていない、またはiPhoneとPCが同じWi-Fiルーターに接続されていない。
* **Termiusで "Permission denied (publickey)" と出る**
  * サーバ側で `chmod 600 ~/.ssh/authorized_keys` の設定が漏れている、またはTermiusで指定した鍵が間違っている。

## 8. セキュリティ配慮
* **パスワード認証の完全無効化:** ブルートフォース攻撃を防ぐため、`sshd_config` で `PasswordAuthentication no` とし、鍵を持つ端末からしかアクセスできない設計とした。
* **不要ポートの閉鎖:** UFWを用いてSSH（22番）以外の外部からの不要な通信を全て遮断している。

## 9. 参考資料
* [OpenSSH Server - Ubuntu Documentation](https://ubuntu.com/server/docs/service-openssh)
* [Fail2ban - Community Help Wiki](https://help.ubuntu.com/community/Fail2ban)
* [USB デバイスを接続する - Windows Subsystem for Linux](https://learn.microsoft.com/ja-jp/windows/wsl/connect-usb)
