# VMretar — Swift Playgrounds向けRV32エミュレーター試作

確認日: 2026-11-06

## この成果物でできること

- 既存のSwiftUI VM設定画面を補完するモデル・VMStore・一覧画面
- 元の画面構成・配色・ボタン配置を維持。RV32実行画面への入口は既存「機能一覧」>「ツール」に追加
- VMごとのJSON設定保存、RAWスパースディスク作成、ISO/IMG/QCOW2の取り込み（既存実装）
- RV32IMAインタープリター（既存 `RV32Machine.swift`）を実行ループへ接続
- ELF32 little-endian / EM_RISCV の PT_LOAD セグメントをRAMへロード
- UART出力を既存TerminalFramebufferへ転送
- Pause / Resume / Force Stop
- ソフトウェア・ハードウェアキーボード入力経路

## 重要：これはUTM相当の完成VMではない

本試作はRISC-VのベアメタルELFを実行するCPUエミュレーターです。
Linuxディストリビューション、Windows、一般的なISOインストーラーを起動できる完成システムエミュレーターではありません。

現状のRV32MachineはM/U特権レベル、MMUなし、RAM、UART、CLINT、SYSCONを中心とした最小マシンです。
S-mode、Sv32、PLIC、VirtIO block/net、PCI、OpenSBIの実装が不足しているため、通常のRISC-V Linuxを起動するためのハードウェア環境がありません。
また、ISOは取り込み・保管のみで、仮想CD-ROMとしてゲストへ接続されません。RAWディスクも現時点ではゲストCPUからアクセスできません。

## 実行条件

- iPadOSのSwift Playgrounds App Playgroundを想定
- UIKit / SwiftUI / Foundation / CryptoKit / Networkを利用
- 全ファイルを同じApp Playgroundのソースに追加
- `MyApp.swift`をApp entry pointとして使用
- このソース一式はiOS/iPadOS向けUIKit実装です。macOSネイティブアプリとしてはそのままビルドできません。
- `RV32Machine.swift`はFoundationだけで型チェックできることをLinux上のSwift 6.2.1で確認しています。ただしSwiftUI/UIKitを含む全アプリのiPadOS実機ビルド・実行はこの環境では実施していません。

## ファイル構成

### 既存ファイル
- `MyApp.swift`
- `CapabilityView.swift`
- `CreateVMView.swift`
- `ExportPicker.swift`
- `FDTBuilder.swift`
- `FramebufferView.swift`
- `ImageManager.swift`
- `NetworkManager.swift`
- `NetworkToolsView.swift`
- `RV32Machine.swift`
- `StorageManager.swift`
- `TerminalEmulator.swift`

### 追加ファイル
- `VMCore.swift`: モデル、VMStore、一覧UI、容量表示
- `FramebufferSource.swift`: フレームバッファと入力の共通プロトコル
- `RV32ELFLoader.swift`: ELF32/RISC-V PT_LOADローダー
- `RV32ExecutionController.swift`: CPU実行ループと状態制御
- `RV32RunnerView.swift`: ELF選択・実行UI

## Swift Playgroundsへの追加

1. 新しい「App」プロジェクトを作成する。
2. ZIPを展開し、SwiftVMKit内のすべての `.swift` ファイルをApp Playgroundへ追加する。
3. 既存の `ContentView.swift` が `@main` を持つ場合は削除するか、`MyApp.swift`との二重エントリーポイントを解消する。
4. iPadOS向けに実行する。
5. VM一覧の情報ボタンから「機能一覧」を開き、「ツール」>「RV32 ELF エミュレーター」を選択する。
6. RISC-V 32-bit / little-endian / ELF形式のベアメタルプログラムを選択する。

## ELFの前提

- ELFCLASS32
- ELFDATA2LSB
- e_machine = EM_RISCV (243)
- PT_LOADセグメントが `0x80000000` から始まるゲストRAM範囲内
- エントリーポイントはRAM範囲内かつ4バイト境界
- 命令セットは既存RV32Machineが実装するRV32IMA + Zicsr + Zifenceiの範囲

Linux向けELFやMMU前提のユーザーランド実行を保証しません。ベアメタルのUART出力を使うテストファームウェアで確認してください。

## 既知の制約・未実装

- ISOブート / CD-ROMデバイス
- Linux用S-mode / Sv32 MMU / PLIC / OpenSBI
- 仮想ディスクのブロックデバイス接続
- ゲストNAT / DHCP / DNS / ポートフォワード
- ゲストとホストの共有フォルダ
- x86 / x86_64 / ARM64 / RV64
- グラフィカルディスプレイ / GPU / USB / サウンド
- RAM状態を含むスナップショット
- ゲストの自動再起動（SYSCON rebootは検出して停止表示）
- macOS向けVirtualization.frameworkバックエンド

## 追加実装の推奨順

1. RV32 privileged architectureのS-mode、例外委譲、Sv32 MMU
2. PLICと割り込み配線
3. VirtIO-mmio transport
4. VirtIO block + RAWディスク接続
5. OpenSBI相当のブートファームウェア、RISC-V Linux `virt`互換DTB
6. VirtIO console / input / GPU
7. VirtIO net + NAT backend
8. ELF/Kernel/Initrd/DTBの正式なブート構成
9. macOSでは別バックエンドとしてVirtualization.frameworkを追加

## 技術調査（公式・一次資料）

- Apple Virtualization framework:
  https://developer.apple.com/documentation/virtualization
- Apple Hypervisor framework:
  https://developer.apple.com/documentation/hypervisor
- Apple Swift Playgrounds:
  https://developer.apple.com/swift-playground/
- Swift PlaygroundsへのSwiftファイル追加:
  https://support.apple.com/guide/playgrounds-ipad/add-swift-files-images-and-swift-packages-itc18b7bce9d/ipados
- Swift Playgrounds Swift Package:
  https://developer.apple.com/documentation/swift-playgrounds/add-a-swift-package
- UTM architecture:
  https://github.com/utmapp/UTM/blob/main/Documentation/Architecture.md
- UTM source / license notes:
  https://github.com/utmapp/UTM
- UTM iOS install / UTM SE:
  https://docs.getutm.app/installation/ios/
- QEMU system emulation:
  https://www.qemu.org/docs/master/system/
- QEMU RISC-V system emulation:
  https://www.qemu.org/docs/master/system/target-riscv.html
- QEMU RISC-V virt machine:
  https://www.qemu.org/docs/master/system/riscv/virt.html
- QEMU license:
  https://github.com/qemu/qemu/blob/master/LICENSE
- Apple App Review Guidelines:
  https://developer.apple.com/app-store/review/guidelines/

## 技術選定上の結論

- iPadOS: JITを使わないSwiftインタープリターが安全で実装可能な出発点。速度は低い。
- macOS: LinuxゲストならVirtualization.frameworkが第一候補。異種CPUエミュレーションや多数のデバイスが必要ならQEMU TCG。
- Hypervisor.framework: macOS上でホストと同じCPUアーキテクチャの仮想化を加速する用途。異種CPUの命令エミュレーションではない。
- WebAssembly: サンドボックス内で既存エミュレーターを動かす代替案にはなるが、JIT制約、ファイルI/O、ネットワーク、パフォーマンス、WASMランタイム自体の統合が必要で、今回の自作Swift CPUより単純ではない。

## ライセンス

この成果物の追加コードは独自実装として提供します。元のアップロードファイルに含まれる著作権・ライセンス表示がある場合は保持してください。
QEMUは全体としてGPLv2を中心とするライセンス構成で、ファイルごとの例外・依存ライブラリの条件があります。QEMUコードを移植・リンク・再配布する場合は、単純に「UTMがApache-2.0だから全体もApache-2.0」と判断してはいけません。
ゲストOSのISOやファームウェアはユーザーが正規の配布元から取得し、ライセンス条件に従ってください。
