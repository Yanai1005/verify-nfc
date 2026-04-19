# FeliCa 学生証読み取り 技術調査ログ

Sony RC-S300 (PaSoRi 4.0) で FeliCaを読み取るための調査記録。

---

## 目次

1. [プロジェクト概要](#1-プロジェクト概要)
2. [ハードウェア情報](#2-ハードウェア情報)
3. [Phase 1: nfcpy での試行と挫折](#3-phase-1-nfcpy-での試行と挫折)
4. [Phase 2: pyscard (PC/SC) への移行](#4-phase-2-pyscard-pcsc-への移行)
5. [Phase 3: macOS での FeliCa ブロック読み取り調査](#5-phase-3-macos-での-felica-ブロック読み取り調査)
6. [Phase 4: Transparent Session の発見と限界](#6-phase-4-transparent-session-の発見と限界)
7. [Phase 5: サービスコード総当たり探索](#7-phase-5-サービスコード総当たり探索)
8. [Phase 6: Windows PC/SC の調査](#8-phase-6-windows-pcsc-の調査)
9. [Phase 7: Sony FeliCa Library (felica.dll) への移行](#9-phase-7-sony-felica-library-felicadll-への移行)
10. [Phase 8: rw.dll 直接呼び出し](#10-phase-8-rwdll-直接呼び出し)
11. [現在の状態と次のステップ](#11-現在の状態と次のステップ)
12. [技術リファレンス](#12-技術リファレンス)

---

## 1. プロジェクト概要

### 目的
FeliCa 学生証を NFC リーダーで読み取り、出席データ (学籍番号 + 氏名) を API サーバーへ送信する。

### 環境
- **リーダー**: Sony RC-S300/P (PaSoRi 4.0)
- **開発環境**: devbox + Python 3.11
- **OS**: macOS (メイン開発) + Windows 11 (FeliCa ブロック読み取り用)
- **パッケージ管理**: uv (推奨) / devbox

### 参考記事
- [立命館の学生証の話 (cysec.ise.ritsumei.ac.jp)](https://cysec.ise.ritsumei.ac.jp/2022/06/13/%E7%AB%8B%E5%91%BD%E9%A4%A8%E3%81%AE%E5%AD%A6%E7%94%9F%E8%A8%BC%E3%81%AE%E8%A9%B1/)
  - サービスコード `0x1A8B` (num=106, attr=0x0B) に学生データ
  - Block 0: 学籍番号 (UTF-8/ASCII)
  - Block 1: 氏名 (Shift-JIS 半角カナ)

---

## 2. ハードウェア情報

### RC-S300/P スペック
| 項目 | 値 |
|------|-----|
| USB Vendor ID | `054C` (Sony) |
| USB Product ID | `0DC9` |
| USB Product Name | RC-S380/S (※内部名称) |
| PC/SC リーダー名 | SONY FeliCa Port/PaSoRi 4.0 |
| ファームウェア | 1.00 |

### カード情報 (テスト用学生証)
| 項目 | 値 |
|------|-----|
| IDm | `01 2E 4C D4 54 4D 45 1A` |
| PMm | `03 32 42 82 82 47 AA FF` |
| ATR | `3B 8F 80 01 80 4F 0C A0 00 00 03 06 11 00 3B 00 00 00 00 42` |
| カード種別 | FeliCa |

### Windows ドライバ構成 (2つのインターフェース)

RC-S300 は Windows 上で **2つのインターフェース** を持つ:

| No. | インターフェース | ドライバ | 機能 |
|-----|-----------------|---------|------|
| 1 | RC-S300/P (PC/SC) | v1.1.5.1 | 制限付き FeliCa (IDm のみ) |
| 2 | RC-S300/P WinUSB | v1.0.4.0 | フル FeliCa アクセス (felica.dll 経由) |

---

## 3. Phase 1: nfcpy での試行と挫折

### 当初の計画
```
nfcpy==1.0.4 + pyusb → USB 直接制御 → FeliCa ブロック読み取り
```

### 結果: ❌ 失敗

**原因**: nfcpy 1.0.4 は RC-S300 (PID `0x0DC9`) を **サポートしていない**。

nfcpy がサポートする PID:
- `0x0193` — PN531
- `0x02E1` — RC-S330/360/370
- `0x06C1`, `0x06C3` — RC-S380

GitHub Issues #214, #240 でも RC-S300 非対応が報告されている。

### macOS 追加問題
- SIP が `DYLD_LIBRARY_PATH` を除去 → pyusb が libusb バックエンドを見つけられない
- devbox の nix store から libusb シンボリックリンクを作成して回避

### 教訓
> RC-S300 と RC-S380 は USB 上の **異なるデバイス**。名前が似ているが互換性なし。

---

## 4. Phase 2: pyscard (PC/SC) への移行

### 選定理由
PC/SC は OS 標準の NFC/スマートカードインターフェースで、ドライバが対応していれば追加ソフト不要。

### インストール
```bash
uv pip install pyscard
```

### 成功した操作

#### IDm 取得 (`FF CA 00 00 00`)
```
CMD: FF CA 00 00 00
SW=9000
Data: 01 2E 4C D4 54 4D 45 1A (8 bytes = IDm)
```

#### PMm 取得 (`FF CA 01 00 00`)
```
CMD: FF CA 01 00 00
SW=9000
Data: 03 32 42 82 82 47 AA FF (8 bytes = PMm)
```

### 結果: ✅ IDm/PMm の取得は全 OS で動作

---

## 5. Phase 3: macOS での FeliCa ブロック読み取り調査

### 試行した APDU コマンド一覧

| 方式 | APDU | 結果 | 備考 |
|------|------|------|------|
| Thru コマンド | `FF 00 00 00 [Lc] [FeliCa cmd]` | ❌ `6A81` | Function not supported |
| Transparent Session | `FF C2 00 00 02 81 00` | ✅ `9000` | Session Start 成功 |
| Switch Protocol | `FF C2 00 02 04 8F 02 03 00` | ✅ `9000` | FeliCa モード切替 |
| Read (P2=00) | `FF C2 00 00 [Lc] 95 ...` | ❌ `6A81` | P2=00 は不可 |
| Read (P2=01) | `FF C2 00 01 [Lc] 95 ...` | ✅ `9000` | **カード応答あり** |
| Read (P2=02) | `FF C2 00 02 ...` | ❌ `0x8010000C` | 接続切断 |
| CCID Escape | `CTL=0x42000001` | ❌ `0x80100004` | macOS 非対応 |
| ReadBinary | `FF B0 00 00 10` | ❌ `6A81` | |
| SCardControl | 各種 IOCTL | ❌ | |
| Direct Mode | `SCARD_SHARE_DIRECT` | ❌ `0x80100011` | macOS で INVALID_VALUE |

---

## 6. Phase 4: Transparent Session の発見と限界

### 発見: PC/SC Part 3 Transparent Session が動作

**参考**: [Zenn 記事](https://zenn.dev/compass/articles/39bb050bdaeaaa)、dekirukigasuru blog

#### 動作シーケンス
```
1. Session Start:  FF C2 00 00 02 81 00 00 → 9000 (C0 03 00 90 00)
2. Switch FeliCa:  FF C2 00 02 04 8F 02 03 00 00 → 9000
3. Read (TLV):     FF C2 00 01 [Lc] 95 [len] [FeliCa Read cmd] 00 → 9000
4. Session End:    FF C2 00 00 02 82 00 00 → 9000
```

#### Read コマンドの構造 (Tag=95)
```
FF C2 00 01 12   ← CLA INS P1=00 P2=01 Lc=18
  95 10          ← TLV Tag=0x95, Length=16
    10           ← FeliCa cmd length (16)
    06           ← Read Without Encryption (cmd=0x06)
    [IDm 8bytes] ← カードの IDm
    01           ← サービス数=1
    8B 1A        ← サービスコード 0x1A8B (LE)
    01           ← ブロック数=1
    80 00        ← ブロックリスト (2byte: 0x80 | block_num)
  00             ← Le
```

#### レスポンス TLV 構造
```
C0 03 00 90 00   ← Tag=C0 (ステータス): inner SW=9000
92 01 00         ← Tag=92: 不明
96 02 00 00      ← Tag=96: 不明
97 0C            ← Tag=97 (FeliCa レスポンス): Length=12
  0C             ← FeliCa レスポンス長
  07             ← Read レスポンスコード (0x07)
  [IDm 8bytes]   ← カードの IDm
  01             ← SF1 (Status Flag 1) ← 0x01 = エラー
  A6             ← SF2 (Status Flag 2) ← 0xA6 = Illegal service type
```

### 結果: ⚠ カードは応答するが SF2=A6 (Illegal service type)

### 根本原因
macOS の PC/SC ドライバは **System Code `0xFFFF`** でカードを活性化する。
学生証データは別の System Code (例: `0x88B4`) の下にあるため、
`0xFFFF` で活性化されたセッションからは到達できない。

### macOS ドライバがブロックする FeliCa コマンド
| FeliCa コマンド | コード | 結果 |
|----------------|--------|------|
| Polling | `0x04` | ❌ `6A80` |
| Request Service | `0x02` | ❌ `6A80` |
| Request System Code | `0x0C` | ❌ `6A80` |
| Search Service Code | `0x0A` | ❌ `6A80` |
| Read Without Encryption | `0x06` | ✅ 通過 (但し SF エラー) |

> **結論**: macOS PC/SC では FeliCa の System Code を変更・取得する手段がない。

---

## 7. Phase 5: サービスコード総当たり探索

### 目的
System Code 0xFFFF でアクセスできるサービスコードが存在するか確認。

### 試行範囲
- `attr=0x0B` (Read Only): num 0〜300 (サービスコード 0x000B, 0x004B, 0x008B, ...)
- `attr=0x09` (Read/Write): num 0〜100
- 特定コード: `0x000B`, `0x0009`, `0x008B`, `0x010B`, `0x048B`, `0x1A8B`, `0x100B`, `0x200B` 他

### 結果: ❌ 全て失敗

全てのサービスコードが以下のいずれかを返した:
- `C0_err=01 6A 80` (ドライバがブロック)
- `SF1=01 SF2=A6` (Illegal service type — System Code 不一致)
- `connect_err: 0x8010000C` (50回程度で接続切断)

### 結論
> macOS PC/SC では学生証のブロックデータを読み取る方法がない。
> IDm のみの取得に限定し、サーバー側でマッピングする方式に切り替え。

---

## 8. Phase 6: Windows PC/SC の調査

### 期待
Windows の PC/SC ドライバなら FeliCa コマンドがブロックされないかもしれない。

### 結果: ❌ 同じ制限

| コマンド | Windows 結果 |
|---------|-------------|
| `FF CA` (IDm) | ✅ 動作 |
| `FF 00` (Thru) | ❌ エラー |
| `FF C2` (Transparent) | ❌ `0x00000016` (デバイスがコマンドを認識できません) |

> Windows の PC/SC インターフェースも FeliCa パススルーをサポートしない。

### 転機: Sony NFC Port Software 自己診断

ユーザーが Sony NFC ポート自己診断ツールを実行したところ、**2つのインターフェース** が確認された:

```
No.1: RC-S300/P (PC/SC)
  ICカードとの通信(PC/SC): OK
  → 制限付き (IDm のみ)

No.2: RC-S300/P WinUSB
  ICカードとの通信(WinUSB): OK
  → フル FeliCa アクセス
```

```
FeliCa Library Status
  FeliCa Lib: initialize OK, open OK, close OK, dispose OK
```

> **発見**: RC-S300 は PC/SC とは別に **WinUSB インターフェース** を持ち、
> Sony の `felica.dll` 経由でフル FeliCa 操作が可能。

---

## 9. Phase 7: Sony FeliCa Library (felica.dll) への移行

### DLL 情報

```
場所: C:\Program Files\Common Files\Sony Shared\FeliCaLibrary\
felica.dll: v2.5.0.1 (2015/02/17) — 高レベルAPI (227関数)
rw.dll:     v2.5.0.1 (2015/02/17) — 低レベルAPI (132関数)
```

### DLL エクスポート調査 (dump_dll.py)

`dump_dll.py` を作成して PE ファイルのエクスポートテーブルを読み取り、
全227関数名を特定。主要な関数:

#### felica.dll 主要関数 (高レベルAPI)
| 関数名 | 説明 |
|--------|------|
| `initialize_library` | ライブラリ初期化 |
| `dispose_library` | ライブラリ終了 |
| `open_reader_writer_auto` | リーダー自動検出・接続 |
| `close_reader_writer` | リーダー切断 |
| `polling_and_get_card_information` | Polling + カード情報取得 |
| `polling_and_request_system_code` | Polling + System Code 取得 |
| `polling_and_search_service_code` | Polling + サービス列挙 |
| `polling_and_read_block_without_encryption` | Polling + ブロック読み取り |
| `polling_and_read_service_without_encryption` | Polling + サービス読み取り |
| `rw_communicate_thru` | 生の FeliCa コマンド送信 |

#### rw.dll 主要関数 (低レベルAPI)
| 関数名 | 説明 |
|--------|------|
| `initialize_library` | 初期化 |
| `dispose_library` | 終了 |
| `open_communications_port` | ポートオープン |
| `close_communications_port` | ポートクローズ |
| `set_plugins_home_directory` | プラグインディレクトリ設定 |
| `initialize_plugins` | プラグイン初期化 |
| `polling` | カード検出 |
| `request_system_code` | System Code 取得 |
| `search_service_code` | サービス列挙 |
| `read_block_without_encryption` | ブロック読み取り |
| `communicate_thru` | 生コマンド送信 |

### felica.dll 呼び出し試行

#### 試行 1: felica.dll + stdcall
```python
dll = ctypes.WinDLL(felica_path)
ret = dll.initialize_library()  # → 1024 (0x400)
```
**結果**: 1024 を返す。成功 (0) ではない。

#### 試行 2: felica.dll + cdecl
```python
dll = ctypes.CDLL(felica_path)
ret = dll.initialize_library()  # → 1024 (0x400)
```
**結果**: 同じく 1024。

#### 試行 3: initialize_library → open_reader_writer_auto (ret 無視で続行)
```python
ret = dll.initialize_library()  # → 1024 (無視)
port = ctypes.c_void_p(0)
ret2 = dll.open_reader_writer_auto(ctypes.byref(port))  # → ?
```
**現状**: テスト中。1024 が "既に初期化済み" の意味ならば動作する可能性。

---

## 10. Phase 8: rw.dll 直接呼び出し

### 初回成功

felica.dll を先にロードし、その後 rw.dll (cdecl) で初期化が成功:
```
rw.dll (cdecl): set_plugins_home_directory OK
rw.dll (cdecl): initialize_plugins OK
rw.dll (cdecl): initialize_library -> 0         ← ✅ 成功!
open_communications_port(0) -> 0                 ← ✅ 成功!
```

### polling() の問題

リーダー接続は成功したが、`polling()` 関数の引数シグネチャが不明:
```
IDm: 00 00 00 00 00 00 00 00  ← 全ゼロ (引数の順序が違う)
```

試行したパターン:
- `A: (sys, req, ts, &num, idm, pmm)` → IDm 全ゼロ
- `B: (sys, req, ts, idm, pmm, &num)` → IDm 全ゼロ
- `C: (sys, req, ts, idm, pmm)` → IDm 全ゼロ
- `D: (sys, idm, pmm)` → IDm 全ゼロ
- `E: (sys, req, ts, &num, pmm, idm)` → IDm 全ゼロ

### 再現性の問題

2回目以降の実行で `initialize_library` が **0 を返さなくなる** 問題が発生:
```
initialize_library -> -525022208   ← ゴミ値
```

**原因推定**:
- 前回の Python プロセスが `dispose_library` を呼ばずに終了した
- DLL がプロセス空間に残存し、内部状態が壊れた
- USB デバイスの抜き差しで回復する場合がある

### 現在のアプローチ: communicate_thru

`polling()` の引数が不明なため、`communicate_thru` で **生の FeliCa コマンド** を送信する方式に切り替え中:
```python
# FeliCa Polling コマンド (プロトコルレベル)
polling_cmd = bytes([0x06, 0x04, 0xFF, 0xFF, 0x01, 0x0F])
# Length=6, Code=Polling, SysCode=FFFF, ReqCode=01, TimeSlot=0F

# communicate_thru の引数パターンを4通り試行
# A: (cmd_len, cmd_buf, &resp_len, resp_buf)
# B: (cmd_buf, cmd_len, resp_buf, &resp_len)
# C: (cmd_buf, cmd_len, &resp_len, resp_buf)
# D: (cmd_len, cmd_buf, resp_buf, &resp_len)
```

---

## 11. 現在の状態と次のステップ

### 完了事項 ✅

| 項目 | 状態 |
|------|------|
| devbox 環境 (Python 3.11 + pyscard) | ✅ |
| IDm/PMm 取得 (全 OS) | ✅ |
| macOS PC/SC 制約の完全な文書化 | ✅ |
| reader.py (Thru + Transparent + IDm フォールバック) | ✅ |
| main.py + sender.py (デュアルモード) | ✅ |
| config.py (.env ベース設定) | ✅ |
| README.md (uv / devbox セットアップ) | ✅ |
| dump_dll.py (felica.dll 関数名特定) | ✅ |
| discover.py (探索スクリプト骨格) | ✅ |

### 未完了 🔧

| 項目 | ブロッカー |
|------|-----------|
| felica.dll / rw.dll の初期化安定化 | DLL 状態管理、プロセス間競合 |
| polling / communicate_thru の引数特定 | SDK 非公開のためリバースエンジニアリング |
| 学生証ブロックデータの読み取り | 上記2つに依存 |
| reader.py への Windows felica.dll 統合 | ブロックデータ読み取り成功後 |
| エンドツーエンドテスト | 上記全てに依存 |

### 次のステップ

1. **RC-S300 を抜き差しして** `discover.py` を再実行
   - `felica.dll` の `open_reader_writer_auto` が成功するか確認
   - `rw.dll` の `communicate_thru` でカード応答があるか確認

2. 成功パターンが見つかったら:
   - Request System Code → サービスコード列挙 → ブロック読み取り
   - 学生証のデータ構造を特定

3. `reader.py` に Windows 用 felica.dll パスを追加

---

## 12. 技術リファレンス

### PC/SC APDU コマンド

| コマンド | CLA | INS | P1 | P2 | 説明 |
|---------|-----|-----|----|----|------|
| Get Data (IDm) | FF | CA | 00 | 00 | FeliCa IDm 取得 |
| Get Data (PMm) | FF | CA | 01 | 00 | FeliCa PMm 取得 |
| Thru | FF | 00 | 00 | 00 | FeliCa パススルー |
| Transparent | FF | C2 | P1 | P2 | PC/SC Part 3 透過制御 |

### PC/SC Transparent Session TLV タグ

| Tag | 名前 | 説明 |
|-----|------|------|
| 0x81 | Session Start | セッション開始 |
| 0x82 | Session End | セッション終了 |
| 0x8F | Switch Protocol | プロトコル切替 (03 00 = FeliCa) |
| 0x95 | Transparent Exchange | FeliCa コマンド送信 |
| 0xC0 | Status | レスポンスステータス |
| 0x97 | Data | FeliCa レスポンスデータ |

### FeliCa コマンドコード

| コード | 名前 | 応答コード |
|--------|------|-----------|
| 0x04 | Polling | 0x05 |
| 0x06 | Read Without Encryption | 0x07 |
| 0x08 | Write Without Encryption | 0x09 |
| 0x0A | Search Service Code | 0x0B |
| 0x0C | Request System Code | 0x0D |
| 0x02 | Request Service | 0x03 |

### FeliCa Status Flag エラーコード

| SF1 | SF2 | 意味 |
|-----|-----|------|
| 0x00 | 0x00 | 成功 |
| 0x01 | 0xA6 | Illegal service type (System Code 不一致) |
| 0x01 | 0xA1 | Access denied |
| 0xFF | 0xFF | Internal error |

### felica.dll エラーコード (推定)

| 戻り値 | 16進 | 推定意味 |
|--------|------|---------|
| 0 | 0x00 | 成功 |
| 1024 | 0x400 | 既に初期化済み / 警告 |
| -525022208 | 0xE0B80000 | 内部エラー (DLL 状態不整合) |
| -669416448 | 0xD8280000 | 内部エラー |
| -470060032 | 0xE3FC0000 | 呼び出し規約不一致 (stdcall) |

### OS 別対応状況

| 機能 | macOS | Windows (PC/SC) | Windows (felica.dll) |
|------|-------|----------------|---------------------|
| IDm 取得 | ✅ | ✅ | 🔧 調査中 |
| PMm 取得 | ✅ | ✅ | 🔧 調査中 |
| System Code 取得 | ❌ | ❌ | 🔧 調査中 |
| サービス列挙 | ❌ | ❌ | 🔧 調査中 |
| ブロック読み取り | ❌ | ❌ | 🔧 調査中 |
| Polling | ❌ | ❌ | 🔧 調査中 |

---

## ファイル構成

```
nfc-reader/
├── main.py          # エントリーポイント (PC/SC ポーリングループ)
├── reader.py        # FeliCa 読み取り (Thru/Transparent/IDm フォールバック)
├── sender.py        # API 送信 (POST JSON)
├── config.py        # 設定読み込み (.env)
├── discover.py      # サービスコード探索 (Windows: felica.dll, macOS: PC/SC)
├── debug_nfc.py     # PC/SC 接続診断・サービスコード総当たり
├── dump_dll.py      # felica.dll / rw.dll エクスポート関数一覧
├── requirements.txt # pyscard, requests, jaconv, python-dotenv
├── devbox.json      # devbox 環境定義
├── .env.example     # 設定テンプレート
├── README.md        # セットアップ・使い方
└── Docs/
    └── investigation-log.md  # この文書
```
