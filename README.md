# VMretar — Swift Playgrounds 向け VM アプリ（自作 RISC-V エンジン付き）

確認日: 2026-11-06

## 何ができるか（正直な説明）

- **RISC-V 32bit（RV32IMA）の「MMU なし Linux」を、純 Swift のインタープリタで起動する設計です。**
  JIT・C/C++ ライブラリ・特別な権限は不要で、Swift Playgrounds だけで動く構成にしてあります。
- **このコードは私の環境でコンパイルも実機起動もできていません。** 起動確認は未実施です。
  エラーや、起動途中で止まる場合は、画面のスクリーンショットや表示内容を送ってください。
- 画面はシリアルコンソール（80×30 の端末）です。グラフィックスやマウスはありません。
- ゲストは RAM（initramfs）上で動くので、**再起動すると内容は消えます**（ディスクデバイスなし）。
- Windows・MMU 付き Linux・x86・ARM64 は動きません（理由は「機能一覧」画面にも表示）。

## 設計（参考: mini-rv32ima）

mini-rv32ima は RV32IMA + Zifencei + Zicsr の小型エミュレータで、MMU なしの Linux を起動できると公開されています。
同じメモリマップを Swift で再実装しました。

| 領域 | アドレス |
|---|---|
| RAM | 0x8000_0000〜（Image を先頭に配置、DTB は末尾） |
| UART（8250 風） | 0x1000_0000（LSR は 0x1000_0005） |
| CLINT | msip 0x1100_0000 / mtimecmp 0x1100_4000 / mtime 0x1100_BFF8 |
| SYSCON | 0x1110_0000（0x5555 = 電源オフ, 0x7777 = 再起動） |

- CPU: M / U モード、割り込み（タイマー・ソフトウェア）、RV32M、RV32A（単一ハート）
- 未実装: C 拡張（圧縮命令）、S モード、MMU(Sv32)、PMP、浮動小数点
- DTB は Swift で生成（FDTBuilder.swift）。ブート引数: `earlycon=uart8250,mmio,0x10000000 console=ttyS0`
- タイマーは 1 MHz（DTB の timebase-frequency と一致）

## 追加・変更されたファイル

| ファイル | 役割 |
|---|---|
| RV32Machine.swift | CPU・メモリ・CLINT・UART・SYSCON（新規） |
| FDTBuilder.swift | デバイスツリー生成（新規） |
| TerminalEmulator.swift | 端末画面と描画（新規） |
| RV32Backend.swift | VirtualizationBackend 実装（新規） |
| VMConfiguration.swift | `riscv32` を追加、既定を RISC-V 32 / 64MB に |
| VMStore.swift | riscv32 の VM に RV32Backend を使う |
| FramebufferView.swift | ハードウェアキーボード対応を改善 |
| CreateVMView.swift / CapabilityView.swift | 説明文の更新 |

既存ファイル（VMDetailView.swift など）は変更なしです。上の変更ファイルを置き換えてください。

## 使い方

1. iPad の Safari で次の URL を開いてダウンロードします。
   `https://github.com/cnlohr/mini-rv32ima-images/raw/master/images/linux-6.1.14-rv32nommu-cnl-1.zip`
   （約 1.4 MB。mini-rv32ima 公式のビルド済み Image。確認日 2026-11-05）
2. 「ファイル」アプリで zip をタップして展開すると `Image` が出てきます。
3. アプリで「＋ → 新規 VM」。CPU は **RISC-V 32**、メモリ **64 MB**（既定）で作成。
   作成時に ISO / IMG として `Image` を選んでも、あとで詳細画面から取り込んでも構いません。
4. VM 詳細 →「起動」。実行画面にカーネルのログが流れ、シェルが出れば成功です。
5. キーボード（画面右上のキーボードボタン、またはハードウェアキーボード）で入力。
   終了はゲスト内で `poweroff`（VM が Stopped になります）。応答しないときは強制停止。

## 動作確認チェックリスト

1. 起動 → 数秒〜数十秒でログが流れる（速度は端末と最適化設定による）
2. `Linux version 6.1...` などのカーネルログが出る
3. シェルのプロンプトが出る → `ls`、`uname -a` が動く
4. 一時停止 → 再開で続きから動く
5. `poweroff` で Stopped になる
6. 強制停止 → 再度起動できる

## 起動しないとき（原因の候補）

| 症状 | 考えられる原因 / 対処 |
|---|---|
| 「カーネルイメージが未設定」 | 詳細の「ISO / IMG を取り込む」で Image を選ぶ |
| 「ELF 形式は未対応」 | zip を展開した `Image`（フラットバイナリ）を使う |
| 画面が真っ黒のまま | 数十秒待つ。変化がなければ、画面の状態を教えてください（命令の未対応や DTB の不一致が疑われます） |
| 極端に遅い | Playgrounds のビルド設定が最適化されていない可能性。メモリを 64 MB 以下にして試す |
| 途中で止まる | 未実装の命令・CSR に当たった可能性。最後に出たログを教えてください |
| 文字化け | ASCII 以外は '?' 表示になります（仕様） |

## 出典

- mini-rv32ima（Charles Lohr）: https://github.com/cnlohr/mini-rv32ima/
  RV32IMA/Zifencei/Zicsr と CLINT・MMIO を実装し、MMU なしの RV32 Linux を起動。MIT ライセンス。
  本アプリは同プロジェクトの仕様を参考に Swift で独自実装したもので、コードの転用はしていません。
- ビルド済み Image: https://github.com/cnlohr/mini-rv32ima-images （再配布条件は各リポジトリを確認してください。アプリには同梱していません）
