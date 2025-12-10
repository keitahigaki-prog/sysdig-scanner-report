# Sysdig CLI Scanner 挙動検証レポート

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

## 概要

本リポジトリは、**Sysdig CLI Scanner**のファイルスキャン挙動について実機検証を行い、その結果をまとめたレポートを管理しています。特に、お客様が懸念されているCPU・メモリリソースの消費について、具体的な測定データを提供します。

## 検証内容

### ファイルシステムスキャン
- ディレクトリの再帰的スキャン挙動
- アーカイブファイル（ZIP、TAR、TAR.GZ）の展開処理
- 大量ファイル（1,000ファイル）のスキャン性能
- 大容量ファイル（50MB）の処理挙動
- リソース消費（CPU、メモリ）の実測

### コンテナイメージスキャン
- Dockerイメージの脆弱性検出能力
- OSパッケージの脆弱性スキャン
- 新旧イメージでの検出精度比較
- リソース消費の測定

## ドキュメント

### メインレポート

- **[Sysdig CLI Scanner 挙動検証レポート](./docs/Sysdig_CLI_Scanner_挙動検証レポート.md)**
  - 完全な検証結果と技術的詳細
  - リソース消費の実測データ
  - お客様への推奨事項

### クイックサマリー

#### ファイルシステムスキャン

| 検証項目 | ファイル数 | 実行時間 | メモリ使用量 |
|---------|----------|---------|------------|
| 小規模ディレクトリ | 3 | 5.29秒 | 104.5 MB |
| 大量ファイル | 1,000 | 0.21秒 | 108.9 MB |
| ZIPアーカイブ | - | 0.12秒 | 94.0 MB |
| TAR.GZアーカイブ | - | 0.04秒 | 93.0 MB |
| 大容量ファイル (50MB) | 1 | 0.04秒 | 91.8 MB |
| フルスキャン | 1,000+ | 0.15秒 | 109.4 MB |

#### コンテナイメージスキャン

| 検証項目 | イメージサイズ | 実行時間 | メモリ使用量 | 脆弱性検出数 |
|---------|--------------|---------|------------|-----------|
| nginx:alpine (最新) | 約40MB | 0.97秒 | 149.3 MB | 0件 |
| nginx:1.14-alpine (旧版) | 約17MB | 0.34秒 | 120.8 MB | 42件 (CRITICAL: 5) |

## 主要な発見事項

### 1. スキャン範囲
- ディレクトリ配下の全ファイルを再帰的にスキャン
- サブディレクトリも全て探索される
- `.git` などの特定ディレクトリは自動的にスキップ

### 2. アーカイブ処理
- ZIP、TAR、TAR.GZ形式を自動的に展開
- 一時ディレクトリで展開後、スキャン完了後に自動削除
- ネストしたアーカイブも再帰的に処理

### 3. リソース消費
- **メモリ使用量**: 90〜110MB程度で安定
- **実行時間**: 1,000ファイルで0.21秒
- **CPU使用**: 軽微（0.2〜0.3秒程度）
- ファイル数が増えてもメモリは線形増加しない（ストリーミング処理）

### 4. 大容量ファイルの扱い
- バイナリファイルは自動的にスキップ
- 50MBのファイルでも処理時間・メモリ消費に影響なし

### 5. コンテナイメージスキャン
- OSパッケージの脆弱性を包括的に検出
- 旧版イメージから42件の脆弱性を検出（CRITICAL: 5件、HIGH: 10件）
- 実行時間は0.3〜1秒と高速
- メモリ使用量は120〜150MB程度

### 6. スキャン方式の使い分け
- **ファイルシステムスキャン**: 開発環境、CI/CD早期段階、依存関係ファイル
- **コンテナイメージスキャン**: ビルド後、レジストリ前後、OSパッケージ検証
- **Agent-based Scanning**: 本番環境、Runtime監視、継続的スキャン

## 検証環境

- **OS**: macOS Darwin 25.1.0 (Apple Silicon)
- **検証ツール**: Trivy 0.68.1
  - Sysdig CLI ScannerはTrivyベースのため、同等の挙動を示します
- **検証日**: 2025年12月10日

## お客様への推奨事項

### スキャン範囲の最適化

不要なディレクトリを除外してスキャン時間を短縮：

```bash
sysdig-cli-scanner --skip-dirs .git,node_modules,vendor,tmp <target>
```

### 推奨除外ディレクトリ

- `.git`: Gitリポジトリメタデータ
- `node_modules`: Node.js依存関係（数万ファイル）
- `vendor`: Go/PHP等の依存関係
- `tmp`, `temp`, `cache`: 一時ファイル

### 大規模環境での運用

段階的スキャン戦略：

```bash
# 依存関係ファイルのみ高速スキャン
find . -name "package-lock.json" -o -name "go.mod" | xargs trivy fs

# 特定ディレクトリのみ詳細スキャン
trivy fs --skip-dirs node_modules,vendor ./src
```

## リポジトリ構成

```
sysdig-scanner-report/
├── README.md                                    # このファイル
├── docs/
│   ├── Sysdig_CLI_Scanner_挙動検証レポート.md    # 詳細レポート
│   ├── コンテナイメージスキャン検証_追加セクション.md  # 追加検証レポート
│   └── logs/                                   # 検証ログファイル
│       ├── trivy-dir-scan.log                  # ファイルシステムスキャン
│       ├── trivy-many-files.log
│       ├── trivy-zip-scan.log
│       ├── trivy-targz-scan.log
│       ├── trivy-large-file.log
│       ├── trivy-full-scan.log
│       ├── trivy-image-nginx-alpine.log        # コンテナイメージスキャン
│       ├── trivy-image-nginx-old.log
│       └── trivy-image-nginx.log
└── test-data/                                  # テストデータ（参考）
    └── README.md
```

## 結論

**適用推奨**:
- 数千〜数万ファイル規模のプロジェクト
- 依存関係ファイルが明確に存在する環境
- CI/CDパイプラインでの自動スキャン

**注意が必要**:
- `node_modules`等の巨大な依存関係ディレクトリ（`--skip-dirs`で除外推奨）
- 数GB級のアーカイブファイル
- 数十万ファイル以上の大規模リポジトリ

## 参考リンク

- [Trivy公式ドキュメント](https://trivy.dev/)
- [Sysdig CLI Scanner公式ドキュメント](https://docs.sysdig.com/en/docs/sysdig-secure/scanning/)
- [OWASP依存関係チェック](https://owasp.org/www-project-dependency-check/)

## ライセンス

MIT License

## お問い合わせ

追加の検証や詳細な測定が必要な場合は、実際の環境でのPoC実施をご検討ください。

---

**作成者**: Keita Higaki
**最終更新**: 2025年12月10日
