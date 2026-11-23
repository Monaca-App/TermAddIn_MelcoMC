# コマンドリファレンス：Mc_WriteWord

## 概要

**Mc_WriteWord** は、MELSEC MC プロトコル（TCP・バイナリ 3E フレーム、コマンド `1401` サブコマンド `0000`）を用いて、シーケンサの **ワードデバイス（`D`, `W`, `R`, `ZR`, `TN`, `CN`）へ一括書込** を行う通信制御系コマンドです。
書き込むデータは 10 進数または 16 進数（`0x` プレフィックス付き）のカンマ区切り、あるいはスペース区切りで指定できます。

---

## 💡 書式・パラメータ仕様

### コマンド構文

```csv
,SEQUENCE, [ステップキー], Mc_WriteWord, [Device], [Values], "[Reserved]"
```

### パラメータ（引数）一覧

| 引数位置 | 引数名 | 必須/任意 | 説明・制約 |
| :--- | :--- | :---: | :--- |
| 第 1 引数 | `Device` | **必須** | 書き込みを開始する先頭ワードデバイス名（対応デバイス: `D`, `W`, `R`, `ZR`, `TN`, `CN`。例: `"D100"`, `"W0"`, `"R10"`, `"ZR1000"`, `"TN0"`, `"CN0"`）。 |
| 第 2 引数 | `Values` | **必須** | 書き込む 16 ビット符号なしワード値列（カンマまたは空白区切り、例: `"100,200"` または `"0x1234,0x5678"`、要素数から書込点数を自動算出）。 |
| 第 3 引数 | `Reserved` | 任意 | 互換用の予備引数（`ValidateParamRange(2, 3)` により 3 引数まで許容されますが、指定された値は参照されません）。 |

---

## 🔄 特殊置換規約・分岐規約の適用

本コマンドの引数における特殊記法および解釈ルールです。
* **動的置換（`$`）**: 引数内で `$PARAMキー$` のように記述すると、実行時に変数プール（PARAM）の値へ動的置換されます。（適用可能な引数: 全引数対応）
* **ジャンプ先指定（`#`）**: 本コマンドはジャンプ制御を行わないため適用されません。（適用可能な引数: 非適用）
* **共通コマンド指定（`@`）**: 本コマンドは外部マクロを呼び出さないため、共通コマンド呼び出し記法は適用されません。（適用可能な引数: 非適用）
* **カンマのエスケープ**: 複数データのカンマ区切り値（例: `"100,200"`）を単一引数として渡す場合は、必ず `"..."` と二重引用符で囲んで記述します。
* **エイリアス（別名）**: `Mc_Write_Word`, `Analysis_Mc_WriteWord`

---

## 💡 動作例（正しい構文での記述例）

### スクリプト記述例

※COMMAND ブロック配下の子要素行（PARAM, SEQUENCE）は必ず行頭カンマから開始し、スペースインデントや行末コメント（`;`）は記述しないでください。

```csv
COMMAND, "MC_WRITE_WORD_DEMO", ""
,PARAM, "WORD_VALUES", Text, "書込ワード列", "100,200"
,SEQUENCE, 010, Mc_Open, "192.168.1.10", "5000", "TCP"
; D100 から 2 ワード（100, 200）を一括書込
,SEQUENCE, 020, Mc_WriteWord, "D100", "$WORD_VALUES$"
,SEQUENCE, 030, Mc_Close
,SEQUENCE, 040, Exit
```

### 実行時の挙動・変数変化

* **実行前の状態**: PLC（`192.168.1.10:5000`）への接続待機中で、`"WORD_VALUES"` に `"100,200"` が設定されている状態。
* **実行後の状態**: ステップ `020` で 3E ワード一括書込フレーム（コマンド `1401` / サブコマンド `0000`）が送信され、`D100=100`, `D101=200` の書込完了応答（終了コード `0x0000`）が確認されて `param.IsCommandGo = true` となります。

---

## 🔄 動作フロー（シーケンス図）

```mermaid
sequenceDiagram
    autonumber
    participant Driver as "TermSequenceDriver (実行エンジン)"
    participant Cmd as "Mc_WriteWordCommand"
    participant Engine as "McDeviceParser / Mc3EFrameEngine"
    participant Socket as "SocketTransportCore / RingBuffer"

    Driver->>Driver: ライセンス検証 (ValidateStep)
    Driver->>Cmd: Func(parent, param)
    activate Cmd
    Cmd->>Cmd: ValidateParamRange(2, 3) ＆ $PARAM$ 展開
    Cmd->>Engine: Parse(devStr) ＆ ParseWordValues(valsStr) ＆ BuildBinaryWordBatchWriteFrame
    Cmd->>Socket: SendBytes(reqPacket) ＆ ReadResponseBytes(param)
    alt 正常 3E 応答受信 (endCode == 0x0000)
        Cmd-->>Driver: 正常完了 (IsCommandGo = true)
    else 無応答タイムアウト または 3E 異常終了コード応答
        Cmd->>Cmd: SetErrorAndHalt(-1, errMsg)
        Cmd-->>Driver: 異常停止 (IsCommandGo = false)
    else 引数範囲外 または デバイス・数値パース例外
        Cmd->>Cmd: _log.LogError / _log.LogCritical ＆ SetErrorAndHalt
        Cmd-->>Driver: 異常停止 (IsCommandGo = false)
    end
    deactivate Cmd
```

---

## 🚫 エラーハンドリング・安全設計（仕様制限）

入力データの異常や実行時エラーに対し、解析・実行エンジンが実行するセーフティ規約です。

1. **引数範囲およびデバイス・数値構文の検証**:
   - 引数が 1 個以下または 4 個以上の場合は `ValidateParamRange(2, 3)` により `_log.LogError` と `SetErrorAndHalt` で遮断します。未対応デバイスや `ushort` 範囲外の数値指定により `McDeviceParser.Parse` / `ParseWordValues` が例外を送出した場合は、`catch` 節で `_log.LogCritical` に記録し `SetErrorAndHalt(-1, ex.Message)` で安全停止します。
2. **書込応答タイムアウトおよび 3E 異常終了コード時の停止**:
   - 受信データが空（タイムアウト）の場合、または PLC が非 `0x0000` の終了コードや不正フレームを返した場合は、`SetErrorAndHalt(-1, ...)` を呼び出してシーケンスを即座に安全停止します。

---

## 🧪 ユニットテスト仕様

本コマンドクラス（`Mc_WriteWordCommand.cs`）に実装されている `RunUnitTest()` メソッドが検証するユニットテストの仕様です。

### テスト実行前提条件

* **実行トリガー**: `RunUnitTest()` メソッドの呼び出し
* **有効化条件**: システム全体の最小ログレベル（`ErrorLogger.MinimumLevel`）が `LogLevel.Test` であること（それ以外の場合は無効化ガードにより早期リターン）
* **検証結果出力**: `_log.LogTest` によるログ出力。検証失敗時は `_log.LogError`、予期せぬ例外発生時は `_log.LogCritical` で記録し、`finally` ブロックで `SocketTransportCore.Shared.IsSimulationMode` を開始前の状態へ復元します。

### テストケース一覧

| No | テストカテゴリ | テスト条件（入力値） | 期待される結果（出力値） | 検証の目的 / 観点 |
| :---: | :--- | :--- | :--- | :--- |
| **1** | 正常系 | `IsSimulationMode = true` 環境下で引数 `"D100"`, `"100,200"` および擬似 3E 正常応答（`D0 00 00 FF FF 03 00 02 00 00 00`）を与えて実行 | `!param.HasError` かつ `param.IsCommandGo == true` でワード一括書込が正常完了する | 3E ワード一括書込フレーム生成および終了コード `0x0000` 応答検証 |
| **2** | 異常系 | 引数 0 個（`new TermSequence { Name = "Mc_WriteWord" }`）で `Func(null, invalidParam)` を実行 | `ValidateParamRange(2, 3)` により拒否され、`invalidParam.HasError == true` かつ `!invalidParam.IsCommandGo` となる | 必須引数欠落時のバリデーション拒否検証 |
