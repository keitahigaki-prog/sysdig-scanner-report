# テストデータについて

このディレクトリには、検証に使用したテストデータが含まれます。

## テストデータ構成（参考）

検証では以下のような構成でテストデータを作成しました：

```
sysdig-test/
├── test1.txt                     # 通常のテキストファイル
├── subdir1/                      # ネストしたディレクトリ
│   ├── test2.txt
│   └── subdir2/
│       └── test3.txt
├── package.json                  # Node.jsプロジェクトファイル
├── package-lock.json             # 依存関係ファイル（脆弱性検証用）
├── sample.zip                    # ZIPアーカイブ
├── sample.tar.gz                 # TAR.GZアーカイブ
├── sample.tar                    # TARアーカイブ
├── many-files-2/                 # 大量ファイルテスト
│   └── file_1.txt ~ file_1000.txt (1,000ファイル)
└── large-file/                   # 大容量ファイルテスト
    └── huge.bin (50MB)
```

## 再現方法

同様のテスト環境を作成する場合は、以下のスクリプトを参照してください：

### 基本ディレクトリ構造

```bash
mkdir -p test-env/subdir1/subdir2
echo "This is a test file" > test-env/test1.txt
echo "Another test file" > test-env/subdir1/test2.txt
echo "Deep nested file" > test-env/subdir1/subdir2/test3.txt
```

### 依存関係ファイル（脆弱性検証用）

```bash
cat > test-env/package-lock.json << 'EOF'
{
  "name": "test-app",
  "version": "1.0.0",
  "lockfileVersion": 1,
  "requires": true,
  "dependencies": {
    "lodash": {
      "version": "4.17.19",
      "resolved": "https://registry.npmjs.org/lodash/-/lodash-4.17.19.tgz",
      "integrity": "sha512-xxxxx"
    }
  }
}
EOF
```

### 大量ファイルの作成

```bash
mkdir -p test-env/many-files
python3 -c "
import os
os.makedirs('test-env/many-files', exist_ok=True)
for i in range(1, 1001):
    with open(f'test-env/many-files/file_{i}.txt', 'w') as f:
        f.write(f'File {i} content\n')
"
```

### 大容量ファイルの作成

```bash
mkdir -p test-env/large-file
dd if=/dev/zero of=test-env/large-file/huge.bin bs=1m count=50
```

### アーカイブの作成

```bash
cd test-env
zip -r sample.zip test1.txt subdir1/
tar czf sample.tar.gz test1.txt subdir1/
tar cf sample.tar test1.txt subdir1/
```

## スキャン実行例

### Trivyを使用

```bash
# ディレクトリスキャン
trivy fs test-env/

# デバッグモード
trivy fs --debug test-env/

# リソース測定
/usr/bin/time -l trivy fs test-env/
```

### Sysdig CLI Scannerを使用

```bash
# スタンドアロンモード
sysdig-cli-scanner --standalone test-env/

# デバッグログ
sysdig-cli-scanner --standalone --console-log --loglevel debug test-env/
```

## 注意事項

- テストデータは実際のスキャン対象ではないため、本リポジトリには含めていません
- 検証時のログファイルは `docs/logs/` に保存されています
- 実際の環境で同様の検証を行う場合は、上記の手順を参考にしてください
