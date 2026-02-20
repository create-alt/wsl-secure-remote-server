# 発展的な環境構築を伴う遠隔操作サーバ構築手順書

## 1. 目的と完成条件の定義
本手順書は、ノートPC上のWSL(Ubuntu)をサーバとして構築し、同一ネットワーク上のスマートフォンから安全に遠隔操作を行う環境を構築することを目的とする。

**完成条件：**
* スマートフォンから公開鍵認証を用いてのみWSLへSSH接続できること（パスワード認証の完全無効化）。
* 複数回の認証失敗に対して、fail2banが適切にIPをブロックすること。
* 接続後、コマンド一つでWSL上のカメラ制御プログラム（OpenCV）が起動し、画像が保存されること。

## 2. 前提条件
* **対象OS:** Ubuntu 22.04 LTS (Windows 11 WSL2 上で稼働)
* **実行環境:** ローカルVM (WSL2)
* **利用ツール・パッケージ管理:** `apt`, `ufw`, `fail2ban`, `python3`, `pip`, `usbipd-win` (Windows側)
* **ネットワーク条件:** ローカルネットワーク（Wi-Fi）。Windowsホスト経由のポートフォワーディングを利用。

## 3. 全体構成図とポート設計


* **通信経路:** スマートフォン(SSH Client) → [Wi-Fi] → Windows 11 (Port: 2222) → [Port Proxy] → WSL2 Ubuntu (Port: 22)
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
# connectaddress は手順1で確認したWSLのIPを指定すること
netsh interface portproxy add v4tov4 listenport=2222 listenaddress=0.0.0.0 connectport=22 connectaddress=172.18.0.5

# Windowsファイアウォールで2222番ポートの外部からの受信を許可
New-NetFirewallRule -DisplayName "WSL SSH" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 2222
```

**3. USBカメラの接続準備 (usbipd)**
WSLからカメラを利用するために、PowerShell(管理者)でデバイスをアタッチする。
```powershell
# デバイス一覧を表示し、カメラのBUSIDを確認
usbipd list

# カメラをWSLに接続 (例: BUSIDが 2-1 の場合)
usbipd bind --busid 2-1
usbipd attach --wsl --busid 2-1
```

## 5. 構築手順 (WSL/Ubuntu側の設定)

### 5.1 パッケージの更新とSSH導入
```bash
# パッケージリストの更新とアップグレード
sudo apt update && sudo apt upgrade -y

# SSHサーバのインストール
sudo apt install openssh-server -y

# SSHサービスの起動と自動起動設定
sudo service ssh start
```

### 5.2 発展要素：ユーザー管理強化 (SSH公開鍵認証)
パスワード認証よりも安全な公開鍵認証を設定する。

```bash
# 1. SSH鍵ペアの生成 (ed25519アルゴリズムを使用)
# パスフレーズは任意だが、手順書上は空("")として進行する
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""

# 2. 公開鍵を認可リストに登録
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys

# 3. 権限の厳格化（所有者のみ読み書き可能に設定。これが甘いとSSH接続できない）
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```
**重要:** 生成された秘密鍵 (`~/.ssh/id_ed25519`) を `cat` コマンド等で表示し、内容をコピーしてスマートフォンのSSHアプリ等に安全に転送・登録しておくこと。

### 5.3 発展要素：ハードニング (UFW と fail2ban)
外部公開に備え、ファイアウォールと侵入検知システムを導入する。

```bash
# UFWのインストール
sudo apt install ufw -y

# SSHポート(22)のみ許可
sudo ufw allow 22/tcp

# UFWの有効化 (yを押して承認)
sudo ufw enable
```

続いて、総当たり攻撃対策の `fail2ban` を設定する。
```bash
# fail2banのインストール
sudo apt install fail2ban -y

# 設定ファイルの作成
sudo nano /etc/fail2ban/jail.local
```

**【設定ファイル完全版】 `/etc/fail2ban/jail.local`**
(保存パス: `/etc/fail2ban/jail.local`, 所有者: root, 権限: 644)
```ini
[DEFAULT]
# 信頼するIP (ローカルホスト)
ignoreip = 127.0.0.1/8 ::1
# ブロック時間 (1時間)
bantime  = 3600
# 監視期間 (10分)
findtime  = 600
# 最大失敗回数
maxretry = 3

[sshd]
enabled = true
port    = ssh
filter  = sshd
logpath = /var/log/auth.log
maxretry = 3
```

```bash
# fail2banの再起動
sudo service fail2ban restart
```

### 5.4 SSH設定のハードニング (パスワード認証の無効化)
鍵認証の準備が整ったら、パスワードによるログインを禁止する。

```bash
sudo nano /etc/ssh/sshd_config
```

**【設定ファイル完全版】 `/etc/ssh/sshd_config`**
(保存パス: `/etc/ssh/sshd_config`, 所有者: root, 権限: 644)
※主要な設定箇所のみ抜粋。既存の行を書き換えるか追記する。
```text
Include /etc/ssh/sshd_config.d/*.conf

# ポート設定
Port 22
AddressFamily any
ListenAddress 0.0.0.0

# 認証関連のセキュリティ設定
Protocol 2
LoginGraceTime 2m
PermitRootLogin no
StrictModes yes
MaxAuthTries 3
MaxSessions 10

# 公開鍵認証の有効化
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys

# パスワード認証の無効化（重要）
PasswordAuthentication no
PermitEmptyPasswords no
ChallengeResponseAuthentication no
UsePAM yes

X11Forwarding yes
PrintMotd no
AcceptEnv LANG LC_*
Subsystem sftp /usr/lib/openssh/sftp-server
```

```bash
# 設定反映
sudo service ssh restart
```

### 5.5 遠隔実行用プログラム (OpenCVカメラ制御) の配置
スマートフォンから実行するPythonスクリプトを作成する。

```bash
# 必要なライブラリのインストール
sudo apt install python3-opencv python3-pip -y

# スクリプトの作成
nano ~/capture.py
```

**【ソースコード完全版】 `~/capture.py`**
(保存パス: `/home/ユーザー名/capture.py`, 権限: 644)
```python
import cv2
import datetime
import os
import sys

def main():
    print(f"[{datetime.datetime.now()}] カメラ処理を開始します...")
    
    # カメラデバイスID (通常は0。usbipdで接続されていること)
    cap = cv2.VideoCapture(0)
    
    if not cap.isOpened():
        print("エラー: カメラを開けませんでした。")
        print("ヒント: Windows側で 'usbipd attach' が実行されているか確認してください。")
        sys.exit(1)

    # フレームの読み込み
    ret, frame = cap.read()
    
    if ret:
        # タイムスタンプ付きファイル名
        filename = f"remote_capture_{datetime.datetime.now().strftime('%Y%m%d_%H%M%S')}.jpg"
        filepath = os.path.join(os.path.expanduser("~"), filename)
        
        # 画像保存
        cv2.imwrite(filepath, frame)
        print(f"成功: 画像を保存しました -> {filepath}")
    else:
        print("エラー: 画像フレームの取得に失敗しました。")

    cap.release()

if __name__ == "__main__":
    main()
```

## 6. 動作確認と検証

**1. 疎通確認 (スマホからの接続)**
* スマートフォンのSSHアプリに設定したプロファイル（IP: WindowsのIP, Port: 2222, Key: 秘密鍵）で接続する。
* パスワードを入力することなくログイン成功することを確認する。
> [画像挿入：スマートフォンでの接続成功画面]

**2. プログラム実行確認**
* SSH接続したターミナルで以下のコマンドを実行する。
```bash
python3 ~/capture.py
```
* 「成功: 画像を保存しました」と表示され、`ls -l` で画像ファイルが確認できること。
> [画像挿入：コマンド実行結果のスクリーンショット]

**3. セキュリティ動作確認 (fail2ban)**
* 別の端末や設定でわざと間違った認証を数回行い、接続が拒否されることを確認する。
* サーバ側で `sudo fail2ban-client status sshd` を実行し、BanされたIPがリストにあることを確認する。
> [画像挿入：fail2banによってBanされたIPリストの表示]

## 7. トラブルシューティング

| 現象 | 原因 | 対処 |
| :--- | :--- | :--- |
| **スマホから接続できない (Timeout)** | WindowsファイアウォールまたはIP設定 | PowerShellでファイアウォールルールを確認。`ipconfig`でIPが変わっていないか確認。 |
| **Permission denied (publickey)** | 鍵の不一致または権限設定 | 秘密鍵が正しいか確認。サーバ側で `chmod 600 ~/.ssh/authorized_keys` を再実行。 |
| **カメラエラー (OpenCV)** | WSLにUSBデバイスが来ていない | Windows管理者PowerShellで `usbipd attach` を再度実行する。WSLを再起動すると解除されるため注意。 |

## 8. セキュリティ配慮
* **権限最小化:** SSHの秘密鍵認証を必須とし、パスワード認証を廃止することで、辞書攻撃のリスクを排除した。
* **自己防衛:** fail2banを導入し、不正アクセス試行を自動的に遮断する仕組みを構築した。
* **秘密情報の管理:** 秘密鍵は安全な経路で転送し、設定ファイル内にはハードコードされたパスワードを含めない構成とした。

## 9. 参考資料
* [OpenSSH Server - Ubuntu Documentation](https://ubuntu.com/server/docs/service-openssh)
* [Fail2ban - Community Help Wiki](https://help.ubuntu.com/community/Fail2ban)
* [USB デバイスを接続する - Windows Subsystem for Linux](https://learn.microsoft.com/ja-jp/windows/wsl/connect-usb)
