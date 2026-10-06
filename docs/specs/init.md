# Google Drive Markdown Note Editor

## 1. 目的

**Google Drive上のMarkdownノートを軽快に編集するための専用エディタ。**

基本思想は、

> Drive上のMarkdownを開く → 左で編集する → 右でHTMLとして確認する → 同じDriveファイルへ保存する

だけに集中する。

---

## 2. UI

```text id="e5g6y3"
┌─────────────────────────┬─────────────────────────┐
│ Markdown                │ HTML Preview            │
│                         │                         │
│ # Hello                 │ Hello                   │
│                         │                         │
│ This is **Markdown**.   │ This is Markdown.       │
│                         │                         │
└─────────────────────────┴─────────────────────────┘
```

* 左：Markdown Editor
* 右：HTML Preview
* 左右のスクロール位置を大まかに同期
* ペイン幅を変更可能
* Markdown編集時にPreviewを更新
* UI文字列は**英語固定**
* UIの多言語化は行わない

### エディタ

**CodeMirror 6を使用。**

* 自前のテキストエディタは実装しない
* 必要なCodeMirror extensionのみ使用
* Markdown syntax highlighting等はCodeMirrorに任せる

---

## 3. Markdown対応

**フルセット対応を目標とする。**

対応仕様：

* **CommonMark**
* **GitHub Flavored Markdown (GFM)**

設定から切り替え可能。

```text id="8x0v47"
Markdown Flavor

○ CommonMark
○ GitHub Flavored Markdown
```

選択したFlavorは、エディタだけでなく**HTML Previewのレンダリング仕様にも反映**する。

```text id="e4q0fd"
Markdown
    ↓
Selected Flavor
    ↓
HTML
    ↓
Preview
```

GFM固有の構文もGFM選択時にはHTMLへ正しくレンダリングする。

---

## 4. Drive連携

製品としての保存先は**Google Driveのみ**。

### Open

* Drive上のMarkdownファイルを開く
* Drive Integration Layerが`fileId`等のDrive固有情報を保持
* ファイル内容をMarkdown Coreへ渡す

### Save

* `Ctrl+S` → 現在開いている**同じDriveファイル**を更新
* Auto Save → 同じファイルへ更新
* ローカルファイルへの保存は行わない

つまり、例の

> Ctrl+S → DownloadsにMarkdownが落ちる

という挙動は絶対にしない。

---

## 5. Storage Callback設計

**Markdown CoreからGoogle Drive APIを直接呼ばない。**

保存先とのやり取りはcallback経由。

概念的には：

```ts id="b3xvtd"
type Storage = {
    load: () => Promise<string>;
    save: (content: string) => Promise<void>;
};
```

Coreは、

```text id="9ydq77"
storage.load()
storage.save(content)
```

だけを呼ぶ。

Drive Integration Layerが実際のcallbackを提供する。

```text id="e4t8b4"
Google Drive
     │
 Drive API
     │
     ▼
Drive Integration Layer
     │
 load / save callbacks
     │
     ▼
Markdown Core
```

したがってCoreは、

* Google Drive
* Google API
* `fileId`
* OAuth

を知らない。

これは**他の保存先をサポートするためというより、CoreとDrive実装を綺麗に分離するための設計**。

現時点でローカルストレージ等の実装は作らない。

---

## 6. 保存

### Ctrl+S

```text id="i9k8ry"
Ctrl+S
 ↓
save callback
 ↓
Drive update
```

* 現在開いているファイルを更新
* ローカルへのダウンロードは発生しない

### Auto Save

* 編集後のdebounce方式
* 最新のMarkdownをDriveへ保存
* 保存中に編集された場合は最新状態を次回保存
* 保存失敗しても編集内容を保持
* 再試行可能

状態表示：

```text id="zv0p93"
Saved
Saving...
Save failed
```

---

## 7. 起動・読み込み性能

**重要な非機能要件。**

目標は、

> 「Driveアプリを起動する」のではなく「ノートを開く」感覚。

そのため、

* 起動時の不要なDrive APIアクセスを避ける
* 必要なデータだけ取得
* ファイル取得後できるだけ早くEditorを表示
* Preview生成でEditorの操作開始をブロックしない
* 不要なCodeMirror extensionを読み込まない
* 不要な依存を導入しない
* 大きなMarkdownでも極端に重くならないようにする
* 初期ロード量を可能な限り小さくする

---

## 8. 設定

設定項目は最小限。

### Font Size

* Small
* Medium
* Large

### Theme

* Light
* Dark

### Markdown Flavor

* CommonMark
* GitHub Flavored Markdown

それ以外の細かいカスタマイズは原則として提供しない。

---

## 9. 外部ライブラリ方針

**外部ライブラリは利用するが、パッケージレジストリには依存しない。**

採用したライブラリは**ソースコードごとプロジェクトに取り込んでビルド**する。

### 方針

* npm等から実行時・ビルド時に取得する構成にしない
* パッケージレジストリへの恒久的な依存を作らない
* 採用ライブラリのソースをリポジトリ内に保持
* バージョンを固定
* ライセンス情報を保持
* 自動アップデートしない
* 更新は必要時のみ手動で行う

目的は、

> **パッケージ配信停止・unpublish・レジストリ消滅によって将来ビルドできなくなることを防ぐ。**

### ライブラリ選定基準

* 枯れている
* 長期的な実績がある
* APIが安定している
* breaking changeが少ない
* 必要以上に巨大でない
* ライセンス上問題がない
* ソースコードを取り込んで維持できる

主要候補：

* CodeMirror 6
* 成熟したCommonMark/GFM parser
* Google公式のDrive/OAuth関連コード・SDK

---

## 10. 依存関係

依存はできるだけ少なくする。

```text id="4eg4ji"
Application
 │
 ├── CodeMirror 6
 ├── Markdown parser
 └── Google Drive integration
```

ライブラリの上にさらに大量のライブラリを積む構成は避ける。

また、外部ライブラリとの境界を薄くし、将来置換できるようにする。

特に、

```text id="t5m1wq"
Markdown Core
      │
      └── Markdown parser adapter
```

```text id="ax2w0j"
Markdown Core
      │
      └── Storage callback
```

のようにする。

---

## 11. スコープ外

意図的に実装しない。

* ローカルファイル保存
* Dropbox等の他クラウド
* WYSIWYG編集
* 多言語UI
* 高機能ファイルマネージャ
* リッチテキスト編集
* 複雑なノート管理
* 大量のユーザー設定
* 不要なオンラインサービスへの依存

---

## 12. コード規模

**アプリ自身の自前コード：4,000〜8,000 SLOC程度**

目標は、

**5,000〜6,000 SLOC前後。**

ただし外部ライブラリをソースごと取り込むため、

* **Application SLOC**
* **Vendored dependency SLOC**

は分けて管理する。

プロジェクト全体のSLOCはCodeMirrorやMarkdown parserの取り込み分だけ大きくなる。

---

## 13. アーキテクチャ

```text id="d7z6mt"
                  Google Drive
                       │
                    Drive API
                       │
                       ▼
          ┌────────────────────────┐
          │ Drive Integration      │
          │                        │
          │ OAuth                  │
          │ File selection         │
          │ fileId management      │
          │ load callback          │
          │ save callback          │
          └───────────┬────────────┘
                      │
                Storage callbacks
                      │
                      ▼
          ┌────────────────────────┐
          │ Markdown Core          │
          │                        │
          │ Document state         │
          │ Auto Save              │
          │ CodeMirror 6           │
          │ Markdown parser        │
          │ HTML rendering         │
          └───────────┬────────────┘
                      │
                ┌─────┴─────┐
                ▼           ▼
           Markdown       HTML
            Editor       Preview
```

**製品としてはDrive専用。内部ではStorage callbackでCoreとDriveを分離。外部依存はソースごと固定。UIと機能は極小。Markdownは本格対応。**

この4点が設計の中心になる。

なお、兄弟リポジトリのboem-conlang-studioの開発体制を参考にする。
設計のクリアさなど。