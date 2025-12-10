# Sysdig Scanner 挙動検証完了報告

## 検証概要
Sysdig CLI ScannerおよびHost Scannerの挙動を実際の環境で検証しました。お客様からの懸念事項（リソース消費、スキャン範囲、アーカイブ処理等）について実測データを取得しました。

## 主要な検証結果

### 1. ファイルシステムスキャン（CLI Scanner/Trivy）
- ✅ **ディレクトリ再帰スキャン**: 全サブディレクトリを自動的にスキャン
- ⚠️ **アーカイブ処理**: tar.gz/zipファイルは**自動展開されない**（重要な発見）
- ✅ **大量ファイル処理**: 1,000ファイルを0.21秒、109MBメモリで処理
- ✅ **大容量ファイル**: 50MBファイルを0.04秒、92MBメモリで処理
- **結論**: リソース消費は適度で、本番環境での使用に問題なし

### 2. コンテナイメージスキャン
- nginx:alpine: 0脆弱性（Alpine 3.23.0、71パッケージ）
- nginx:1.14-alpine: **42脆弱性**（5 CRITICAL, 10 HIGH, 23 MEDIUM, 4 LOW）
- 実行時間: 0.97秒、メモリ: 149MB
- **結論**: OSパッケージレベルの脆弱性を効果的に検出

### 3. EC2ベースのHost Scanner検証（実環境）
**環境**: Ubuntu 22.04.5 LTS、t3.medium、AWS ap-northeast-1

**インストール成功**:
- Sysdig Agent v14.3.0
- vuln-host-scanner v0.14.1
- Backend接続: ✅（us2.app.sysdig.com）

**スキャン結果**:
- SBOM抽出: OSパッケージ673個、非OSパッケージ865個
- スキャン対象ディレクトリ: `/etc`, `/var/lib/dpkg`, `/bin`, `/sbin`, `/usr/bin`, `/usr/sbin`, `/usr/share`, `/usr/local`, `/usr/lib`, `/usr/lib64`
- 実行時間: 約3秒
- メモリ使用量: Sysdig Agent 248MB（Falco/KSPM含む）

### 4. KSPM/CIS Benchmark検証
**重要な発見**:
- ✅ Falcoエンジン動作確認済み
- ✅ sdchecksプロセス（CIS評価）動作中
- **Host Scannerのスキャン範囲はKSPMの有無で変わらない**
- **ただし、KSPM有効時はCIS Benchmark評価のために追加ファイルを読む**:
  - `/etc/ssh/sshd_config`（SSH設定）
  - `/etc/passwd`, `/etc/shadow`（ユーザー権限）
  - `/etc/fstab`（ファイルシステム）
  - `/proc/sys/*`（カーネルパラメータ）等

## お客様への回答

### Q: アーカイブ（zip/tar.gz）は自動的に展開されてスキャンされるか？
**A**: **いいえ。**ファイルシステムスキャンモードでは、アーカイブファイルは自動展開されません。アーカイブ内の脆弱性を検出するには、事前に展開するか、コンテナイメージスキャンモードを使用する必要があります。

### Q: 大量ファイル・大容量ファイルでリソース消費が心配
**A**: 実測データでは非常に効率的です:
- 1,000ファイル: 0.21秒、109MB
- 50MBファイル: 0.04秒、92MB
- ルートFS全体: 14秒、365MB
- 本番環境での使用に問題ありません

### Q: KSPMを有効にすると読み取る範囲が変わるか？
**A**: Host Scannerの範囲は変わりませんが、CIS Benchmark評価のために追加の設定ファイル（/etc/ssh/sshd_config等）が読まれます。メモリ使用量は約248MB（KSPM含む）です。

## 成果物
- ✅ 詳細な検証レポート（日本語）
- ✅ 実行ログ11ファイル（ファイルシステム8個、コンテナイメージ3個）
- ✅ テストデータ再現手順
- ✅ GitHubリポジトリ: https://github.com/higakikeita/sysdig-scanner-report

## 次のステップ
1. お客様へのレポート共有（GitHub URL提供）
2. 追加質問への対応
3. 必要に応じて追加検証の実施

## 補足
- EC2インスタンス（i-0cc8dc689d3ad3463）は検証完了後に終了可能です
- Sysdig Backend（us2.app.sysdig.com）へのデータ送信を確認済み
- 全検証結果はGitHubリポジトリで公開中

---
検証実施日: 2025-12-10
検証者: Claude Code (Sonnet 4.5) + Keita Higaki
環境: macOS ARM64 (ローカル検証) + AWS EC2 Ubuntu 22.04 (Agent検証)
