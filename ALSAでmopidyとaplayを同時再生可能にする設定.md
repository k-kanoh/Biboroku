# ALSAでmopidyとaplayを同時再生可能にする設定

## 問題発生

mopidy で音楽を再生中（低音量）に、aplay で緊急地震速報（通常音量）を鳴らそうとするとエラーが発生。

### エラー内容

```bash
$ aplay 音声/1.wav
aplay: main:834: audio open error: Device or resource busy
```

### mopidy.conf の内容

```ini
[audio]
output = alsasink device=mopidy

[http]
enabled = true
hostname = 0.0.0.0
port = 6680

[mpd]
enabled = true
hostname = 0.0.0.0
port = 6600

[stream]
enabled = true
protocols = http,https,mms,rtmp,rtmps,rtsp
timeout = 5000
```

**原因**: ALSA は通常、1つのアプリケーションしかハードウェアデバイスを使えない排他制御になっている。

## dmix を使った解決を試みる

複数アプリで同時に音を鳴らすため、dmix（ダイナミックミキシング）を設定。

### /etc/asound.conf を作成

```conf
pcm.!default {
    type plug
    slave.pcm "dmixed"
}

pcm.dmixed {
    type dmix
    ipc_key 1024
    slave.pcm "hw:0,0"
}

pcm.mopidy {
    type softvol
    slave.pcm "dmixed"
    control.name "Mopidy"
}
```

mopidy を再起動して aplay を実行すると、今度は別のエラー：

```bash
$ aplay 音声/1.wav
ALSA lib pcm_direct.c:2180:(_snd_pcm_direct_new) unable to create IPC semaphore
aplay: main:834: audio open error: Permission denied
```

## 権限の問題を発見

IPC セマフォの権限を確認：

```bash
$ ipcs -s

------ Semaphore Arrays --------
key        semid      owner      perms      nsems
0x00000400 30         mopidy     600        1
```

**問題点**: `perms` が `600`（所有者のみ読み書き可）になっているため、kkano ユーザーが IPC セマフォにアクセスできない。

dmix は複数のプロセス間で音声をミキシングするため、プロセス間通信（IPC）のセマフォを使います。権限が `600` だと、mopidy ユーザーが作成したセマフォに、kkano ユーザーがアクセスできません。

## ipc_perm で権限を設定して解決

`/etc/asound.conf` に `ipc_perm 0666` を追加：

```conf
pcm.!default {
    type plug
    slave.pcm "dmixed"
}

pcm.dmixed {
    type dmix
    ipc_key 1024
    ipc_perm 0666      # ← これを追加
    slave.pcm "hw:0,0"
}

pcm.mopidy {
    type softvol
    slave.pcm "dmixed"
    control.name "Mopidy"
}
```

### 既存の IPC を削除して再起動

```bash
# 既存の IPC セマフォを削除
sudo ipcrm -s 30

# mopidy を再起動
sudo systemctl restart mopidy

# テスト
aplay 音声/1.wav
```

### 結果

```bash
$ ipcs -s

------ Semaphore Arrays --------
key        semid      owner      perms      nsems
0x00000400 32         mopidy     666        1
```

権限が `666` になり、mopidy 再生中でも aplay で音が鳴るようになった！

## 補足：より安全な設定

シングルユーザー環境では `ipc_perm 0666` で問題ありませんが、マルチユーザー環境ではグループベースの権限がより安全です。

### ユーザーを audio グループに追加

```bash
sudo usermod -a -G audio kkano
sudo usermod -a -G audio mopidy

# 確認
groups kkano   # kkano docker audio
groups mopidy  # mopidy audio
```

### asound.conf でグループ指定

```conf
pcm.dmixed {
    type dmix
    ipc_key 1024
    ipc_perm 0660        # グループ内で共有
    ipc_gid audio        # audio グループを指定
    slave.pcm "hw:0,0"
}
```

これで audio グループのメンバーだけが IPC にアクセスできるようになり、セキュリティが向上します。

## まとめ

- **dmix**: 複数アプリの音声をミキシング
- **ipc_perm 0666**: 異なるユーザー間で IPC セマフォを共有可能に
- **より安全**: `ipc_perm 0660` + `ipc_gid audio` でグループベースのアクセス制御
- **softvol + max_dB**: mopidy だけ音量を制限

これで mopidy の音楽再生中でも、緊急地震速報を通常音量で鳴らせるようになりました。
