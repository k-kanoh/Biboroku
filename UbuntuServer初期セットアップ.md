# Ubuntu-Server初期セットアップ

## 準備
#### root
```bash
# システムパッケージの更新
apt update && apt upgrade -y

# 必要なパッケージのインストール
apt install -y vim ansible git curl
```

#### aaaaa
```bash
# playbooksをClone
git clone https://github.com/k-kanoh/playbooks.git
cd playbooks/

# kkanoユーザー作成
ansible-playbook init-playbook.yml
```

## システムセットアップ
##### scpで.ssh/config及びgithub転送
#### kkano
```bash
# 秘密鍵のパーミッション変更(怒られる)
chmod 600 .ssh/github

# playbooksをClone
git clone git@github.com:k-kanoh/playbooks.git
cd playbooks/ansible/

# システムセットアップ
ansible-playbook site.yml
ansible-playbook ntfs931.yml
ansible-playbook node.yml

# ユーザーをグループに追加
sudo usermod -aG docker $USER
sudo usermod -aG audio $USER
```

## WiFi設定
#### root
```bash
# ネットワークインターフェース一覧の確認
ip link show

# ネットワーク設定ファイルの編集
vi /etc/netplan/50-cloud-init.yaml
```

##### 50-cloud-init.yamlの内容
```yaml
network:
  version: 2
  ethernets:
    eno1:
      addresses:
      - "192.168.0.111/24"
      nameservers:
        addresses:
        - 192.168.0.1
        search: []
      routes:
      - to: "default"
        via: "192.168.0.1"
  wifis:
    wlp0s20f3:
      addresses:
        - "192.168.0.99/24"
      nameservers:
        addresses:
          - 192.168.0.1
      routes:
        - to: default
          via: 192.168.0.1
          metric: 100
      access-points:
        "(SSID)":
          password: "(暗号キー)"
```
- 有線(eno1): 192.168.0.111
- 無線(wlp0s20f3): 192.168.0.99
- WiFiルートのmetric値を100に設定し、有線を優先
- パスワードに `'` や `"` が含まれる場合は適切にエスケープするか、パスワードを変更推奨
```bash
# ネットワーク設定を適用(別端末からping -tして問題なければ適用)
netplan try

# または即座に適用
netplan apply
```

## 完了
#### root
```bash
# 初期セットアップに使用したユーザー(aaaaa)を削除
userdel -r aaaaa

# 電源オフ(PC移動)
poweroff
```

---
### WiFi接続のトラブルシューティング
```bash
# 無線デバイスの状態確認
ip addr show wlp0s20f3
ip link show wlp0s20f3

# 経路確認
ip route

# netplanの詳細ログ付き適用
netplan --debug apply

# netplanが生成した設定ファイルの確認
cat /run/systemd/network/10-netplan-wlp0s20f3.network
cat /run/netplan/wpa-wlp0s20f3.conf

# ネットワーク関連ログの確認
journalctl -u systemd-networkd -n 50
journalctl -u netplan-wpa-wlp0s20f3.service -n 50

# サービスの状態確認
systemctl status netplan-wpa-wlp0s20f3.service

# サービスの再起動
systemctl restart netplan-wpa-wlp0s20f3.service

# 無線デバイスの手動UP
ip link set wlp0s20f3 up

# 接続テスト
ping -c 3 8.8.8.8
ping -c 3 192.168.0.1
```

### 音声テスト
```bash
# オーディオデバイスの確認
aplay -l

# ボリューム設定
alsamixer

# 音声テスト
speaker-test -t wav -c 2

# 音声テスト(デバイス指定)
speaker-test -t wav -c 2 -D hw:0,0
```
### apt関連
##### apt remove
- パッケージのバイナリ（実行ファイル）のみを削除
- **設定ファイルは残る**（`/etc/` 以下など）
- 再インストール時に古い設定がそのまま使われる

##### apt purge
- パッケージのバイナリ**と設定ファイルの両方**を削除
- クリーンな状態に戻せる
- 再インストール時にデフォルト設定が生成される

```bash
# これだと上書きされた nginx.conf が残ってしまう
apt remove -y nginx

# これなら /etc/nginx/ 配下も削除されてクリーン
apt purge -y nginx nginx-common
```

##### パッケージ状態の確認
```bash
# インストール状態の確認
dpkg -l | grep nginx
```

出力の最初の文字が:
- `ii` = インストール済み
- `rc` = 削除済みだが設定ファイルが残っている（remove済み）
- 何も表示されない = 完全に削除されている（purge済み）

##### 削除のドライラン
```bash
# 削除予定のパッケージを確認（実際には削除しない）
apt purge nginx --dry-run
```

### nginxのトラブルシューティング
```bash
# 設定ファイルの構文チェック
nginx -t
```
