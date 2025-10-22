# crontab vs systemd 完全ガイド

## 概要比較

| 項目 | crontab | systemd |
|------|---------|---------|
| 世代 | レガシー（1970年代～） | モダン（2010年代～） |
| 設定の簡単さ | ✅ `crontab -e` のみ | ❌ ファイル作成 + enable + start |
| 初回設定 | 一般ユーザーのみで完結 | ユーザーsystemdは `enable-linger` が必要（rootまたは放置） |
| 再起動制御 | シェルスクリプトでTERMキャッチなど実装が必要 | `Restart=`, `RuntimeMaxSec=` など豊富なオプション |
| ログ | stderr/stdout をリダイレクト | journal に統合（`journalctl` で確認） |
| 必要な知識 | シェルのリダイレクト（`>>`, `2>&1`）は必要だが簡単 | 設定ファイルのファーストインプレッションが最悪 |
| 編集方法 | `crontab -e` | `~/.config/systemd/user/` にファイル配置 |
| 有効化 | 保存すれば即反映 | `systemctl --user enable && start` が必要 |

---

## systemd の設計思想: なぜ分かりにくいのか

### cron との比較

**cron: すべてが1行に集約**

```cron
# スケジュール + 実行内容 + ログ設定が1行
0 * * * * /path/to/script.sh >> /path/to/log 2>&1
```

- ✅ シンプル、直感的
- ❌ 複雑な制御は難しい
- ❌ 責務が混在（いつ・何を・どこに、が分離されていない）

---

**systemd: 責務の分離（Unix哲学）**

```
Timer（いつ実行するか）
  ├─ OnCalendar=hourly      → スケジューリングの責務
  ├─ OnBootSec=2min         → 起動トリガーの責務
  └─ Persistent=true/false  → リトライポリシーの責務

Service（何を実行するか）
  ├─ ExecStart=...          → 実行内容の責務
  ├─ WorkingDirectory=...   → 実行環境の責務
  ├─ Restart=...            → プロセス管理の責務
  └─ RuntimeMaxSec=...      → ライフサイクルの責務
```

- ✅ 高機能、柔軟
- ✅ 責務が明確に分離
- ❌ **初見では分かりにくい**
- ❌ ファイルが複数必要

---

### なぜ分離しているのか

**Unix哲学: "Do One Thing and Do It Well"（1つのことをうまくやる）**

- **Timer**: スケジューリング専門
- **Service**: プロセス実行専門
- **Journal**: ログ管理専門

この分離により：

1. **再利用性**: 同じ Service を複数の Timer から呼べる

```
eqmonitor.service（実行内容は1つ）
  ↑
  ├─ hourly.timer（毎時実行）
  ├─ boot.timer（起動時実行）
  └─ manual.timer（手動トリガー）
```

2. **変更の影響範囲が限定される**
   - スケジュール変更 → Timer だけ編集
   - 実行コマンド変更 → Service だけ編集

3. **テストしやすい**

```bash
# Timer を無視して Service を直接テスト
systemctl --user start eqmonitor.service

# 動作確認後に Timer を有効化
systemctl --user enable --now eqmonitor.timer
```

---

### systemd の他の分離例

#### Socket と Service

```
nginx.socket（ポート80を待ち受ける）
  ↓ 接続があったら
nginx.service（nginxプロセスを起動）
```

- **Socket**: いつ起動するか（ポート監視）
- **Service**: 何を起動するか（nginx実行）

#### Path と Service

```
file-watcher.path（ファイル変更を監視）
  ↓ ファイルが変更されたら
file-processor.service（処理を実行）
```

- **Path**: いつ起動するか（ファイル監視）
- **Service**: 何を起動するか（処理実行）

---

### 初見で分かりにくい理由まとめ

1. **責務が分離されすぎている**
   - cron: 1行で完結
   - systemd: 2ファイル + 複数コマンド

2. **設定ファイルの記述量が多い**
   - INI形式で冗長
   - セクション、キーが多数

3. **enable の概念**
   - cron: 書けば即有効
   - systemd: enable + start の2段階

4. **Timer と Service の関係**
   - 名前を同じにすると自動的に紐付く（暗黙的）
   - Service は enable 不要（Timer が呼ぶから）

**しかし、この複雑さが高機能・柔軟性を実現している**

---

## rootのsystemdとuserのsystemd

### システムサービス（root）

**配置場所**: `/etc/systemd/system/`

```bash
# 作成（管理者として）
sudo nano /etc/systemd/system/myapp.service

# 管理
sudo systemctl daemon-reload
sudo systemctl enable myapp.service
sudo systemctl start myapp.service
```

**特徴**:
- root権限で管理
- すべてのユーザーから利用可能
- システム起動時に自動起動
- 本番サービス（Webアプリ、データベースなど）で使用
- `User=kkano` で実行ユーザーを指定可能

```ini
[Service]
User=kkano
Group=kkano
WorkingDirectory=/home/kkano/myapp
ExecStart=/home/kkano/myapp/run.sh
```

---

### ユーザーサービス

**配置場所**: `~/.config/systemd/user/`

```bash
# 作成（一般ユーザーとして）
mkdir -p ~/.config/systemd/user
nano ~/.config/systemd/user/myapp.service

# 管理
systemctl --user daemon-reload
systemctl --user enable myapp.service
systemctl --user start myapp.service
```

**特徴**:
- 一般ユーザー権限で管理
- そのユーザー専用
- **デフォルトではログイン中のみ動作**
- ログインなしで動作させるには `sudo loginctl enable-linger kkano` が必要
- 個人用スクリプト、開発環境で使用

---

## systemdのServiceとTimerについて

### 役割分担（責務の分離）

```
Timer (時計役) ← "いつ"実行するか
  ↓ 定時になったら
Service (実行本体) ← "何を"実行するか
  ↓ プロセス起動
実際のプログラム
```

- **Timer**: いつ実行するか（スケジューリングの責務）
- **Service**: 何を実行するか（実行内容の責務）
- **名前を同じにする**ことで自動的に紐付く
  - `eqmonitor.timer` → `eqmonitor.service`

### なぜ分離するのか

**cron の場合（責務が混在）:**

```cron
# スケジュール・実行内容・ログ設定がすべて1行
0 * * * * /path/to/script.sh >> /path/to/log 2>&1
```

**systemd の場合（責務が分離）:**

```ini
# eqmonitor.timer: スケジュールのみ
[Timer]
OnCalendar=hourly

# eqmonitor.service: 実行内容のみ
[Service]
ExecStart=/path/to/script.sh
StandardOutput=journal
```

**利点:**
- スケジュール変更時に実行内容に影響しない
- 同じ Service を複数の Timer から呼べる
- テストしやすい（Service を単独で起動可能）

---

## 設定ファイルの項目解説

### Timer ファイル (`eqmonitor.timer`)

```ini
[Unit]
Description=Start P2PQuake Receiver every hour
```
- **Description**: タイマーの説明文（`systemctl status` で表示）

```ini
[Timer]
OnCalendar=hourly
```
- **OnCalendar**: 実行スケジュール（"いつ"の責務）
  - `hourly` = 毎時0分（`*-*-* *:00:00`）
  - `daily` = 毎日0時
  - `weekly` = 毎週月曜0時
  - `*:00/15:00` = 15分ごと
  - `Mon,Fri 10:00` = 月曜と金曜の10時

```ini
OnBootSec=2min
```
- **OnBootSec**: システム起動後の実行（起動トリガーの責務）
  - `2min` = 起動2分後に実行
  - `OnCalendar` と併用可能（両方のトリガーが有効）

```ini
Persistent=true
```
- **Persistent**: システム停止中に実行されるはずだったタスクを起動後に実行するか（リトライポリシーの責務）
  - `true`: 停止中のタスクを起動後すぐ実行
  - `false`（デフォルト）: 次の予定時刻まで待つ
  - 毎時実行なら `false` でOK（1回スキップしても次がすぐ来る）

```ini
[Install]
WantedBy=timers.target
```
- **WantedBy**: `systemctl enable` 時にどこに登録するか
- `timers.target` = すべてのタイマーを管理する特殊なターゲット
- これがあるから `enable` が意味を持つ

---

### Service ファイル (`eqmonitor.service`)

```ini
[Unit]
Description=P2PQuake WebSocket Receiver
After=network-online.target
```
- **Description**: サービスの説明文
- **After**: 指定したターゲットの後に起動（依存関係の責務）
  - `network-online.target` = ネットワーク接続後

```ini
[Service]
Type=simple
```
- **Type**: サービスの起動タイプ（プロセス管理の責務）
  - `simple`: プロセスがフォアグラウンドで実行（最も一般的）
  - `forking`: デーモン化（バックグラウンド実行）
  - `oneshot`: 実行して終了（スクリプト実行など）

```ini
WorkingDirectory=/home/kkano/eqmonitor
```
- **WorkingDirectory**: 実行時のカレントディレクトリ（実行環境の責務）
- `cd /home/kkano/eqmonitor` してから実行するのと同じ

```ini
ExecStart=/home/kkano/eqmonitor/.venv/bin/python -u mon.py --env prod
```
- **ExecStart**: 実行するコマンド（実行内容の責務）
  - 絶対パス推奨
  - **`-u`**: Python のバッファリング無効化（ログをリアルタイム出力）
  - `WorkingDirectory` があるので `mon.py` は相対パスでもOK

```ini
Restart=no
```
- **Restart**: プロセス終了時の再起動ポリシー（ライフサイクルの責務）
  - `no`: 再起動しない
  - `always`: 常に再起動
  - `on-failure`: 異常終了時のみ再起動
  - `on-abort`: シグナルで終了時のみ再起動

```ini
RuntimeMaxSec=3600
```
- **RuntimeMaxSec**: 最大実行時間（秒）（ライフサイクルの責務）
- 指定時間経過後に自動的にSIGTERMを送信
- タイマーで定期起動する場合、強制終了に便利

```ini
TimeoutStopSec=10
```
- **TimeoutStopSec**: 停止時のタイムアウト（秒）（ライフサイクルの責務）
- SIGTERM送信後、この時間待ってもプロセスが終了しなければSIGKILLを送る

```ini
StandardOutput=journal
StandardError=journal
```
- **StandardOutput/StandardError**: 標準出力・エラー出力の行き先（ログ管理の責務）
  - `journal`: systemd のログシステム（journalctl で確認）
  - `null`: 破棄
  - `file:/path/to/log`: ファイル出力
  - `append:/path/to/log`: ファイル追記

```ini
Environment="PYTHONUNBUFFERED=1"
```
- **Environment**: 環境変数の設定（実行環境の責務）
- `PYTHONUNBUFFERED=1` は `python -u` と同じ効果

```ini
[Install]
WantedBy=default.target
```
- **WantedBy**: `systemctl enable` 時にどこに登録するか
- `default.target` = ユーザーサービスのデフォルトターゲット
- `multi-user.target` = システムサービス用（root）
- **Timer から呼ばれるだけの Service には不要**（だから今回は書いていない）

---

## enable の意味と必要性

### enable とは

**「システム起動時に自動的にこのユニットを起動する」** という設定です。

- `systemctl --user enable eqmonitor.timer` → 再起動後も timer が自動起動
- `systemctl --user enable eqmonitor.service` → 再起動後も service が自動起動

### enable の仕組み

`[Install]` セクションの `WantedBy=` を見て、シンボリックリンクを作成します：

```ini
[Install]
WantedBy=default.target
```

これがあると：

```bash
systemctl --user enable eqmonitor.timer
# → ~/.config/systemd/user/default.target.wants/eqmonitor.timer へのシンボリックリンクを作成
```

システム起動時、`default.target` が起動されると、その `.wants/` ディレクトリ内のすべてのユニットも起動されます。

---

### なぜ Service には `[Install]` がないのか

**Timer から呼び出されるだけの Service だから**

```
システム起動
  ↓
default.target 起動
  ↓
eqmonitor.timer 起動（enable されているから）
  ↓ 毎時0分
eqmonitor.service 起動（Timer から呼ばれる）
```

- `eqmonitor.timer` は `enable` が必要（システム起動時に自動起動させたい）
- `eqmonitor.service` は `enable` 不要（Timer が勝手に起動してくれる）

だから `eqmonitor.service` には `[Install]` セクションが不要です。

---

### enable が必要なユニット

| ユニット | enable の必要性 | 理由 |
|---------|---------------|------|
| `eqmonitor.timer` | ✅ **必要** | システム起動時に自動起動させたい |
| `eqmonitor.service` | ❌ 不要 | Timer から呼ばれるだけ |

**Service を enable しようとすると...**

```bash
systemctl --user enable eqmonitor.service
# → エラーメッセージ: "[Install] セクションがないので enable できません"
```

これは**正常な動作**です。Timer から呼ばれる Service には `[Install]` は不要。

---

### 正しい操作

```bash
# Timer だけ enable & start
systemctl --user enable --now eqmonitor.timer

# Service は enable しない（Timer が勝手に起動してくれる）
```

---

## 基本的なコマンド

### サービス管理

```bash
# 設定ファイル再読み込み（編集後は必須）
systemctl --user daemon-reload

# 自動起動を有効化
systemctl --user enable eqmonitor.timer

# タイマー開始
systemctl --user start eqmonitor.timer

# 有効化 + 開始（同時実行）
systemctl --user enable --now eqmonitor.timer

# タイマー停止
systemctl --user stop eqmonitor.timer

# 自動起動を無効化
systemctl --user disable eqmonitor.timer

# サービス（実行中のプロセス）を停止
systemctl --user stop eqmonitor.service
```

---

### 状態確認

```bash
# タイマー一覧（次回実行予定時刻も表示）
systemctl --user list-timers

# 全タイマー一覧（無効なものも含む）
systemctl --user list-timers --all

# タイマーの状態確認
systemctl --user status eqmonitor.timer

# サービスの状態確認
systemctl --user status eqmonitor.service

# 実行中のサービス一覧
systemctl --user list-units --type=service --state=running

# 依存関係を表示
systemctl --user list-dependencies eqmonitor.timer
```

---

### ログ確認

```bash
# サービスのログを表示
journalctl --user -u eqmonitor.service

# 最新10行のみ表示
journalctl --user -u eqmonitor.service -n 10

# リアルタイムでログ表示（tail -f 風）
journalctl --user -u eqmonitor.service -f

# 今日以降のログのみ
journalctl --user -u eqmonitor.service --since today

# 期間指定
journalctl --user -u eqmonitor.service --since "2025-10-21 20:00" --until "2025-10-21 23:00"

# タイマーのログ
journalctl --user -u eqmonitor.timer

# ログ使用量確認
journalctl --user --disk-usage

# ログを期間で削減
journalctl --user --vacuum-time=7d

# ログをサイズで削減
journalctl --user --vacuum-size=10M
```

---

### rootサービスの場合

`--user` を外して `sudo` を付けるだけ：

```bash
sudo systemctl daemon-reload
sudo systemctl enable myapp.service
sudo systemctl start myapp.service
sudo systemctl status myapp.service
sudo journalctl -u myapp.service -f
```

---

## プロセスの停止方法（重要）

### Service と Timer の関係

```
Timer (時計) → Service (プロセス) → 実際のプログラム
```

- **Service を stop**: 今動いているプロセスだけ停止、タイマーは動き続ける
- **Timer を stop**: 定時実行を停止、今動いているプロセスはそのまま

---

### 停止パターン

#### パターン1: 今動いているプロセスだけ停止（タイマーは残す）

```bash
systemctl --user stop eqmonitor.service
```

- ✅ 今のプロセスが停止
- ✅ 次の定時（23:00など）にまた自動起動される

---

#### パターン2: 定時実行を停止（プロセスは残る）

```bash
systemctl --user stop eqmonitor.timer
```

- ✅ タイマーが停止
- ❌ 今動いているプロセスは継続
- ✅ 次の定時になっても新しいプロセスは起動しない

---

#### パターン3: 完全停止（両方止める）

```bash
systemctl --user stop eqmonitor.timer
systemctl --user stop eqmonitor.service
```

または

```bash
systemctl --user stop eqmonitor.timer eqmonitor.service
```

---

### enable と disable

- **enable**: システム起動時に自動起動する設定
- **disable**: 自動起動を無効化

```bash
# Timer の enable/disable は意味がある
systemctl --user enable eqmonitor.timer   # 再起動後も自動起動
systemctl --user disable eqmonitor.timer  # 再起動後は起動しない

# Service の enable/disable は意味がない（Timerから呼ばれるだけ）
systemctl --user enable eqmonitor.service   # ❌ ほぼ無意味
```

---

### 拡張子の省略

```bash
# .service は省略可能
systemctl --user stop eqmonitor.service
systemctl --user stop eqmonitor            # 同じ意味

# .timer は省略不可
systemctl --user stop eqmonitor.timer      # ✅ 正しい
systemctl --user stop eqmonitor            # ❌ これは .service を止める
```

**推奨**: 混乱を避けるため、常に `.service` `.timer` を明示する

---

## ログクリアについて

### 基本方針: **消さなくてOK**

systemd の journal は **Windowsのイベントログと同じ** です：

- トラブル時の調査に役立つ
- 自動で圧縮・ローテート
- デフォルトで数MB～数百MB程度（ディスク圧迫しない）

**消すのはアンチパターン**。Linux文化では**ログは資産**として扱います。

---

### どうしても消したい場合

```bash
# 期間指定で削除（7日より古いもの）
journalctl --user --vacuum-time=7d

# サイズ指定で削除（10MB以下に削減）
journalctl --user --vacuum-size=10M

# 特定サービスのログのみ削除
journalctl --user --vacuum-time=1s -u eqmonitor.service
```

---

### ログ使用量の確認

```bash
journalctl --user --disk-usage
# 例: Archived and active journals take up 8.0M in the file system.
```

通常は気にする必要なし。

---

## Serviceを修正した場合

**必ず `daemon-reload` が必要**：

```bash
# 1. 設定ファイルを編集
nano ~/.config/systemd/user/eqmonitor.service

# 2. 再読み込み（必須）
systemctl --user daemon-reload

# 3. サービス再起動（変更を反映）
systemctl --user restart eqmonitor.service

# または Timer 経由で動かす場合
systemctl --user restart eqmonitor.timer
```

`daemon-reload` を忘れると、**変更が反映されない**ので注意。

---

## reboot時に即起動したい場合

### Timer に設定を追加する

```ini
[Unit]
Description=Start P2PQuake Receiver every hour

[Timer]
OnCalendar=hourly
OnBootSec=2min        # ← これを追加（起動2分後に実行）

[Install]
WantedBy=timers.target
```

- **OnBootSec=2min**: システム起動2分後に1回実行
- **OnCalendar=hourly**: その後は毎時0分に実行

### 動作の流れ

```
システム起動
  ↓
2分後: eqmonitor.service 起動（OnBootSec）
  ↓
その後: 毎時0分に eqmonitor.service 起動（OnCalendar）
```

### Service を enable しても意味がない

Service に `[Install]` を追加して enable しても：

```ini
# こうしても...
[Service]
...

[Install]
WantedBy=default.target
```

```bash
systemctl --user enable eqmonitor.service
```

これは **「システム起動時に Service を常駐させる」** という意味になります。

- Service が起動しっぱなし（WebSocket接続が1つだけ）
- Timer は無視される（定時実行されない）

**あなたが望む動作ではありません。**

---

## crontab の例（比較用）

### 基本的な crontab

```cron
# 毎時0分に実行
0 * * * * /home/kkano/eqmonitor/start.sh >> /home/kkano/eqmonitor/cron.log 2>&1
```

### start.sh（プロセス重複チェック付き）

```bash
#!/bin/bash
cd /home/kkano/eqmonitor

# 既に起動しているかチェック
if pgrep -f "python.*mon.py" > /dev/null; then
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] Already running. Skipping."
    exit 0
fi

# 起動
echo "[$(date '+%Y-%m-%d %H:%M:%S')] Starting mon.py..."
.venv/bin/python -u mon.py --env prod --ignore 3 2>> error.log
```

```bash
chmod +x /home/kkano/eqmonitor/start.sh
```

---

## どちらを選ぶべきか

### crontab が向いているケース

- ✅ 個人の実験・開発用スクリプト
- ✅ シンプルな定時実行だけで十分
- ✅ root権限が使えない
- ✅ 設定が簡単な方がいい
- ✅ すぐに始めたい（学習コストを避けたい）

---

### systemd が向いているケース

- ✅ 本番サービス・重要なアプリ
- ✅ 自動再起動・依存関係管理が必要
- ✅ ログを統合管理したい（journalctl）
- ✅ プロセス管理を systemctl で統一したい
- ✅ 実行時間制限などの高度な制御が必要
- ✅ 責務を明確に分離したい

---

## 結論: どちらも一長一短

| 項目 | crontab | systemd |
|------|---------|---------|
| 学習コスト | 低い | 高い（初見殺し） |
| 機能の豊富さ | 限定的 | 非常に豊富 |
| 設定の手軽さ | ✅ 非常に簡単 | ❌ 複雑 |
| ログ管理 | 自前で実装 | journalctl 統合 |
| プロセス管理 | シェルスクリプト | systemctl で統一 |
| 責務の分離 | すべて1行 | Timer/Service で分離 |
| 本番環境での採用 | 減少傾向 | 標準 |

**機能の豊富さ・設計思想では systemd に軍配**が上がりますが、**個人用途なら crontab で十分**なケースも多いです。

systemd は初見では分かりにくいですが、**責務の分離という設計思想を理解すれば納得できる**はずです。

---

## Python実行時の注意: バッファリング問題

Pythonはデフォルトで標準出力をバッファリングします。systemd で使う場合は **必ず `-u` オプション** を付けてください：

```ini
ExecStart=/path/to/python -u script.py
```

または環境変数で設定：

```ini
Environment="PYTHONUNBUFFERED=1"
ExecStart=/path/to/python script.py
```

これを忘れると `print()` の内容が journalctl にリアルタイムで表示されません。

---

## その他の Tips

### ユーザーサービスをログインなしで動かす

```bash
# 管理者として実行（初回のみ）
sudo loginctl enable-linger kkano

# 確認
loginctl show-user kkano | grep Linger
# Linger=yes ならOK
```

これで再起動後、kkano がログインしなくてもユーザーサービスが動きます。

---

### systemd の視覚的な管理

- **netdata**: `http://localhost:19999` → systemd セクション
- **cockpit**: Web UI で systemd を管理（`sudo apt install cockpit`）
- **systemd-manager**: GTK アプリ（`sudo apt install systemd-manager`）

---

### よくあるトラブルと対処法

#### 1. タイマーが動かない

```bash
# タイマーが enabled & started か確認
systemctl --user list-timers
systemctl --user status eqmonitor.timer

# Active: active (waiting) ならOK
# Active: inactive (dead) なら start していない
systemctl --user start eqmonitor.timer
```

---

#### 2. 設定を変更したのに反映されない

```bash
# daemon-reload を忘れている
systemctl --user daemon-reload
systemctl --user restart eqmonitor.timer
```

---

#### 3. ログが表示されない

```bash
# Python のバッファリング問題
# ExecStart に -u を追加
ExecStart=/path/to/python -u script.py
```

---

#### 4. Permission denied

```bash
# ユーザーサービスなのに sudo を使っている
systemctl --user status eqmonitor.service  # ✅ 正しい
sudo systemctl status eqmonitor.service    # ❌ 間違い（rootのサービスを探す）
```

---

#### 5. enable しようとしたらエラーが出る

```bash
systemctl --user enable eqmonitor.service
# エラー: "[Install] セクションがないので enable できません"
```

**これは正常です。** Timer から呼ばれる Service には `[Install]` は不要。Timer だけ enable してください：

```bash
systemctl --user enable eqmonitor.timer  # ✅ 正しい
```

---

## systemd の思想を理解するための例

### 例1: Webサーバー（nginx）

**責務の分離:**

```
nginx.socket (ポート80監視)
  ↓ 接続があったら
nginx.service (nginxプロセス起動)
```

- Socket: いつ起動するか（ポート監視）
- Service: 何を起動するか（nginx実行）

**利点:**
- Socket だけ有効にしておけば、初回アクセス時に自動起動（遅延起動）
- メモリ節約（使われない時は起動しない）

---

### 例2: 定期バックアップ

**責務の分離:**

```
backup.timer (毎日3:00)
  ↓
backup.service (バックアップスクリプト実行)
```

- Timer: スケジューリング（毎日3:00）
- Service: バックアップ処理

**利点:**
- スケジュール変更は Timer だけ編集
- バックアップ処理変更は Service だけ編集
- 手動バックアップも可能（`systemctl start backup.service`）

---

### 例3: ファイル監視

**責務の分離:**

```
file-watcher.path (/var/log を監視)
  ↓ ファイル変更を検知
log-processor.service (ログ解析実行)
```

- Path: トリガー（ファイル変更監視）
- Service: 処理内容（ログ解析）

**利点:**
- 監視対象変更は Path だけ編集
- 処理内容変更は Service だけ編集
- 手動実行も可能

---

## cron vs systemd の具体例

### シナリオ: 毎時Pythonスクリプトを実行

**cron の場合:**

```cron
0 * * * * cd /home/kkano/app && /home/kkano/app/.venv/bin/python -u script.py >> /home/kkano/app/log.txt 2>&1
```

- ✅ 1行で完結
- ❌ 実行時間制限なし（暴走の可能性）
- ❌ 自動再起動なし
- ❌ ログローテートは自前実装
- ❌ プロセス監視は別途必要

---

**systemd の場合:**

```ini
# app.timer
[Unit]
Description=Run app every hour

[Timer]
OnCalendar=hourly
OnBootSec=2min

[Install]
WantedBy=timers.target
```

```ini
# app.service
[Unit]
Description=Python App
After=network-online.target

[Service]
Type=simple
WorkingDirectory=/home/kkano/app
ExecStart=/home/kkano/app/.venv/bin/python -u script.py
Restart=on-failure
RestartSec=5
RuntimeMaxSec=3600
TimeoutStopSec=10

StandardOutput=journal
StandardError=journal
```

```bash
systemctl --user enable --now app.timer
```

- ❌ 設定が複雑
- ✅ 実行時間制限（1時間で強制終了）
- ✅ 異常終了時に自動再起動
- ✅ ログは journal が自動管理
- ✅ `systemctl status` でプロセス監視

---

## 設定ファイルの最小構成

### 最小限の Timer

```ini
[Timer]
OnCalendar=hourly

[Install]
WantedBy=timers.target
```

これだけで動きます。`[Unit]` の `Description` は省略可能。

---

### 最小限の Service

```ini
[Service]
ExecStart=/path/to/command
```

これだけで動きます。他はすべてデフォルト値が使われます。

---

### 実用的な最小構成（推奨）

```ini
# eqmonitor.timer
[Unit]
Description=EQ Monitor Timer

[Timer]
OnCalendar=hourly

[Install]
WantedBy=timers.target
```

```ini
# eqmonitor.service
[Unit]
Description=EQ Monitor

[Service]
WorkingDirectory=/home/kkano/eqmonitor
ExecStart=/home/kkano/eqmonitor/.venv/bin/python -u mon.py --env prod
Restart=on-failure
RuntimeMaxSec=3600

StandardOutput=journal
StandardError=journal
```

これで本番運用可能です。

---

## まとめ: systemd を使うべきか

### あなたが個人開発者なら

**crontab から始めて、必要に応じて systemd に移行**

1. まず crontab でプロトタイプ
2. 安定してきたら systemd に移行
3. 本番環境では systemd

---

### あなたがチーム開発・本番環境なら

**最初から systemd**

- 標準化されている
- チームメンバーが理解しやすい
- トラブルシューティングが容易
- ログ管理が統一される

---

### systemd の学習曲線

```
難易度
 ^
 |     cron
 |     ----
 |
 |            systemd
 |            -------
 |                    \
 |                     \___（慣れると簡単）
 +-------------------------> 時間
```

**初見は難しいが、慣れれば cron より使いやすい**

---

## 最後に: systemd の哲学

> **"Everything is a unit"**
> （すべてはユニットである）

- Service: プロセス実行
- Timer: スケジューリング
- Socket: ポート監視
- Path: ファイル監視
- Target: グループ化
- Mount: マウント管理
- Device: デバイス管理

**すべてが同じ仕組みで管理される → 統一感**

これが systemd の強みであり、同時に初見で分かりにくい理由でもあります。

---

## クイックリファレンス

### 最もよく使うコマンド

```bash
# 設定変更後は必ず
systemctl --user daemon-reload

# タイマー管理
systemctl --user enable --now eqmonitor.timer   # 有効化＋起動
systemctl --user stop eqmonitor.timer           # 停止
systemctl --user restart eqmonitor.timer        # 再起動
systemctl --user status eqmonitor.timer         # 状態確認
systemctl --user list-timers                    # 一覧表示

# ログ確認
journalctl --user -u eqmonitor.service -f       # リアルタイム表示
journalctl --user -u eqmonitor.service -n 50    # 最新50行
journalctl --user -u eqmonitor.service --since today  # 今日のログ

# プロセス停止
systemctl --user stop eqmonitor.service         # 今のプロセスのみ
systemctl --user stop eqmonitor.timer           # 定時実行を停止
```

---

### 設定ファイルのテンプレート

**コピペ用（Timer）:**

```ini
[Unit]
Description=My App Timer

[Timer]
OnCalendar=hourly
OnBootSec=2min

[Install]
WantedBy=timers.target
```

**コピペ用（Service）:**

```ini
[Unit]
Description=My App Service
After=network-online.target

[Service]
Type=simple
WorkingDirectory=/home/user/myapp
ExecStart=/home/user/myapp/.venv/bin/python -u main.py
Restart=on-failure
RuntimeMaxSec=3600

StandardOutput=journal
StandardError=journal
```

**配置:**

```bash
mkdir -p ~/.config/systemd/user
# ファイルをコピー
systemctl --user daemon-reload
systemctl --user enable --now myapp.timer
```

---

これで systemd の全体像が理解できたはずです。**初見では複雑に見えますが、責務の分離という設計思想を理解すれば納得できる**はずです。頑張ってください！ 🚀
