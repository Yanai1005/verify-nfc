# macOS 制限事項

Sony RC-S300 (PaSoRi 4.0) の macOS における FeliCa 操作の制限と対処法。

---

## 概要

macOS では RC-S300 は **PC/SC 経由** でのみアクセス可能。
Sony の felica.dll は Windows 専用のため、macOS では使用できない。

PC/SC 経由で可能な操作は **IDm/PMm の取得のみ** であり、
FeliCa のブロックデータ（学籍番号・氏名）は読み取れない。

---

## 動作する機能

| 操作 | APDU | 結果 |
|------|------|------|
| IDm 取得 | `FF CA 00 00 00` | ✅ 8 bytes |
| PMm 取得 | `FF CA 01 00 00` | ✅ 8 bytes |
| リーダー検出 | pyscard `get_readers()` | ✅ |

---

## 動作しない機能と理由

### FF 00 (Thru コマンド)
```
CMD: FF 00 00 00 [Lc] [FeliCa コマンド]
RSP: 6A 81 (Function not supported)
```
macOS の PC/SC ドライバ (CryptoTokenKit) が FF 00 をサポートしない。

### FF C2 (Transparent Session)
セッション自体は確立できるが、中で実行できる FeliCa コマンドが **Read Without Encryption のみ** に制限される。

```
✅ Session Start  (Tag=0x81)
✅ Switch FeliCa  (Tag=0x8F, 03 00)
✅ Read W/O Enc   (Tag=0x95, cmd=0x06) — ただし SF エラー
❌ Polling         (cmd=0x04) → 6A80
❌ Request Service (cmd=0x02) → 6A80
❌ Request SysCode (cmd=0x0C) → 6A80
❌ Search Service  (cmd=0x0A) → 6A80
```

### なぜ Read で SF エラーが出るか

macOS の PC/SC ドライバはカードを **System Code `0xFFFF`** で活性化する。
学生証データは別の System Code (例: `0x88B4`) の下にあるため、
`0xFFFF` のコンテキストから学生証サービスにアクセスすると
`SF2=0xA6` (Illegal service type) が返る。

System Code を変更するには `Polling` (0x04) コマンドが必要だが、
macOS ドライバはこれをブロックする。

---

## macOS での運用方式

### IDm のみ送信 → サーバー側マッピング

```
[学生証タッチ] → [IDm 取得] → [API 送信: {"idm": "012E4CD4544D451A"}]
                                      ↓
                               [サーバー側で IDm → 学籍番号 変換]
```

1. 初回は学生証 IDm と学籍番号のマッピングテーブルを作成
2. 以降は IDm だけで学生を特定

### 利点
- macOS で確実に動作
- リーダー側のコードがシンプル

### 欠点
- 初回マッピング登録が必要
- IDm はカードごとに固有だが、カード再発行で変わる

---

## macOS 環境構築

### devbox
```bash
devbox shell
pip install -r requirements.txt
python main.py
```

### uv (推奨)
```bash
uv venv --python 3.11
source .venv/bin/activate
uv pip install -r requirements.txt
python main.py
```

### sudo 不要
macOS では pyscard (PC/SC) が **sudo なしで** NFC リーダーにアクセスできる。
nfcpy のような USB 直接アクセスとは異なり、CryptoTokenKit 経由のため権限不要。

---

## デバッグ

### 接続確認
```bash
ioreg -p IOUSB -l | grep -i "RC-S\|Sony"
```

### PC/SC リーダー一覧
```python
from smartcard.System import readers
print(readers())
# ['SONY FeliCa Port/PaSoRi 4.0 0']
```

### debug_nfc.py
```bash
python debug_nfc.py
```
Transparent Session テスト、サービスコード探索を実行。
