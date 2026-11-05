# MyVM — Swift Playgrounds 向け VM 管理アプリ（エンジン未接続版）

確認日: 2026-11-05

## このアプリの位置づけ（重要）

- **ゲスト OS は起動しません。** Swift Playgrounds 上では実エンジンを組み込めないため、
  起動操作は「エンジン未接続」の理由付きエラーになります（成功したふりはしません）。
- 実装済み: VM 設定の保存、RAW ディスク作成、ISO/IMG/QCOW2 取り込み（検証・SHA-256）、
  スナップショット（停止中のディスクコピー）、複製、エクスポート/インポート、TCP/HTTPS テスト、
  描画・入力経路の診断。
- エンジン接続用の境界は `VirtualizationBackend` プロトコル。`VMStore.backendFactory` を差し替えます。
- **このコードは私の環境ではコンパイル・実機確認できていません。** エラーが出たらメッセージを送ってください。

## 前提

- iPad: Swift Playgrounds 4.x 以降、iPadOS 17 以上
- App プロジェクトの設定で最小 iOS を 17 以上にする（`ContentUnavailableView` 等を使用）

## ファイル構成

| ファイル | 役割 |
|---|---|
| MyApp.swift | エントリポイント |
| VMConfiguration.swift | モデル・状態・ハード上限 |
| VirtualizationBackend.swift | エンジン抽象化、FramebufferSource、UnavailableBackend |
| StorageManager.swift | VM フォルダ、RAW ディスク、スナップショット |
| ImageManager.swift | イメージ検証・コピー・SHA-256 |
| VMStore.swift | 設定管理・VM 管理・状態管理（ViewModel） |
| NetworkManager.swift | TCP / HTTPS 接続テスト |
| VMListView.swift | VM 一覧 |
| CreateVMView.swift | VM 作成 |
| VMDetailView.swift | VM 詳細・操作 |
| VMEditView.swift | 設定編集 |
| VMRunView.swift | 実行画面 |
| FramebufferView.swift | 描画・入力層、診断用ソース |
| CapabilityView.swift | 機能一覧、診断 |
| NetworkToolsView.swift | 接続テスト画面 |
| ExportPicker.swift | エクスポート |

## 構築手順（Swift Playgrounds）

1. Swift Playgrounds → 「＋」→ **App** を新規作成。
2. 既存の `ContentView.swift` を削除し、`MyApp.swift` の中身を本セットの `MyApp.swift` に置換。
3. 「＋」→ 新規 Swift ファイルを作り、上表のファイル名と同じ名前で内容を貼り付け（またはファイルを挿入）。
4. プロジェクト設定で最小 iOS を 17 以上にする。
5. ▶ で実行。

## 動作確認チェックリスト

1. 初回起動: 空の一覧と「仮想マシンがありません」が表示される
2. 新規作成: 名前を入力して「作成」→ 一覧に出る
3. ISO 読み込み: 作成画面で ISO を選択 → コピー中の進捗 → 詳細に SHA-256 が表示される
4. 仮想ディスク: 詳細の「ストレージ」に仮想サイズと実使用量が出る
5. 起動: 「起動」→ 実行画面に **エンジン未接続** のエラーが表示される（想定どおり）
6. 診断: 左上 ⓘ →「描画・入力の診断」でテストパターン、タップ座標、キー入力が反映される
7. ネットワーク: 「接続テスト」で example.com:443 が成功する
8. スナップショット: 作成 → 復元できる
9. 再起動保持: アプリを終了して再起動 → VM、ISO、スナップショットが残っている

## よくあるエラー

| 症状 | 原因 / 対処 |
|---|---|
| `ContentUnavailableView` が見つからない | 最小 iOS が 16 以下。17 以上に設定 |
| ISO で「ISO9660 の署名が見つかりません」 | ファイル破損、または UDF のみの ISO。再ダウンロード |
| 「SHA-256 が一致しません」 | 破損または別ファイル。公式サイトのチェックサムを再確認 |
| 空き容量不足 | ISO サイズ + 256 MB 以上の空きが必要 |
| 外部バッキング QCOW2 を拒否 | 安全のため仕様。単体の QCOW2 を用意 |
| 他端末の VM をインポートできない | このアプリが書き出した VM フォルダ（config.json を含む）のみ対応 |

## 調査メモ（出典）

- UTM は QEMU ベース。通常版は JIT が必要で、iOS では脱獄または回避策が必要。UTM SE は JIT なしのインタープリタ版（低速）
  - https://docs.getutm.app/installation/ios （確認日 2026-11-05）
  - https://github.com/utmapp/UTM
- Playgrounds.app はパッケージ内のバイナリ成果物や C/C++ コードに対応していないという開発者フォーラムの回答がある（古い情報のため最新版での再確認が必要）
  - https://forums.swift.org/t/spm-binary-packages-swift-playgrounds-on-ipad/62916
- UTM のライセンスは Apache 2.0 だが LGPL/GPL 部品を含む。再配布時は各ライセンスを確認すること

## 実エンジンを組み込むには（Mac + Xcode が必要）

1. macOS: `Virtualization.framework`（entitlement 必要）で ARM64 Linux を起動するバックエンドを実装
2. iPadOS: QEMU（UTM SE 方式）を XCFramework 化して `VirtualizationBackend` を実装
3. `VMStore.backendFactory` を差し替え、`FramebufferSource` にフレームを供給
