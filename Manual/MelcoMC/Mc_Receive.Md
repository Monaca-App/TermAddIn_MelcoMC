# コマンドリファレンス：Mc_Receive

## 概要

**Mc_Receive** は、`Mc_Send` 等により送信された要求電文に対するシーケンサからの **MC プロトコル 3E 応答フレームを受信し、終了コードを検証して応答データ部を抽出** する汎用受信用コマンドです。
抽出したペイロードは、第 2 引数 `Format` の指定に従い 16 進ダンプ文字列（既定値 `"HEX"`）または 10 進ワード文字列（`"DEC"`）へ整形して `DestKey` に格納します。

---

## 💡 書式・パラメータ仕様

### コマンド構文

```csv
,SEQUENCE, [ステップキー], Mc_Receive, [DestKey], "[Format: HEX]"
```

### パラメータ（引数）一覧

| 引数位置 | 引数名 | 必須/任意 | 説明・制約 |
| :--- | :--- | :---: | :--- |
| 第 1 引数 | `DestKey` | **必須** | 抽出された応答データ部を格納する変数キー名またはセル指定（例: `"@RAW_RES"`）。 |
| 第 2 引数 | `Format` | 任意 | 格納データ形式（`"HEX"`: 16 進ダンプ文字列、`"DEC"`: 16 ビットワード単位の 10 進カンマ区切り文字列、省略時既定値: `"HEX"`）。 |

---

## 🔄 特殊置換規約・分岐規約の適用

本コマンドの引数における特殊記法および解釈ルールです。
* **動的置換（`$`）**: 引数内で `$PARAMキー$` のように記述すると、実行時に変数プール（PARAM）の値へ動的置換されます。（適用可能な引数: 全引数対応）
* **ジャンプ先指定（`#`）**: 本コマンドはジャンプ制御を行わないため適用されません。（適用可能な引数: 非適用）
* **共通コマンド指定（`@`）**: 本コマンドは外部マクロを呼び出さないため、共通コマンド呼び出し記法は適用されません。（適用可能な引数: 非適用）
* **カンマのエスケープ**: 通信条件文字列（例: `"9600,E,7,1"`）や複数データのカンマ区切り値を単一引数として渡す場合は、必ず `"..."` と二重引用符で囲んで記述します。
* **エイリアス（別名）**: `Mc_R`, `Mc_Res_R`, `Analysis_Mc_Receive`

---

## 💡 動作例（正しい構文での記述例）

### スクリプト記述例

※COMMAND ブロック配下の子要素行（PARAM, SEQUENCE）は必ず行頭カンマから開始し、スペースインデントや行末コメント（`;`）は記述しないでください。

```csv
COMMAND, "SAMPLE_MC_RECEIVE", ""
; 1. 接続開始
,SEQUENCE, 010, Mc_Open, "192.168.1.10", "5000", "TCP"
; 2. 3E 要求フレーム（D100 から 2 ワード読出）を直接送信
,SEQUENCE, 020, Mc_Send, "500000FFFF03000C00100001040000640000A80200"
; 3. 3E 応答フレームを受信し、ペイロードを HEX 形式（既定）で取得
,SEQUENCE, 030, Mc_Receive, "@RAW_RES", "HEX"
; 4. 切断
,SEQUENCE, 040, Mc_Close
,SEQUENCE, 050, Exit
```

### 実行時の挙動・変数変化

* **実行前の状態**: `Mc_Send` により要求フレームが送出され、受信リングバッファまたは `RecData` に 3E 応答フレームが到着している状態。
* **実行後の状態**: ステップ `030` で 3E サブヘッダ（`D0 00`）および終了コード（`0x0000`）が検証され、抽出されたデータペイロードが 16 進文字列（`Format` に `"DEC"` を指定した場合は 10 進カンマ区切り文字列）として `@RAW_RES` に格納されます。

---

## 🔄 動作フロー（シーケンス図）

```mermaid
sequenceDiagram
    autonumber
    participant Script as "シーケンス実行エンジン"
    participant Cmd as "Mc_ReceiveCommand"
    participant Engine as "Mc3EFrameEngine"
    participant Buf as "TermRingBufferManager / 変数プール"

    Script->>Cmd: Func(parent, param)
    activate Cmd
    Cmd->>Cmd: パラメータ範囲検証 (DestKey, Format)
    Cmd->>Buf: ReadResponseBytes(param)
    alt 応答データが空 (無応答) の場合
        Cmd->>Buf: DestKey へ "ERROR_TIMEOUT" を格納
        Cmd-->>Script: 受信失敗 (IsCommandGo = false)
    else 応答バイト列を取得した場合
        Cmd->>Engine: TryParseBinary3EResponse(respBytes)
        alt 終了コード正常 (endCode == 0x0000)
            Cmd->>Cmd: Format ("HEX" または "DEC") に従いペイロード整形
            Cmd->>Buf: DestKey および RecData へ整形結果を格納
            Cmd-->>Script: 正常完了 (IsCommandGo = true)
        else 3E ヘッダ不正または異常終了コードの場合
            Cmd->>Buf: DestKey へエラーメッセージを格納
            Cmd-->>Script: 異常応答停止 (IsCommandGo = false)
        end
    end
    deactivate Cmd
```

---

## 🚫 エラーハンドリング・安全設計（仕様制限）

入力データの異常や実行時エラーに対し、解析・実行エンジンが実行するセーフティ規約です。

1. **引数範囲・プロトコル書式バリデーション保護**:
   - 必須引数の欠落、デバイス表記・数値範囲の不正、または余剰引数が指定された場合は `_log.LogError` に詳細を記録し、`SetErrorAndHalt` により異常電文の送出を未然に遮断します。
2. **通信タイムアウト・異常応答・排他ドメイン保護**:
   - 相手機器の無応答タイムアウト、NAK／異常終了コード応答、または他通信ドメインとの排他競合が発生した場合は、例外クラッシュを起こさず安全にエラーハンドリングを行います。

---

## 🧪 ユニットテスト仕様

本コマンドクラス（`Mc_ReceiveCommand.cs`）に実装されている `RunUnitTest()` メソッドが検証するユニットテストの仕様です。

### テスト実行前提条件

* **実行トリガー**: `RunUnitTest()` メソッドの呼び出し
* **有効化条件**: システム全体の最小ログレベル（`ErrorLogger.MinimumLevel`）が `LogLevel.Test` であること（それ以外の場合は無効化ガードにより早期リターン）
* **検証結果出力**: `_log.LogTest` によるログ出力。検証失敗時は `_log.LogError`、予期せぬ例外発生時は `_log.LogCritical` で記録し、`finally` ブロックで `SocketTransportCore.Shared.IsSimulationMode` を開始前の状態へ復元します。

### テストケース一覧

| No | テストカテゴリ | テスト条件（入力値） | 期待される結果（出力値） | 検証の目的 / 観点 |
| :---: | :--- | :--- | :--- | :--- |
| **1** | 正常系 | `IsSimulationMode = true` 環境下で引数 `"RCV_HEX"`, `"HEX"` および擬似 3E 応答（`D0 00 00 FF FF 03 00 04 00 00 00 BB AA`）を与えて実行 | `!param.HasError` かつ `param.IsCommandGo == true`、格納先 `SBuf["RCV_HEX"]` が `"BBAA"` と一致する | 3E 応答フレームの終了コード検証および HEX ペイロード抽出格納の検証 |
| **2** | 異常系 | 必須引数 `DestKey` 欠落・引数範囲外（`ValidateParamRange(1, 3)` 違反）、空応答受信、または 3E 異常終了コード応答 | 空応答時は `DestKey` に `"ERROR_TIMEOUT"`、異常終了コード時はエラー文字列を格納して `IsCommandGo = false` で安全停止する | 引数境界検証および無応答・PLC エラー応答時のフェイルセーフ検証 |
