# Gitメモ

##### Remote変更
```bash
# リモートリポジトリを追加
git remote add origin git@github.com:k-kanoh/playbooks.git

# リモートリポジトリの確認
git remote -v

# 初回プッシュ
git push -u origin main
```

##### タグ操作
```bash
# タグを作成
git tag v1.0

# ローカルのタグを削除
git tag -d v1.0

# タグをリモートにプッシュ
git push origin v1.0

# リモートのタグを削除
git push --delete origin v1.0
```

## ブランチ操作

##### 履歴をリセットして現在の状態を新しいInitial commitにする
```bash
# 現在の内容を保持したまま履歴を持たない新しいブランチを作成
git checkout --orphan new_main

# 新しい Initial commit を作成（すべてのファイルは既にステージング済み）
git commit -m "Initial commit"

# 古い main ブランチを削除
git branch -D main

# new_main を main にリネーム
git branch -m main

# リモートに強制プッシュ
git push -f origin main
```

##### developブランチをmainブランチにする
```bash
# developブランチの内容でmainブランチを上書き
git switch develop
git branch -f main develop
git switch main
```

※ `.git/refs/heads/main` を develop の HEAD と同じコミットIDにするのと同じ
