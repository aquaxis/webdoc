---
title: "第12章：SystemVerilog Assertions（SVA）"
---

# 第12章：SystemVerilog Assertions（SVA）

## 12.1 この章で学ぶこと

本章では、**SystemVerilog Assertions（SVA）**について解説します。アサーションとは、設計が意図した通りに動作していることを形式的に記述するための仕組みです。バグの早期発見、設計仕様の形式化、デバッグの効率化に大きく貢献します。即時アサーションと並行アサーション、シーケンスとプロパティの記述方法、そしてアサーションを活用した動作検証の手法を学びます。

---

## 12.2 アサーションとは

### 12.2.1 アサーションの役割

アサーションは、設計の**不変条件**（常に成り立つべき条件）や**時間的な振る舞い**（あるイベントの後にこうなるべき、という条件）を記述します。テストベンチで条件をチェックする `if` 文との主な違いは以下のとおりです：

- **宣言的**: 「何が正しいか」を記述する（手続き的な「どうチェックするか」ではない）
- **常時監視**: シミュレーション全体を通じて自動的に評価される
- **形式検証対応**: 形式検証（Formal Verification）ツールでも使用可能
- **カバレッジ連携**: アサーションカバレッジとの統合が容易

### 12.2.2 アサーションの分類

SystemVerilogのアサーションは大きく2種類に分かれます：

| 種類 | 特徴 | 時間的振る舞い | 使用場面 |
|------|------|--------------|---------|
| **即時アサーション** | 手続きブロック内で即座に評価 | なし（単一時刻） | 組み合わせ条件のチェック |
| **並行アサーション** | クロックに同期して評価 | あり（複数サイクル） | プロトコル・タイミング検証 |

---

## 12.3 即時アサーション（Immediate Assertions）

### 12.3.1 基本構文

即時アサーションは、`always` ブロックや `initial` ブロックなどの手続きコード内で使用します。指定した条件が `false` になるとアサーション違反が報告されます。

```systemverilog
// 基本的な即時アサーション
always_comb begin
    assert (state != ILLEGAL_STATE)
        else $error("Illegal state detected: %s", state.name());
end

// initial ブロック内での使用
initial begin
    Packet pkt = new();
    assert (pkt.randomize())
        else $fatal(1, "Randomization failed!");
end
```

### 12.3.2 アサーションの3つのブロック

即時アサーションには、成功時と失敗時の処理を指定できます。

```systemverilog
always_ff @(posedge clk) begin
    // assert (条件) 成功時の処理 else 失敗時の処理;
    assert (!data_valid || data !== 'x)
        $display("[%0t] Data valid check passed", $time);  // 成功時（省略可）
    else
        $error("[%0t] Data is X when valid is high!", $time);  // 失敗時
end
```

### 12.3.3 重大度レベル

アサーション違反時のメッセージには4つの重大度があります：

| システムタスク | 重大度 | シミュレーションへの影響 |
|--------------|--------|----------------------|
| `$info()` | 情報 | 続行 |
| `$warning()` | 警告 | 続行 |
| `$error()` | エラー | 続行（デフォルト） |
| `$fatal()` | 致命的 | 即座に停止 |

```systemverilog
// 重大度の使い分け
always_ff @(posedge clk) begin
    // パフォーマンス警告
    assert (fifo_count < FIFO_DEPTH - 2)
        else $warning("FIFO almost full: %0d/%0d", fifo_count, FIFO_DEPTH);

    // プロトコル違反
    assert (!(wr_en && rd_en))
        else $error("Simultaneous read and write!");

    // 致命的エラー
    assert (rst_n !== 1'bx)
        else $fatal(1, "Reset signal is X!");
end
```

---

## 12.4 並行アサーション（Concurrent Assertions）

### 12.4.1 基本構文

並行アサーションはクロックに同期して評価され、複数サイクルにわたる時間的な振る舞いを検証できます。`assert property` 構文を使用します。

```systemverilog
// クロックの立ち上がりで評価される並行アサーション
assert property (@(posedge clk) disable iff (!rst_n)
    req |-> ##[1:3] ack  // reqの後、1〜3サイクル以内にackが来る
) else $error("ACK not received within 3 cycles of REQ");
```

### 12.4.2 シーケンス（Sequence）

**シーケンス**は、時間的なイベントの連なりを記述するための構文です。`sequence` キーワードで名前を付けて再利用できます。

```systemverilog
// シーケンスの定義
sequence req_ack_seq;
    req ##[1:3] ack;  // reqの後、1〜3サイクル後にack
endsequence

sequence write_seq;
    wr_en ##1 (data_valid && !busy) ##1 wr_done;
endsequence

// 名前付きシーケンスをアサーションで使用
assert property (@(posedge clk) disable iff (!rst_n)
    req_ack_seq
);
```

#### 遅延演算子（##）

| 演算子 | 意味 | 例 |
|--------|------|-----|
| `##n` | ちょうどnサイクル後 | `a ##2 b` → aの2サイクル後にb |
| `##[m:n]` | m〜nサイクル後 | `a ##[1:5] b` → aの1〜5サイクル後にb |
| `##[0:$]` | 0サイクル以上（いつか） | `a ##[0:$] b` → aの後いつかb |

```systemverilog
// 遅延演算子の例
sequence handshake;
    req                    // サイクル0: reqがアサート
    ##1 (gnt && !busy)     // サイクル1: gntがアサートされbusyでない
    ##1 data_valid          // サイクル2: data_validがアサート
    ##[1:4] done;          // サイクル3〜6: doneがアサート
endsequence
```

### 12.4.3 プロパティ（Property）

**プロパティ**は、シーケンスに含意（implication）や否定などの論理を加えた、検証したい条件を記述するものです。

```systemverilog
// プロパティの定義
property req_must_be_acked;
    @(posedge clk) disable iff (!rst_n)
    req |-> ##[1:5] ack;  // reqが来たら1〜5サイクル以内にackが来る
endproperty

// プロパティのアサート
assert property (req_must_be_acked)
    else $error("REQ was not ACKed in time!");

// プロパティのカバー（このパターンが実際に発生したか記録）
cover property (req_must_be_acked);
```

#### 含意演算子（Implication）

含意演算子は「前提条件が成り立つ場合に帰結が成り立つ」ことを検証します。

| 演算子 | 名称 | 意味 |
|--------|------|------|
| `\|->` | オーバーラッピング含意 | 前提と同じサイクルから帰結を評価 |
| `\|=>` | ノンオーバーラッピング含意 | 前提の次のサイクルから帰結を評価 |

```systemverilog
property overlapping_example;
    @(posedge clk)
    req |-> ack;           // reqと同じサイクルでackを確認
endproperty

property non_overlapping_example;
    @(posedge clk)
    req |=> ack;           // reqの次のサイクルでackを確認
    // req |=> ack は req |-> ##1 ack と同等
endproperty
```

![含意演算子の動作](/images/systemverilog-complete-guide/ch12_implication_operators.drawio.png)

---

## 12.5 高度なシーケンス演算子

### 12.5.1 繰り返し演算子

```systemverilog
// 連続繰り返し [*n]: n回連続
sequence burst_4;
    req ##1 data_valid[*4] ##1 done;
    // data_validが4サイクル連続 → done
endsequence

// 範囲繰り返し [*m:n]: m〜n回連続
sequence variable_burst;
    req ##1 data_valid[*1:8] ##1 done;
endsequence

// goto繰り返し [->n]: 必ずしも連続でないn回
sequence scattered_ack;
    req ##1 ack[->3];  // ackが（非連続で）3回発生
endsequence

// ノンコンセキュティブ繰り返し [=n]: 非連続n回（最後の出現後に別のシグナルが来てもOK）
sequence non_consec;
    req ##1 ack[=3] ##1 done;
endsequence
```

### 12.5.2 intersect

`intersect` は、2つのシーケンスが同時に開始し、同時に終了することを要求します。

```systemverilog
sequence data_phase;
    data_valid[*4];    // 4サイクル
endsequence

sequence handshake_phase;
    req ##3 ack;       // 4サイクル（0〜3）
endsequence

// 両方が同じ長さで同時に発生
sequence full_transfer;
    data_phase intersect handshake_phase;
endsequence
```

### 12.5.3 within

`within` は、一方のシーケンスが他方のシーケンスの期間内に完全に含まれることを検証します。

```systemverilog
sequence short_seq;
    a ##1 b;           // 2サイクル
endsequence

sequence long_seq;
    c ##1 d ##1 e ##1 f;  // 4サイクル
endsequence

// short_seqがlong_seqの期間内に発生する
property check_within;
    @(posedge clk)
    short_seq within long_seq;
endproperty
```

### 12.5.4 throughout

`throughout` は、シーケンス全体を通じてある信号が一定の値を保つことを検証します。

```systemverilog
// バースト転送中、busyが常にHighであること
property burst_keeps_busy;
    @(posedge clk) disable iff (!rst_n)
    $rose(burst_start) |->
        busy throughout (data_valid[*1:16] ##1 burst_done);
endproperty

assert property (burst_keeps_busy);
```

### 12.5.5 first_match

`first_match` は、複数のマッチが可能な場合に最初のマッチのみを採用します。

```systemverilog
property first_ack;
    @(posedge clk)
    req |-> first_match(##[1:10] ack);
    // 1〜10サイクル以内の最初のackのみマッチ
endproperty
```

---

## 12.6 システム関数

SVAでよく使われるシステム関数を紹介します。

```systemverilog
property edge_detection;
    @(posedge clk)
    // $rose: 0→1の立ち上がりを検出
    $rose(start) |-> ##[1:5] $rose(done);
endproperty

property level_checks;
    @(posedge clk)
    // $fell: 1→0の立ち下がりを検出
    $fell(valid) |-> ##1 !busy;
endproperty

property stable_check;
    @(posedge clk)
    // $stable: 前サイクルから値が変化していないことを検出
    hold_req |-> $stable(data);
endproperty

property past_check;
    @(posedge clk)
    // $past: 1サイクル前の値を参照
    done |-> (result == expected_result) && ($past(valid) == 1'b1);
endproperty

property count_check;
    @(posedge clk)
    // $countones: ビット中の1の数をカウント
    valid |-> ($countones(byte_enable) > 0);
endproperty
```

---

## 12.7 アサーションの実践例

### 12.7.1 AXIプロトコルのアサーション

```systemverilog
// AXI仕様: VALIDはREADYがアサートされるまで保持しなければならない
property axi_valid_stable;
    @(posedge aclk) disable iff (!aresetn)
    (awvalid && !awready) |=> awvalid;
endproperty

// AXI仕様: VALIDがアサートされたら、アドレスは安定していなければならない
property axi_addr_stable;
    @(posedge aclk) disable iff (!aresetn)
    (awvalid && !awready) |=> $stable(awaddr);
endproperty

// AXI仕様: RESPはVALIDと同時にアサートされる
property axi_resp_with_valid;
    @(posedge aclk) disable iff (!aresetn)
    bvalid |-> (bresp !== 2'bxx);
endproperty

// アサーションの適用
assert property (axi_valid_stable)
    else $error("AXI Protocol Violation: AWVALID dropped before AWREADY");
assert property (axi_addr_stable)
    else $error("AXI Protocol Violation: AWADDR changed while waiting for AWREADY");
assert property (axi_resp_with_valid)
    else $error("AXI Protocol Violation: BRESP is X when BVALID is high");
```

### 12.7.2 FIFOのアサーション

```systemverilog
module fifo_assertions #(parameter DEPTH = 16) (
    input logic clk,
    input logic rst_n,
    input logic push,
    input logic pop,
    input logic full,
    input logic empty,
    input int   count
);

    // オーバーフロー検出
    property no_overflow;
        @(posedge clk) disable iff (!rst_n)
        full |-> !push;
    endproperty

    // アンダーフロー検出
    property no_underflow;
        @(posedge clk) disable iff (!rst_n)
        empty |-> !pop;
    endproperty

    // カウンタの整合性
    property count_consistency;
        @(posedge clk) disable iff (!rst_n)
        (count == 0) == empty && (count == DEPTH) == full;
    endproperty

    // push後にemptyは解除される
    property push_clears_empty;
        @(posedge clk) disable iff (!rst_n)
        (empty && push && !pop) |=> !empty;
    endproperty

    assert property (no_overflow)     else $error("FIFO overflow!");
    assert property (no_underflow)    else $error("FIFO underflow!");
    assert property (count_consistency) else $error("Count inconsistency!");
    assert property (push_clears_empty) else $error("Push didn't clear empty!");

endmodule
```

### 12.7.3 bind によるアサーションの接続

`bind` を使うと、設計モジュールのソースコードを変更せずにアサーションモジュールを接続できます。

```systemverilog
// アサーションモジュールをFIFOに接続
bind fifo fifo_assertions #(.DEPTH(DEPTH)) u_assertions (
    .clk   (clk),
    .rst_n (rst_n),
    .push  (push),
    .pop   (pop),
    .full  (full),
    .empty (empty),
    .count (count)
);
```

---

## 12.8 まとめ

本章では、SystemVerilog Assertions（SVA）について学びました。

1. **即時アサーション**は手続きコード内で単一時刻の条件を検証します。`$error`、`$fatal` などで重大度を指定します。
2. **並行アサーション**はクロックに同期して複数サイクルの時間的振る舞いを検証します。`assert property` で記述します。
3. **シーケンス**（`sequence`）は時間的なイベントの連なりを定義します。`##n`（遅延）、`[*n]`（繰り返し）などの演算子を使用します。
4. **プロパティ**（`property`）はシーケンスに含意（`|->`, `|=>`）を加えた検証条件です。
5. **高度な演算子**として `intersect`（同時開始・終了）、`within`（包含）、`throughout`（持続条件）があります。
6. **`bind`** で設計ソースを変更せずにアサーションを接続できます。

次章では、テストの網羅性を数値化する機能カバレッジについて学びます。
