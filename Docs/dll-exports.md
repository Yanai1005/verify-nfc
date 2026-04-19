# felica.dll / rw.dll エクスポート関数一覧

`dump_dll.py` で取得した Sony FeliCa Library のエクスポート関数一覧。

DLL 所在: `C:\Program Files\Common Files\Sony Shared\FeliCaLibrary\`

---

## felica.dll (227 関数)

### ライブラリ管理
| 関数名 | 推定用途 |
|--------|---------|
| `initialize_library` | ライブラリ初期化 |
| `dispose_library` | ライブラリ終了処理 |
| `get_version_information` | バージョン情報取得 |
| `get_version_number` | バージョン番号 |
| `get_copyright_information` | 著作権情報 |

### リーダー接続
| 関数名 | 推定用途 |
|--------|---------|
| `open_reader_writer` | リーダー指定接続 |
| `open_reader_writer_auto` | 自動検出接続 |
| `open_reader_writer_without_encryption` | 暗号化なし接続 |
| `close_reader_writer` | リーダー切断 |
| `reconnect_reader_writer` | 再接続 |
| `reader_writer_is_alive` | 接続状態確認 |
| `reader_writer_is_open` | オープン状態確認 |

### ポーリング + 複合操作 (高レベルAPI)
| 関数名 | 推定用途 |
|--------|---------|
| `polling_and_get_card_information` | Polling + カード情報取得 |
| `polling_and_request_system_code` | Polling + System Code 一覧 |
| `polling_and_request_service` | Polling + サービス情報 |
| `polling_and_search_service_code` | Polling + サービス列挙 |
| `polling_and_search_service_code_2` | 同上 (別バージョン) |
| `polling_and_read_block_without_encryption` | Polling + ブロック読み取り |
| `polling_and_read_service_without_encryption` | Polling + サービス読み取り |
| `polling_and_write_block_without_encryption` | Polling + ブロック書き込み |
| `polling_and_write_service_without_encryption` | Polling + サービス書き込み |
| `polling_and_mutual_authentication` | Polling + 相互認証 |

### ブロック操作
| 関数名 | 推定用途 |
|--------|---------|
| `read_block` | 暗号化ブロック読み取り |
| `read_block_without_encryption` | 非暗号化ブロック読み取り |
| `read_service` | サービス読み取り |
| `write_block` | 暗号化ブロック書き込み |
| `write_block_without_encryption` | 非暗号化ブロック書き込み |
| `write_service` | サービス書き込み |

### 設定・状態
| 関数名 | 推定用途 |
|--------|---------|
| `get_polling_timeout` | ポーリングタイムアウト取得 |
| `set_polling_timeout` | ポーリングタイムアウト設定 |
| `get_time_out` | タイムアウト取得 |
| `set_time_out` | タイムアウト設定 |
| `get_lock_timeout` | ロックタイムアウト取得 |
| `set_lock_timeout` | ロックタイムアウト設定 |
| `get_retry_count` | リトライ回数取得 |
| `set_retry_count` | リトライ回数設定 |
| `get_number_of_cards` | カード枚数取得 |
| `get_last_card_information` | 最終カード情報 |
| `get_last_error_type` | 最終エラー種別 |
| `get_device_information` | デバイス情報 |
| `get_reader_writer_mode` | リーダーモード取得 |
| `get_sequence_number` | シーケンス番号 |
| `get_open_process_count` | オープンプロセス数 |

### 認証・鍵管理
| 関数名 | 推定用途 |
|--------|---------|
| `make_access_keys` | アクセスキー生成 |
| `make_access_keys_2` | 同上 v2 |
| `append_key_list_to_access_keys` | アクセスキーに鍵追加 |
| `exchange_key` | 鍵交換 |
| `exchange_key_for_type_2` | Type2 鍵交換 |
| `exchange_key_from_package` | パッケージから鍵交換 |
| `generate_exchange_key_package` | 鍵交換パッケージ生成 |
| `polling_and_mutual_authentication` | 相互認証 |
| `reconnect_reader_writer_and_mutual_authentication` | 再接続+認証 |
| `polling_mutual_authentication_and_read_block` | 認証+読み取り |
| `polling_mutual_authentication_and_read_service` | 認証+サービス読み取り |
| `polling_mutual_authentication_and_write_block` | 認証+書き込み |
| `polling_mutual_authentication_and_write_service` | 認証+サービス書き込み |

### 通信制御 (rw_ プレフィックス)
`rw_` プレフィックスの関数は rw.dll の機能を felica.dll 経由で呼び出すラッパー。
全 100+ 関数。代表的なもの:

| 関数名 | 推定用途 |
|--------|---------|
| `rw_initialize_library` | rw 初期化 |
| `rw_dispose_library` | rw 終了 |
| `rw_open_communications_port` | ポートオープン |
| `rw_close_communications_port` | ポートクローズ |
| `rw_polling` | Polling |
| `rw_communicate_thru` | 生コマンド送信 |
| `rw_request_system_code` | System Code 取得 |
| `rw_search_service_code` | サービス列挙 |
| `rw_read_block_without_encryption` | ブロック読み取り |
| `rw_write_block_without_encryption` | ブロック書き込み |

### その他
| 関数名 | 推定用途 |
|--------|---------|
| `start_polling` | 非同期ポーリング開始 |
| `stop_polling` | 非同期ポーリング停止 |
| `start_plug_and_play_watch` | PnP 監視開始 |
| `stop_plug_and_play_watch` | PnP 監視停止 |
| `set_call_back_parameters` | コールバック設定 |
| `set_reader_writer_mode_list` | リーダーモードリスト |
| `set_reader_writer_control_library` | 制御ライブラリ設定 |
| `transaction_lock` | トランザクションロック |
| `transaction_unlock` | ロック解除 |
| `dumb` | テスト / ダミー |

---

## rw.dll (132 関数)

rw.dll は **低レベル API** を提供する。felica.dll の `rw_*` 関数から内部的に呼ばれるが、
直接呼び出すことも可能。

### 初期化シーケンス
```
set_plugins_home_directory(dir) → initialize_plugins() → initialize_library()
```

### 主要関数
| 関数名 | 推定用途 |
|--------|---------|
| `initialize_library` | 初期化 |
| `dispose_library` | 終了 |
| `set_plugins_home_directory` | プラグインディレクトリ設定 |
| `initialize_plugins` | プラグイン初期化 |
| `open_communications_port` | ポートオープン (引数: ポート番号 0) |
| `close_communications_port` | ポートクローズ |
| `polling` | カード検出 |
| `request_service` | サービス情報取得 |
| `request_system_code` | System Code 取得 |
| `search_service_code` | サービス列挙 |
| `read_block_without_encryption` | ブロック読み取り |
| `write_block_without_encryption` | ブロック書き込み |
| `communicate_thru` | 生 FeliCa コマンド送信 |
| `disconnect` | カード切断 |
| `release` | リソース解放 |
| `enum_devices` | デバイス列挙 |

### プラグインディレクトリ
```
C:\Program Files\Common Files\Sony Shared\FeliCaLibrary\
├── plugins\
│   ├── command\
│   │   ├── basic.gp
│   │   └── mobile2.gp
│   └── device\
│       └── usb.gp
```

---

## 呼び出し規約まとめ

| DLL | cdecl (`CDLL`) | stdcall (`WinDLL`) |
|-----|----------------|-------------------|
| felica.dll | ret=1024 | ret=1024 |
| rw.dll | **ret=0 ✅** | ret=ゴミ値 ❌ |

> **rw.dll は `ctypes.CDLL` (cdecl) で呼び出すこと。**
