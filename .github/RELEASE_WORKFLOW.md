# リリース自動化ワークフロー

PR に `release:*` ラベルを付けてマージするだけで、ファームウェアの自動ビルドと GitHub Release の作成が行われます。

---

## ファイル構成

```
.github/
  workflows/
    check-release-label.yml   # PR のラベルチェック（必須ラベルが1つだけあるか確認）
    create-release.yml        # ビルド・タグ作成・GitHub Release 作成
  RELEASE_WORKFLOW.md         # このファイル
```

---

## 使い方

### 1. ラベルを付ける

PR を作成したら、以下の `release:*` ラベルを **1つだけ** 付けてください。

| ラベル | 動作 | 使うタイミング |
|---|---|---|
| `release:major` | MAJOR バージョンを上げてリリース | 破壊的変更・後方互換性のない変更 |
| `release:minor` | MINOR バージョンを上げてリリース | 後方互換のある新機能追加 |
| `release:patch` | PATCH バージョンを上げてリリース | バグ修正・軽微な改善 |
| `release:none` | リリースしない | ドキュメント・CI・ワークフロー変更など |

ラベルが付いていない場合、または2枚以上付いている場合は `check-release-label` チェックが失敗し、マージできません。

### 2. マージする

`release:none` 以外のラベルでマージすると、以下が自動実行されます。

```
マージ
  ↓
バージョン番号を計算（既存タグから自動計算）
  ↓
ファームウェアをビルド（px4_fmu-v6c_default, px4_fmu-v6x_default）
  ↓
git タグを作成・push（例: v1.16.1-0.2.1）
  ↓
GitHub Release を作成し .px4 ファイルを添付
```

### 3. バージョン形式

```
v1.16.1-<MAJOR>.<MINOR>.<PATCH>
```

- `v1.16.1` は PX4 ベースバージョン（`create-release.yml` 内の `PX4_BASE` 変数）
- `MAJOR.MINOR.PATCH` はカスタムファームウェアのバージョン
- 初回リリースはラベルの種類に関わらず `v1.16.1-0.1.0` になります

---

## 別のリポジトリへの導入手順

### Step 1: ワークフローファイルをコピー

```bash
mkdir -p .github/workflows
cp check-release-label.yml .github/workflows/
cp create-release.yml .github/workflows/
```

### Step 2: カスタマイズ箇所を変更

以下の箇所を環境に合わせて変更してください。

#### `check-release-label.yml`

```yaml
on:
  pull_request:
    branches:
      - v1.16.1-main   # ★ デフォルトブランチ名に変更
```

#### `create-release.yml`

```yaml
on:
  pull_request:
    branches:
      - v1.16.1-main   # ★ デフォルトブランチ名に変更
```

```bash
PX4_BASE="v1.16.1"   # ★ 使用する PX4 バージョンに変更（タグ形式に影響）
```

```bash
make px4_fmu-v6c_default   # ★ 使用するボード名に変更
make px4_fmu-v6x_default   # ★ 使用するボード名に変更
```

```bash
cp build/px4_fmu-v6c_default/px4_fmu-v6c_default.px4 "px4_fmu-v6c-v${NEXT_VERSION}.px4"
cp build/px4_fmu-v6x_default/px4_fmu-v6x_default.px4 "px4_fmu-v6x-v${NEXT_VERSION}.px4"
# ★ ボード名に合わせてパスとファイル名を変更
```

変更箇所まとめ：

| 場所 | 変数 / 行 | 変更内容 |
|---|---|---|
| 両ファイル `branches:` | `v1.16.1-main` | デフォルトブランチ名 |
| `create-release.yml` | `PX4_BASE` | 使用する PX4 バージョン（例: `v1.16.0`） |
| `create-release.yml` Build firmware | `make px4_fmu-v6c_default` 等 | ビルドするボード名 |
| `create-release.yml` Build firmware | `cp build/...` 等 | ボード名に合わせたパス |
| `create-release.yml` Build firmware | `v6c_asset=` / `v6x_asset=` | 出力変数のボード名 |
| `create-release.yml` Create GitHub Release | `"$V6C_ASSET" "$V6X_ASSET"` | Release に添付するファイル |

### Step 3: コミット・PR 作成・マージ

```bash
git add .github/workflows/
git commit -m "feat: add release automation workflows"
git push origin <ブランチ名>
```

GitHub で PR を作成し、`release:none` ラベルを付けてマージします（ワークフロー自体はリリース不要なため）。

### Step 4: ラベルを作成

リポジトリの **Issues → Labels** で以下の4つを作成してください。

| ラベル名 | 推奨カラー |
|---|---|
| `release:major` | `#B60205`（赤） |
| `release:minor` | `#0075CA`（青） |
| `release:patch` | `#0E8A16`（緑） |
| `release:none` | `#E4E669`（黄） |

`gh` CLI がある場合は以下のコマンドで一括作成できます：

```bash
gh label create "release:major" --color "B60205" --description "Breaking change (MAJOR bump)"
gh label create "release:minor" --color "0075CA" --description "New feature, backward compatible (MINOR bump)"
gh label create "release:patch" --color "0E8A16" --description "Bug fix or minor improvement (PATCH bump)"
gh label create "release:none"  --color "E4E669" --description "No release (docs, CI changes, etc.)"
```

### Step 5: 動作確認

テスト用ブランチで PR を作成し、`release:patch` ラベルでマージします。  
初回リリース `v<PX4_BASE>-0.1.0` が作成されれば導入完了です。

---

## ビルドターゲットの追加・変更

### ボードを追加する場合

`create-release.yml` の `Build firmware` ステップと `Create GitHub Release` ステップを以下のように変更します。

**Before（2ボード）:**

```yaml
- name: Build firmware
  id: build
  run: |
    NEXT_VERSION="${{ steps.calc-version.outputs.next_version }}"

    make px4_fmu-v6c_default
    make px4_fmu-v6x_default

    cp build/px4_fmu-v6c_default/px4_fmu-v6c_default.px4 "px4_fmu-v6c-v${NEXT_VERSION}.px4"
    cp build/px4_fmu-v6x_default/px4_fmu-v6x_default.px4 "px4_fmu-v6x-v${NEXT_VERSION}.px4"

    echo "v6c_asset=px4_fmu-v6c-v${NEXT_VERSION}.px4" >> "$GITHUB_OUTPUT"
    echo "v6x_asset=px4_fmu-v6x-v${NEXT_VERSION}.px4" >> "$GITHUB_OUTPUT"
```

**After（3ボード追加例: `px4_fmu-v5x_default`）:**

```yaml
- name: Build firmware
  id: build
  run: |
    NEXT_VERSION="${{ steps.calc-version.outputs.next_version }}"

    make px4_fmu-v5x_default
    make px4_fmu-v6c_default
    make px4_fmu-v6x_default

    cp build/px4_fmu-v5x_default/px4_fmu-v5x_default.px4 "px4_fmu-v5x-v${NEXT_VERSION}.px4"
    cp build/px4_fmu-v6c_default/px4_fmu-v6c_default.px4 "px4_fmu-v6c-v${NEXT_VERSION}.px4"
    cp build/px4_fmu-v6x_default/px4_fmu-v6x_default.px4 "px4_fmu-v6x-v${NEXT_VERSION}.px4"

    echo "v5x_asset=px4_fmu-v5x-v${NEXT_VERSION}.px4" >> "$GITHUB_OUTPUT"
    echo "v6c_asset=px4_fmu-v6c-v${NEXT_VERSION}.px4" >> "$GITHUB_OUTPUT"
    echo "v6x_asset=px4_fmu-v6x-v${NEXT_VERSION}.px4" >> "$GITHUB_OUTPUT"
```

**Create GitHub Release ステップも更新:**

```yaml
- name: Create GitHub Release
  run: |
    ...
    V5X_ASSET="${{ steps.build.outputs.v5x_asset }}"   # 追加
    V6C_ASSET="${{ steps.build.outputs.v6c_asset }}"
    V6X_ASSET="${{ steps.build.outputs.v6x_asset }}"

    ...
    echo "  - px4_fmu-v5x_default"   # Boards セクションに追加
    echo "  - px4_fmu-v6c_default"
    echo "  - px4_fmu-v6x_default"
    ...
    echo "  - ${V5X_ASSET}"          # Artifacts セクションに追加
    echo "  - ${V6C_ASSET}"
    echo "  - ${V6X_ASSET}"
    ...

    gh release create "$GIT_TAG" \
      --title "Custom Firmware v${NEXT_VERSION}" \
      --notes-file release-body.md \
      "$V5X_ASSET" "$V6C_ASSET" "$V6X_ASSET"   # ファイルを追加
```

### カスタムビルドオプションを付ける場合

`make` コマンドに変数を渡すことができます。

```bash
# 例: 並列ビルドを有効化
make -j$(nproc) px4_fmu-v6c_default

# 例: カスタム cmake 変数を指定
make px4_fmu-v6c_default EXTRA_CMAKE_ARGS="-DMY_OPTION=ON"
```

---

## トラブルシューティング

### check-release-label が失敗する

- `release:*` ラベルが付いていない → ラベルを1つ付けてください
- `release:*` ラベルが2つ以上付いている → 1つだけ残して他を外してください
- ラベルが存在しない → リポジトリの Labels ページでラベルを作成してください

### create-release ワークフローが Actions の一覧に表示されない

`pull_request: closed` トリガーのワークフローは手動実行に対応していないため、Actions の一覧ページには表示されません。PR をマージすることで初めて実行されます。また、ワークフローファイルがデフォルトブランチに存在しないと表示されません。

### ビルドが失敗する

- サブモジュールが不足していないか確認（`submodules: recursive` で取得しています）
- `Tools/setup/ubuntu.sh` が正常に完了しているかログを確認してください
- ボード名のスペルを確認してください（`make <board>` でローカルでも確認可能）

### タグが重複してエラーになる

同じバージョンのタグがすでに存在する場合に発生します。手動でタグを削除してください：

```bash
git tag -d v1.16.1-0.1.0
git push origin --delete v1.16.1-0.1.0
```
