---
title: "第17章：デザインパターンと実践"
---

# 第17章：デザインパターンと実践

## 17.1 この章で学ぶこと

本章では、SystemVerilogにおける**デザインパターンと実践的なコーディング技法**について解説します。ソフトウェア工学で確立されたデザインパターンの考え方をハードウェア設計・検証に適用し、再利用性・保守性・信頼性の高い設計を実現する方法を学びます。RTL設計における頻出パターン（FSM、パイプライン、ハンドシェイク、FIFO、アービタ）、検証におけるテストベンチアーキテクチャパターン、再利用可能なコンポーネント設計、コーディングベストプラクティス、よくある落とし穴とアンチパターン、そしてUVM（Universal Verification Methodology）の概要まで、実践的なSystemVerilog開発に必要な知識を体系的に習得します。

![デザインパターンの全体像](/images/systemverilog-complete-guide/ch16_design_patterns_overview.drawio.png)

---

## 17.2 RTLデザインパターン

RTL設計では、特定の回路構造が繰り返し現れます。これらをパターンとして理解し、正しく実装することで、合成可能で効率的なハードウェアを設計できます。

### 17.2.1 FSM（有限状態マシン）パターン

FSMはデジタル回路で最も基本的なデザインパターンです。推奨される2プロセス方式では、順序回路と組み合わせ回路を明確に分離します。

```systemverilog
module fsm_two_process (
    input  logic       clk, rst_n, start, done,
    output logic [1:0] ctrl
);
    typedef enum logic [1:0] {
        IDLE = 2'b00, RUN = 2'b01, WAIT = 2'b10, FINISH = 2'b11
    } state_t;

    state_t current_state, next_state;

    // プロセス1：状態レジスタ（順序回路）
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) current_state <= IDLE;
        else        current_state <= next_state;
    end

    // プロセス2：次状態論理 + 出力論理（組み合わせ回路）
    always_comb begin
        next_state = current_state; // デフォルト値でラッチ推論を防止
        ctrl       = 2'b00;
        case (current_state)
            IDLE:   begin if (start) next_state = RUN;    ctrl = 2'b00; end
            RUN:    begin next_state = WAIT;               ctrl = 2'b01; end
            WAIT:   begin if (done) next_state = FINISH;   ctrl = 2'b10; end
            FINISH: begin next_state = IDLE;               ctrl = 2'b11; end
            default: next_state = IDLE;
        endcase
    end
endmodule
```

- **`enum` 型の使用**: 状態名を文字列として扱え、デバッグ性が向上します
- **2プロセス方式**: 順序回路と組み合わせ回路を分離し、見通しを良くします
- **デフォルト値の設定**: `always_comb` 内の先頭でデフォルト値を設定し、ラッチ推論を防止します

![FSMの2プロセスパターン](/images/systemverilog-complete-guide/ch16_fsm_two_process.drawio.png)

### 17.2.2 パイプラインパターン

パイプラインは、処理をステージに分割してスループットを向上させるパターンです。各ステージ間にレジスタを挿入し、データを1サイクルずつ流します。

```systemverilog
module pipeline_example #(parameter DATA_WIDTH = 32)(
    input  logic                   clk, rst_n, valid_in,
    input  logic [DATA_WIDTH-1:0]  data_in,
    output logic                   valid_out,
    output logic [DATA_WIDTH-1:0]  data_out
);
    logic valid_s1, valid_s2;
    logic [DATA_WIDTH-1:0] data_s1, data_s2;

    // ステージ1
    always_ff @(posedge clk or negedge rst_n)
        if (!rst_n) begin valid_s1 <= 0; data_s1 <= '0; end
        else        begin valid_s1 <= valid_in; data_s1 <= data_in + 1; end

    // ステージ2
    always_ff @(posedge clk or negedge rst_n)
        if (!rst_n) begin valid_s2 <= 0; data_s2 <= '0; end
        else        begin valid_s2 <= valid_s1; data_s2 <= data_s1 << 1; end

    // ステージ3（最終出力）
    always_ff @(posedge clk or negedge rst_n)
        if (!rst_n) begin valid_out <= 0; data_out <= '0; end
        else        begin valid_out <= valid_s2; data_out <= data_s2 ^ 32'hDEAD; end
endmodule
```

- **バリッドビットの伝搬**: データと一緒に `valid` 信号をパイプラインに通し、有効性を追跡します
- **パイプラインストール**: バックプレッシャーが必要な場合は `stall` 信号を追加します

### 17.2.3 ハンドシェイクプロトコルパターン

モジュール間のデータ転送を安全に行うために、`valid`/`ready` 方式のハンドシェイクプロトコルを使用します。AXIバスプロトコルでも採用されている標準的な手法です。

```systemverilog
interface handshake_if #(parameter DATA_WIDTH = 32)(
    input logic clk, input logic rst_n
);
    logic valid, ready;
    logic [DATA_WIDTH-1:0] data;

    modport src (output valid, data, input  ready); // 送信側
    modport dst (input  valid, data, output ready); // 受信側
endinterface
```

![ハンドシェイクプロトコルのタイミング](/images/systemverilog-complete-guide/ch16_handshake_timing.drawio.png)

- **valid先行**: 送信側は `ready` を待たずに `valid` をアサートできます
- **転送成立**: `valid && ready` が同時に成立したクロックエッジでデータが転送されます
- **デッドロック回避**: `valid` は `ready` に依存してはならないルールが一般的です

### 17.2.4 FIFOパターン

FIFOは異なる速度で動作するモジュール間のデータバッファです。ポインタのMSBでラップアラウンドを検出する手法が一般的です。

```systemverilog
module sync_fifo #(
    parameter DATA_WIDTH = 8, DEPTH = 16, ADDR_WIDTH = $clog2(DEPTH)
)(
    input  logic clk, rst_n, wr_en, rd_en,
    input  logic [DATA_WIDTH-1:0] wr_data,
    output logic [DATA_WIDTH-1:0] rd_data,
    output logic full, empty
);
    logic [DATA_WIDTH-1:0] mem [DEPTH];
    logic [ADDR_WIDTH:0] wr_ptr, rd_ptr; // MSBはラップアラウンド検出用

    assign full  = (wr_ptr[ADDR_WIDTH] != rd_ptr[ADDR_WIDTH]) &&
                   (wr_ptr[ADDR_WIDTH-1:0] == rd_ptr[ADDR_WIDTH-1:0]);
    assign empty = (wr_ptr == rd_ptr);

    always_ff @(posedge clk or negedge rst_n)
        if (!rst_n) wr_ptr <= '0;
        else if (wr_en && !full) begin
            mem[wr_ptr[ADDR_WIDTH-1:0]] <= wr_data;
            wr_ptr <= wr_ptr + 1;
        end

    always_ff @(posedge clk or negedge rst_n)
        if (!rst_n) rd_ptr <= '0;
        else if (rd_en && !empty) rd_ptr <= rd_ptr + 1;

    assign rd_data = mem[rd_ptr[ADDR_WIDTH-1:0]];
endmodule
```

### 17.2.5 アービタパターン

複数のリクエスタが共有リソースにアクセスする場合、アービタによる調停が必要です。ラウンドロビン方式は全リクエスタに均等な帯域を保証します。

```systemverilog
module round_robin_arbiter #(parameter NUM_REQ = 4)(
    input  logic clk, rst_n,
    input  logic [NUM_REQ-1:0] req,
    output logic [NUM_REQ-1:0] grant  // ワンホットエンコーディング
);
    logic [$clog2(NUM_REQ)-1:0] priority_ptr;
    logic [NUM_REQ-1:0] mask_req, masked_grant, unmasked_grant;

    always_comb begin
        mask_req = '0;
        for (int i = 0; i < NUM_REQ; i++)
            if (i >= priority_ptr) mask_req[i] = req[i];
    end

    assign masked_grant   = mask_req & (~mask_req + 1); // 最下位ビット優先
    assign unmasked_grant = req & (~req + 1);
    assign grant = (|mask_req) ? masked_grant : unmasked_grant;

    always_ff @(posedge clk or negedge rst_n)
        if (!rst_n) priority_ptr <= '0;
        else if (|grant)
            for (int i = 0; i < NUM_REQ; i++)
                if (grant[i]) priority_ptr <= (i + 1) % NUM_REQ;
endmodule
```

---

## 17.3 検証デザインパターン

検証環境にもデザインパターンがあります。テストベンチを体系的に構築することで、再利用性と拡張性が向上します。

### 17.3.1 階層化テストベンチパターン

階層化テストベンチでは、テストベンチを機能ごとのレイヤーに分割します。

![階層化テストベンチの構造](/images/systemverilog-complete-guide/ch16_layered_testbench.drawio.png)

| レイヤー | 責務 | 例 |
|---------|------|-----|
| **テストレイヤー** | テストシナリオの定義 | テストケース、制約の設定 |
| **シナリオレイヤー** | トランザクションの生成 | パケットシーケンスの生成 |
| **ファンクショナルレイヤー** | プロトコル処理 | ドライバ、モニタ |
| **コマンドレイヤー** | 信号レベルの操作 | ピンレベルのドライブ |
| **シグナルレイヤー** | DUTとの接続 | インターフェース、クロック生成 |

### 17.3.2 ドライバ/モニタ/スコアボードパターン

検証環境の中核を成す3つのコンポーネントです。

```systemverilog
// トランザクションクラス
class Transaction;
    rand bit [7:0]  addr;
    rand bit [31:0] data;
    rand bit        write;
    function string to_string();
        return $sformatf("addr=0x%02h data=0x%08h %s",
                         addr, data, write ? "WR" : "RD");
    endfunction
endclass

// ドライバ：トランザクションを信号レベルに変換
class Driver;
    mailbox #(Transaction) gen2drv;
    virtual bus_if vif;
    task run();
        Transaction tr;
        forever begin
            gen2drv.get(tr);
            @(posedge vif.clk);
            vif.addr <= tr.addr; vif.data <= tr.data;
            vif.write <= tr.write; vif.valid <= 1'b1;
            @(posedge vif.clk);
            vif.valid <= 1'b0;
        end
    endtask
endclass

// モニタ：信号を観測してトランザクションを再構成
class Monitor;
    mailbox #(Transaction) mon2scb;
    virtual bus_if vif;
    task run();
        Transaction tr;
        forever begin
            @(posedge vif.clk);
            if (vif.valid) begin
                tr = new(); tr.addr = vif.addr;
                tr.data = vif.data; tr.write = vif.write;
                mon2scb.put(tr);
            end
        end
    endtask
endclass

// スコアボード：期待値と実際の結果を比較
class Scoreboard;
    mailbox #(Transaction) mon2scb;
    Transaction expected_queue[$];
    int pass_count, fail_count;

    task run();
        Transaction actual, expected;
        forever begin
            mon2scb.get(actual);
            if (expected_queue.size() > 0) begin
                expected = expected_queue.pop_front();
                if (actual.data == expected.data) pass_count++;
                else begin
                    fail_count++;
                    $error("[SCB] Mismatch: exp=%s act=%s",
                           expected.to_string(), actual.to_string());
                end
            end
        end
    endtask
endclass
```

- **ドライバ**: トランザクションレベルの操作を信号レベルに変換してDUTに印加します
- **モニタ**: DUTの出力信号を監視し、トランザクションレベルに再構成します
- **スコアボード**: 期待される出力と実際の出力を比較し、テスト結果を判定します

### 17.3.3 ファクトリパターン

ファクトリパターンを使うと、テストケースごとに異なるトランザクションクラスを動的に切り替えられます。

```systemverilog
class BaseTransaction;
    rand bit [7:0] addr;
    rand bit [31:0] data;
    virtual function BaseTransaction clone();
        BaseTransaction t = new(); t.addr = this.addr; t.data = this.data;
        return t;
    endfunction
endclass

class ErrorTransaction extends BaseTransaction;
    rand bit inject_error;
    constraint error_c { inject_error dist {1 := 30, 0 := 70}; }
    virtual function BaseTransaction clone();
        ErrorTransaction t = new();
        t.addr = this.addr; t.data = this.data;
        t.inject_error = this.inject_error;
        return t;
    endfunction
endclass

// 簡易ファクトリ：型名でオブジェクトを生成
class TransactionFactory;
    static BaseTransaction prototypes[string];
    static function void register_type(string name, BaseTransaction proto);
        prototypes[name] = proto;
    endfunction
    static function BaseTransaction create(string name);
        if (prototypes.exists(name)) return prototypes[name].clone();
        else begin $error("Unknown type: %s", name); return null; end
    endfunction
endclass
```

### 17.3.4 コールバックパターン

コールバックパターンを使うと、既存のコンポーネントを変更せずに振る舞いを拡張できます。

```systemverilog
class DriverCallback;
    virtual task pre_drive(Transaction tr);  endtask // デフォルト：何もしない
    virtual task post_drive(Transaction tr); endtask
endclass

class CallbackDriver;
    DriverCallback callbacks[$];
    function void add_callback(DriverCallback cb);
        callbacks.push_back(cb);
    endfunction
    task run();
        Transaction tr;
        forever begin
            gen2drv.get(tr);
            foreach (callbacks[i]) callbacks[i].pre_drive(tr);  // 前処理
            drive_transaction(tr);
            foreach (callbacks[i]) callbacks[i].post_drive(tr); // 後処理
        end
    endtask
endclass

// テスト固有のコールバック（エラー注入）
class ErrorInjectionCallback extends DriverCallback;
    virtual task pre_drive(Transaction tr);
        if ($urandom_range(0, 9) == 0) tr.data = $urandom(); // 10%でデータ破壊
    endtask
endclass
```

- **拡張性**: 既存のドライバコードを修正せずに新しい振る舞いを追加できます
- **テスト固有のカスタマイズ**: テストケースごとに異なるコールバックを登録できます

---

## 17.4 再利用可能なコンポーネント設計

### 17.4.1 パラメータ化モジュール

パラメータを使用することで、同じモジュールをさまざまな構成で再利用できます。

```systemverilog
module parameterized_register #(
    parameter int WIDTH = 8, int RESET_VAL = 0, bit ASYNC_RST = 1'b1
)(
    input logic clk, rst_n, en,
    input logic [WIDTH-1:0] d, output logic [WIDTH-1:0] q
);
    generate
        if (ASYNC_RST) begin : gen_async_rst
            always_ff @(posedge clk or negedge rst_n)
                if (!rst_n) q <= WIDTH'(RESET_VAL); else if (en) q <= d;
        end else begin : gen_sync_rst
            always_ff @(posedge clk)
                if (!rst_n) q <= WIDTH'(RESET_VAL); else if (en) q <= d;
        end
    endgenerate
endmodule
```

### 17.4.2 パッケージによる共通定義の管理

パッケージで型、定数、関数を一元管理し、プロジェクト全体で共有します。

```systemverilog
package project_pkg;
    parameter int ADDR_WIDTH = 32;
    parameter int DATA_WIDTH = 64;

    typedef enum logic [1:0] {
        RESP_OKAY = 2'b00, RESP_EXOKAY = 2'b01,
        RESP_SLVERR = 2'b10, RESP_DECERR = 2'b11
    } resp_t;

    typedef struct packed {
        logic [ADDR_WIDTH-1:0] addr;
        logic [DATA_WIDTH-1:0] data;
        logic                  write;
    } bus_cmd_t;

    function automatic logic [31:0] byte_swap(logic [31:0] data);
        return {data[7:0], data[15:8], data[23:16], data[31:24]};
    endfunction
endpackage
```

![再利用可能なコンポーネント設計の構造](/images/systemverilog-complete-guide/ch16_reusable_components.drawio.png)

---

## 17.5 コーディングベストプラクティス

### 17.5.1 命名規則

一貫した命名規則はコードの可読性を大幅に向上させます。

| 対象 | 規則 | 例 |
|------|------|-----|
| モジュール名 | スネークケース（小文字） | `uart_transmitter` |
| インスタンス名 | `u_` プレフィックス | `u_uart_tx` |
| パラメータ | 大文字スネークケース | `DATA_WIDTH` |
| 信号名（アクティブLow） | `_n` サフィックス | `rst_n`, `cs_n` |
| クロック信号 | `clk` プレフィックス | `clk`, `clk_200mhz` |
| 列挙型 | `_t` サフィックス | `state_t` |
| 列挙値 | 大文字スネークケース | `IDLE`, `READ_DATA` |
| `generate` ブロック | `gen_` プレフィックス | `gen_async_rst` |
| レジスタ出力 | `_r` または `_ff` サフィックス | `data_r` |
| 組み合わせ出力 | `_next` または `_w` サフィックス | `data_next` |

### 17.5.2 コーディングガイドライン

- **`always_ff`**: 順序回路にのみ使用します。ノンブロッキング代入 `<=` を使います
- **`always_comb`**: 組み合わせ回路にのみ使用します。ブロッキング代入 `=` を使います
- **`logic` 型の使用**: `reg` や `wire` の代わりに `logic` を使用し、ドライバ競合を検出します
- **ビット幅の明示**: 定数のビット幅を明示し、意図しないビット拡張を防ぎます
- **ANSIスタイルのポート宣言**: モジュールポートは常にANSIスタイルで記述します
- **名前付きポート接続**: インスタンス接続は位置ベースではなく名前ベースを使います

```systemverilog
// 推奨：名前付きポート接続
good_module #(.WIDTH(8)) u_mod (
    .clk      (sys_clk),
    .rst_n    (sys_rst_n),
    .data_in  (din),
    .data_out (dout)
);
// 非推奨：位置ベースの接続
// good_module u_mod (sys_clk, sys_rst_n, din, dout);
```

---

## 17.6 よくある落とし穴とアンチパターン

### 17.6.1 ラッチの意図しない推論

`always_comb` 内ですべての条件分岐に出力値を代入しないと、ラッチが推論されます。

```systemverilog
// 悪い例：2'b11のケースが欠落 → ラッチ推論！
always_comb begin
    case (sel)
        2'b00: out = a;
        2'b01: out = b;
        2'b10: out = c;
    endcase
end

// 良い例1：先頭でデフォルト値を設定
always_comb begin
    out = '0;  // デフォルト値
    case (sel)
        2'b00: out = a;  2'b01: out = b;  2'b10: out = c;
    endcase
end

// 良い例2：defaultケースを追加
always_comb begin
    case (sel)
        2'b00: out = a;  2'b01: out = b;
        2'b10: out = c;  default: out = '0;
    endcase
end
```

### 17.6.2 レース条件

ブロッキング代入とノンブロッキング代入の不適切な使用はレース条件を引き起こします。

```systemverilog
// 悪い例：順序回路でブロッキング代入
always_ff @(posedge clk) begin
    b = a;   // bの新しい値をcが使ってしまう可能性
    c = b;
end

// 良い例：順序回路ではノンブロッキング代入
always_ff @(posedge clk) begin
    b <= a;  // bの古い値がcに使われる（正しい動作）
    c <= b;
end
```

| 回路種別 | 代入方式 | 理由 |
|---------|---------|------|
| 順序回路（`always_ff`） | ノンブロッキング `<=` | 評価順序に依存しない |
| 組み合わせ回路（`always_comb`） | ブロッキング `=` | 代入順序が論理的に重要 |

### 17.6.3 不適切なリセット処理

リセットで一部のレジスタを初期化し忘れると、シミュレーションと実機で異なる動作を引き起こします。

```systemverilog
// 悪い例：valid_outのリセットが漏れている
always_ff @(posedge clk or negedge rst_n)
    if (!rst_n) data_out <= '0;       // valid_outが未初期化！
    else begin data_out <= data_in; valid_out <= valid_in; end

// 良い例：すべてのレジスタをリセット
always_ff @(posedge clk or negedge rst_n)
    if (!rst_n) begin data_out <= '0; valid_out <= 1'b0; end
    else        begin data_out <= data_in; valid_out <= valid_in; end
```

### 17.6.4 その他の一般的なアンチパターン

- **マルチドライバ**: 同じ信号を複数の `always` ブロックでドライブしてはなりません
- **非同期リセットの同期化忘れ**: 外部リセット信号はダブルフリップフロップで同期化が必要です
- **クロックドメイン交差（CDC）の無視**: 異なるクロックドメイン間にはシンクロナイザが必要です
- **`initial` ブロックでのハードウェア記述**: `initial` は合成不可（テストベンチ専用）です

![よくある落とし穴の分類](/images/systemverilog-complete-guide/ch16_common_pitfalls.drawio.png)

---

## 17.7 UVM（Universal Verification Methodology）概要

### 17.7.1 UVMとは

**UVM（Universal Verification Methodology）**は、Accellera Systems Initiative が策定した業界標準の検証方法論です。SystemVerilogのクラスベース検証機能を体系化し、再利用可能な検証環境の構築フレームワークを提供します。

- **標準化されたアーキテクチャ**: コンポーネントの構成と接続方法が統一されています
- **ファクトリメカニズム**: テスト時にオブジェクトの型を差し替えられます
- **フェーズ機構**: `build_phase`, `connect_phase`, `run_phase` 等のライフサイクルが定義されています
- **シーケンスメカニズム**: テストシナリオをシーケンスとして構造化し再利用できます
- **TLM（Transaction Level Modeling）**: コンポーネント間通信をトランザクションレベルで抽象化します

![UVMアーキテクチャ](/images/systemverilog-complete-guide/ch16_uvm_architecture.drawio.png)

### 17.7.2 UVMコンポーネント階層

| コンポーネント | 基底クラス | 役割 |
|-------------|-----------|------|
| **テスト** | `uvm_test` | テストシナリオの定義、環境の構成 |
| **環境（env）** | `uvm_env` | エージェントやスコアボードの集約 |
| **エージェント** | `uvm_agent` | ドライバ、シーケンサ、モニタの集約 |
| **ドライバ** | `uvm_driver` | シーケンスアイテムをピンレベルに変換 |
| **シーケンサ** | `uvm_sequencer` | シーケンスとドライバの仲介 |
| **モニタ** | `uvm_monitor` | DUT信号の監視、トランザクション再構成 |
| **スコアボード** | `uvm_scoreboard` | 結果の照合と検証 |

### 17.7.3 UVMの基本的なコード例

```systemverilog
// UVMシーケンスアイテム
class my_seq_item extends uvm_sequence_item;
    `uvm_object_utils(my_seq_item)
    rand bit [7:0] addr;
    rand bit [31:0] data;
    rand bit write;
    function new(string name = "my_seq_item"); super.new(name); endfunction
endclass

// UVMドライバ
class my_driver extends uvm_driver #(my_seq_item);
    `uvm_component_utils(my_driver)
    virtual bus_if vif;
    function new(string name, uvm_component parent); super.new(name, parent); endfunction

    virtual task run_phase(uvm_phase phase);
        my_seq_item item;
        forever begin
            seq_item_port.get_next_item(item);
            @(posedge vif.clk);
            vif.addr <= item.addr; vif.data <= item.data;
            vif.valid <= 1'b1;
            @(posedge vif.clk);
            vif.valid <= 1'b0;
            seq_item_port.item_done();
        end
    endtask
endclass

// UVMテスト
class my_test extends uvm_test;
    `uvm_component_utils(my_test)
    my_env env;
    function new(string name, uvm_component parent); super.new(name, parent); endfunction

    virtual function void build_phase(uvm_phase phase);
        super.build_phase(phase);
        env = my_env::type_id::create("env", this); // ファクトリ経由で生成
    endfunction

    virtual task run_phase(uvm_phase phase);
        my_sequence seq;
        phase.raise_objection(this);
        seq = my_sequence::type_id::create("seq");
        seq.start(env.agent.sequencer);
        phase.drop_objection(this);
    endtask
endclass
```

### 17.7.4 UVMフェーズ

| フェーズ | 種類 | 説明 |
|---------|------|------|
| `build_phase` | トップダウン | コンポーネントの生成 |
| `connect_phase` | ボトムアップ | TLMポート接続 |
| `run_phase` | 並行実行 | メインのシミュレーション実行 |
| `check_phase` | ボトムアップ | 結果の検証 |
| `report_phase` | ボトムアップ | レポート生成 |

---

## 17.8 デザインパターン適用のガイドライン

### 17.8.1 パターン選択の基準

| 要件 | 推奨パターン | 理由 |
|------|-------------|------|
| 制御フロー | FSMパターン | 状態遷移が明確に管理できる |
| 高スループット | パイプラインパターン | 各ステージが並行動作し性能向上 |
| モジュール間通信 | ハンドシェイクパターン | データの欠落や重複を防止 |
| 速度差の吸収 | FIFOパターン | プロデューサ/コンシューマの速度差を吸収 |
| 共有リソース | アービタパターン | 公平で安全なリソース共有を実現 |
| テスト拡張性 | コールバックパターン | テスト固有のカスタマイズが容易 |
| 型の切り替え | ファクトリパターン | 実行時に生成するオブジェクトを変更可能 |

### 17.8.2 アンチパターンの回避チェックリスト

設計レビューで確認すべきポイントです。

- **ラッチ推論チェック**: `always_comb` 内の全出力に全条件でデフォルト値が設定されているか
- **リセット漏れチェック**: `always_ff` 内の全レジスタがリセット条件で初期化されているか
- **代入方式チェック**: `always_ff` ではノンブロッキング、`always_comb` ではブロッキングを使用しているか
- **マルチドライバチェック**: 各信号のドライバが1箇所のみか
- **クロックドメイン交差チェック**: 異なるクロックドメイン間の信号に同期化処理があるか
- **`default` ケースチェック**: `case` 文に `default` があるか、または `unique case` を使用しているか

---

## 17.9 まとめ

本章では、SystemVerilogにおけるデザインパターンと実践的なコーディング技法について学びました。主要なポイントを振り返ります。

- **RTLデザインパターン**: FSM（2プロセス方式）、パイプライン、ハンドシェイクプロトコル、FIFO、アービタなど、合成可能な回路の定番パターンを理解しました。これらを正しく実装することで、信頼性の高いハードウェアを設計できます
- **検証デザインパターン**: 階層化テストベンチ、ドライバ/モニタ/スコアボード、ファクトリパターン、コールバックパターンなど、検証環境のアーキテクチャパターンを学びました。これらにより再利用性と拡張性の高い検証環境を構築できます
- **再利用可能なコンポーネント設計**: パラメータ化モジュール、インターフェース、パッケージを活用した再利用性の高い設計手法を学びました
- **コーディングベストプラクティス**: 命名規則、代入方式の使い分け、`always_ff`/`always_comb` の正しい使用など、保守性の高いコードを書くためのガイドラインを確認しました
- **よくある落とし穴**: ラッチ推論、レース条件、リセット漏れ、マルチドライバなど、頻出するバグのパターンとその回避方法を理解しました
- **UVM概要**: UVMのアーキテクチャ、フェーズ機構、基本コンポーネントの概念を学びました。UVMは業界標準の検証フレームワークであり、大規模プロジェクトの検証生産性を大幅に向上させます

デザインパターンは「正解」ではなく「道具」です。プロジェクトの要件、チームの規模、設計の複雑さに応じて、適切なパターンを選択・組み合わせることが重要です。パターンの本質を理解し、各パターンが解決する問題を意識して使い分けましょう。

次の第17章では、**SystemVerilogプロジェクト演習**として、本書で学んだ知識を総合的に活用する実践的なプロジェクトに取り組みます。RTL設計から検証環境の構築まで、一貫した開発フローを体験します。
