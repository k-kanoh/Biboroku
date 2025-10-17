# Ubuntu-Server初期セットアップ

## 準備
```bash
# システムパッケージの更新
apt update && apt upgrade -y

# 必要なパッケージのインストール
apt install -y vim ansible git curl

# 初期セットアップに使用したユーザー(aaaaa)を削除
userdel -r aaaaa
```

## WiFi設定
```bash
# ネットワークインターフェース一覧の確認
ip link show

# ネットワーク設定ファイルの編集
vi /etc/netplan/50-cloud-init.yaml
```

**50-cloud-init.yamlに追加する内容:**
```yaml
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
- 有線(eno1): 192.168.0.98
- 無線(wlp0s20f3): 192.168.0.99
- WiFiルートのmetric値を100に設定し、有線を優先
- パスワードに `'` や `"` が含まれる場合は適切にエスケープするか、パスワードを変更推奨
```bash
# ネットワーク設定を適用(別端末からping -tして問題なければ適用)
netplan try

# または即座に適用
netplan apply
```

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
