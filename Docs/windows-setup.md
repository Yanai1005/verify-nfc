# Windows セットアップガイド

Sony RC-S300 + felica.dll を使用した FeliCa 学生証読み取りの Windows 環境構築手順。

---

## 前提条件

- Windows 10/11 (x64)
- Sony RC-S300/P (PaSoRi 4.0) USB 接続
- [Sony NFC Port Software](https://www.sony.co.jp/Products/felica/consumer/support/download/) インストール済み
- Python 3.11

---

## 1. Sony NFC Port Software の確認

### インストール確認
```
C:\Program Files\Common Files\Sony Shared\FeliCaLibrary\felica.dll
C:\Program Files\Common Files\Sony Shared\FeliCaLibrary\rw.dll
```

上記ファイルが存在することを確認。

### 自己診断ツール
NFC Port Software に付属の自己診断ツールで以下を確認:
- ICカードとの通信 (PC/SC): **OK**
- ICカードとの通信 (WinUSB): **OK**
- FeliCa Lib: **initialize OK, open OK, close OK, dispose OK**

---

## 2. Python 環境構築 (uv 推奨)

### uv のインストール
```powershell
# PowerShell
irm https://astral.sh/uv/install.ps1 | iex
```

### プロジェクトセットアップ
```powershell
cd C:\Users\<ユーザー名>\Desktop\workspace\verify-nfc-reader

# venv 作成 + 依存インストール
uv venv --python 3.11
.venv\Scripts\activate
uv pip install -r requirements.txt
```

### 必要パッケージ (requirements.txt)
```
pyscard>=2.0.0
requests>=2.31.0
jaconv>=0.3.4
python-dotenv>=1.0.0
```

---

## 3. 動作確認

### Step 1: DLL エクスポート確認
```powershell
python dump_dll.py
```
`felica.dll` (227 関数) と `rw.dll` (132 関数) のエクスポート一覧が表示されれば OK。

### Step 2: サービスコード探索
```powershell
# RC-S300 を USB に接続した状態で:
python discover.py
```

### Step 3: IDm 読み取り (PC/SC)
```powershell
python main.py
```

---

## 4. トラブルシューティング

### `initialize_library` が 0 以外を返す

**症状**: `initialize_library -> 1024` または大きな負数

**対処**:
1. USB ケーブルを抜き差し
2. タスクマネージャーで Python プロセスを終了
3. Smart Card サービスの再起動:
   ```powershell
   net stop SCardSvr
   net start SCardSvr
   ```

### `function 'xxx' not found`

**症状**: ctypes が DLL 関数を見つけられない

**原因**: 関数名が間違っている。正しい名前は `dump_dll.py` で確認。
- ❌ `initialize` → ✅ `initialize_library`
- ❌ `dispose` → ✅ `dispose_library`

### 32bit/64bit 問題

**症状**: DLL 呼び出しでアクセス違反やゴミ値

**対処**: 
- `felica.dll` の bitness と Python の bitness を一致させる
- 64bit Windows の場合:
  - `C:\Program Files\Common Files\...` → 64bit DLL
  - `C:\Program Files (x86)\Common Files\...` → 32bit DLL
- `python -c "import struct; print(struct.calcsize('P') * 8)"` で確認

### デバイスがコマンドを認識できません (0x00000016)

**症状**: PC/SC 経由の Transparent Session コマンドが失敗

**原因**: RC-S300 の PC/SC ドライバは FF C2 (Transparent) を**サポートしない**。
これは仕様上の制限であり、felica.dll 経由でのアクセスが必要。

---

## 5. DLL 呼び出し規約

| DLL | 呼び出し規約 | Python |
|-----|-------------|--------|
| felica.dll | 未確定 (cdecl/stdcall とも 1024) | `ctypes.CDLL` or `ctypes.WinDLL` |
| rw.dll | **cdecl** | `ctypes.CDLL` |

> rw.dll は `ctypes.CDLL` (cdecl) で初期化に成功 (ret=0)。
> `ctypes.WinDLL` (stdcall) ではゴミ値を返す。

---

## 6. rw.dll 初期化シーケンス

```python
import ctypes, os

rw_path = r"C:\Program Files\Common Files\Sony Shared\FeliCaLibrary\rw.dll"
plugins_dir = r"C:\Program Files\Common Files\Sony Shared\FeliCaLibrary"

dll = ctypes.CDLL(rw_path)

# 1. プラグインディレクトリ設定
buf = ctypes.create_string_buffer(plugins_dir.encode("ascii"))
ret = dll.set_plugins_home_directory(buf)
print(f"set_plugins_home_directory: {ret}")  # 期待: 1

# 2. プラグイン初期化
ret = dll.initialize_plugins()
print(f"initialize_plugins: {ret}")  # 期待: 0 または正の値

# 3. ライブラリ初期化
ret = dll.initialize_library()
print(f"initialize_library: {ret}")  # 期待: 0

# 4. ポートオープン
ret = dll.open_communications_port(0)
print(f"open_communications_port: {ret}")  # 期待: 0

# ... FeliCa 操作 ...

# 終了処理
dll.close_communications_port()
dll.dispose_library()
```
