---
title: "第18章：プロジェクト演習"
---

# 第18章：プロジェクト演習

## 18.1 この章で学ぶこと

本章では、これまでに学んだSystemVerilogの知識を統合し、**実践的なプロジェクト演習**に取り組みます。3つのプロジェクト――UARTコントローラ、簡易CPU、SPIマスターコントローラ――を通じて、RTL設計から検証までの一連のフローを体験します。各プロジェクトでは仕様策定、RTLコーディング、テストベンチ作成、機能カバレッジの収集、デバッグまでを一貫して実施します。個々の章で学んだ技術がどのように組み合わさるかを実感し、実務に近い設計・検証スキルを身につけましょう。

![プロジェクト演習の全体構成](/images/systemverilog-complete-guide/ch17_project_overview.drawio.png)

---

## 18.2 プロジェクト演習の進め方

### 18.2.1 設計・検証フローの概要

各プロジェクトは以下のフローに従って進めます。

1. **仕様策定**: 機能仕様を明確化し、インターフェースを定義します
2. **アーキテクチャ設計**: モジュール分割とデータパスを決定します
3. **RTLコーディング**: SystemVerilogで各モジュールを実装します
4. **テストベンチ作成**: 制約付きランダムテストとダイレクトテストを構築します
5. **シミュレーション実行**: テストを実行し、波形とログで動作を確認します
6. **カバレッジ収集・分析**: 機能カバレッジとコードカバレッジの達成率を確認します
7. **デバッグ・修正**: 不具合を発見した場合は原因を特定し修正します

![設計・検証フロー](/images/systemverilog-complete-guide/ch17_design_verification_flow.drawio.png)

### 18.2.2 検証目標の設定

プロジェクト演習では、以下のカバレッジ目標を設定します。

| カバレッジ種類 | 目標値 | 説明 |
|--------------|--------|------|
| **コードカバレッジ（行）** | 95%以上 | RTLコードの行がどれだけ実行されたか |
| **コードカバレッジ（分岐）** | 90%以上 | if/case文のすべての分岐が通過したか |
| **機能カバレッジ** | 100% | 仕様上の検証シナリオをすべて網羅したか |
| **アサーション成功率** | 100% | すべてのアサーションが違反なしで完了したか |

### 18.2.3 ファイル構成の推奨

各プロジェクトで以下のディレクトリ構成を推奨します。

```
project_name/
├── rtl/                  # RTLソースファイル
│   ├── top_module.sv
│   └── sub_modules.sv
├── tb/                   # テストベンチ
│   ├── tb_top.sv
│   ├── env.sv
│   └── test.sv
├── sim/                  # シミュレーション用スクリプト
│   └── Makefile
└── doc/                  # 仕様書・設計メモ
    └── spec.md
```

---

## 18.3 プロジェクト1：UARTコントローラ

### 18.3.1 UARTの仕様

**UART（Universal Asynchronous Receiver/Transmitter）**は、非同期シリアル通信の標準的なインターフェースです。本プロジェクトでは以下の仕様でUARTコントローラを設計します。

- **データ幅**: 8ビット
- **ボーレート**: 9600 / 115200 bps（パラメータで切り替え可能）
- **パリティ**: なし / 偶数 / 奇数（設定可能）
- **ストップビット**: 1ビット
- **FIFO**: 送信・受信それぞれ8段のFIFO
- **クロック周波数**: 50 MHz

![UARTフレームフォーマット](/images/systemverilog-complete-guide/ch17_uart_frame_format.drawio.png)

UARTフレームは以下の構成です。

| フィールド | ビット数 | 説明 |
|-----------|---------|------|
| **スタートビット** | 1 | 常に0（通信開始を示す） |
| **データビット** | 8 | LSBファースト |
| **パリティビット** | 0または1 | 偶数/奇数パリティ |
| **ストップビット** | 1 | 常に1（通信終了を示す） |

### 18.3.2 モジュール構成

UARTコントローラは以下のサブモジュールで構成します。

- **uart_top**: トップモジュール（TX/RXの統合とレジスタインターフェース）
- **uart_tx**: 送信モジュール
- **uart_rx**: 受信モジュール
- **baud_gen**: ボーレートジェネレータ
- **uart_fifo**: 送信/受信用FIFO

![UARTモジュール構成](/images/systemverilog-complete-guide/ch17_uart_module_structure.drawio.png)

### 18.3.3 ボーレートジェネレータ

ボーレートジェネレータは、システムクロックを分周してUART通信のタイミング信号を生成します。

```systemverilog
module baud_gen #(
    parameter int CLK_FREQ  = 50_000_000,  // システムクロック周波数
    parameter int BAUD_RATE = 115200        // ボーレート
)(
    input  logic clk,
    input  logic rst_n,
    output logic tick        // ボーレートタイミング信号
);

    // 分周カウンタの上限値を計算
    localparam int DIVISOR = CLK_FREQ / (BAUD_RATE * 16);
    localparam int CNT_WIDTH = $clog2(DIVISOR);

    logic [CNT_WIDTH-1:0] counter;

    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            counter <= '0;
            tick    <= 1'b0;
        end else if (counter == DIVISOR - 1) begin
            counter <= '0;
            tick    <= 1'b1;  // 1クロック幅のパルス
        end else begin
            counter <= counter + 1'b1;
            tick    <= 1'b0;
        end
    end

endmodule
```

### 18.3.4 UART送信モジュール

送信モジュールは、パラレルデータをシリアルに変換して出力します。

```systemverilog
module uart_tx #(
    parameter int DATA_BITS = 8  // データビット数
)(
    input  logic                 clk,
    input  logic                 rst_n,
    input  logic                 tick,       // ボーレートtick
    input  logic                 tx_start,   // 送信開始信号
    input  logic [DATA_BITS-1:0] tx_data,    // 送信データ
    output logic                 tx,         // シリアル出力
    output logic                 tx_busy     // 送信中フラグ
);

    // 送信ステートマシン
    typedef enum logic [2:0] {
        IDLE,       // アイドル状態
        START,      // スタートビット送信
        DATA,       // データビット送信
        PARITY,     // パリティビット送信（未使用時スキップ）
        STOP        // ストップビット送信
    } tx_state_t;

    tx_state_t state, next_state;

    logic [3:0]          tick_cnt;    // 16倍オーバーサンプリングカウンタ
    logic [2:0]          bit_cnt;     // 送信ビットカウンタ
    logic [DATA_BITS-1:0] shift_reg;  // シフトレジスタ

    // 状態遷移（順序回路）
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n)
            state <= IDLE;
        else
            state <= next_state;
    end

    // 次状態論理（組み合わせ回路）
    always_comb begin
        next_state = state;
        case (state)
            IDLE: begin
                if (tx_start)
                    next_state = START;
            end
            START: begin
                if (tick && tick_cnt == 4'd15)
                    next_state = DATA;
            end
            DATA: begin
                if (tick && tick_cnt == 4'd15 && bit_cnt == DATA_BITS - 1)
                    next_state = STOP;
            end
            STOP: begin
                if (tick && tick_cnt == 4'd15)
                    next_state = IDLE;
            end
            default: next_state = IDLE;
        endcase
    end

    // データパス（カウンタ・シフトレジスタ制御）
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            tx        <= 1'b1;   // アイドル時はHigh
            tick_cnt  <= '0;
            bit_cnt   <= '0;
            shift_reg <= '0;
            tx_busy   <= 1'b0;
        end else begin
            case (state)
                IDLE: begin
                    tx      <= 1'b1;
                    tx_busy <= 1'b0;
                    if (tx_start) begin
                        shift_reg <= tx_data;
                        tx_busy   <= 1'b1;
                    end
                end
                START: begin
                    tx <= 1'b0;  // スタートビット = 0
                    if (tick)
                        tick_cnt <= tick_cnt + 1'b1;
                end
                DATA: begin
                    tx <= shift_reg[0];  // LSBから送信
                    if (tick) begin
                        tick_cnt <= tick_cnt + 1'b1;
                        if (tick_cnt == 4'd15) begin
                            shift_reg <= shift_reg >> 1;
                            bit_cnt   <= bit_cnt + 1'b1;
                        end
                    end
                end
                STOP: begin
                    tx <= 1'b1;  // ストップビット = 1
                    if (tick) begin
                        tick_cnt <= tick_cnt + 1'b1;
                        if (tick_cnt == 4'd15) begin
                            tick_cnt <= '0;
                            bit_cnt  <= '0;
                        end
                    end
                end
            endcase
        end
    end

endmodule
```

### 18.3.5 UART受信モジュール

受信モジュールは、シリアル入力をパラレルデータに変換します。16倍オーバーサンプリングにより、ビットの中央でデータをサンプリングします。

```systemverilog
module uart_rx #(
    parameter int DATA_BITS = 8  // データビット数
)(
    input  logic                 clk,
    input  logic                 rst_n,
    input  logic                 tick,       // ボーレートtick
    input  logic                 rx,         // シリアル入力
    output logic [DATA_BITS-1:0] rx_data,    // 受信データ
    output logic                 rx_valid    // 受信完了信号
);

    typedef enum logic [1:0] {
        IDLE,    // アイドル状態
        START,   // スタートビット検出
        DATA,    // データ受信中
        STOP     // ストップビット確認
    } rx_state_t;

    rx_state_t state, next_state;

    logic [3:0]          tick_cnt;     // オーバーサンプリングカウンタ
    logic [2:0]          bit_cnt;      // 受信ビットカウンタ
    logic [DATA_BITS-1:0] shift_reg;   // シフトレジスタ

    // 状態遷移
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n)
            state <= IDLE;
        else
            state <= next_state;
    end

    // 次状態論理
    always_comb begin
        next_state = state;
        case (state)
            IDLE: begin
                if (~rx)  // スタートビット（Low）検出
                    next_state = START;
            end
            START: begin
                if (tick && tick_cnt == 4'd7) begin
                    if (~rx)
                        next_state = DATA;  // ビット中央で確認
                    else
                        next_state = IDLE;  // ノイズと判断
                end
            end
            DATA: begin
                if (tick && tick_cnt == 4'd15 && bit_cnt == DATA_BITS - 1)
                    next_state = STOP;
            end
            STOP: begin
                if (tick && tick_cnt == 4'd15)
                    next_state = IDLE;
            end
        endcase
    end

    // データパス
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            tick_cnt  <= '0;
            bit_cnt   <= '0;
            shift_reg <= '0;
            rx_data   <= '0;
            rx_valid  <= 1'b0;
        end else begin
            rx_valid <= 1'b0;  // デフォルトでLow
            case (state)
                IDLE: begin
                    tick_cnt <= '0;
                    bit_cnt  <= '0;
                end
                START: begin
                    if (tick)
                        tick_cnt <= tick_cnt + 1'b1;
                    if (tick && tick_cnt == 4'd7)
                        tick_cnt <= '0;
                end
                DATA: begin
                    if (tick) begin
                        tick_cnt <= tick_cnt + 1'b1;
                        if (tick_cnt == 4'd15) begin
                            // ビット中央（8カウント目）でサンプリング
                            shift_reg <= {rx, shift_reg[DATA_BITS-1:1]};
                            bit_cnt   <= bit_cnt + 1'b1;
                        end
                    end
                end
                STOP: begin
                    if (tick && tick_cnt == 4'd15) begin
                        rx_data  <= shift_reg;  // 受信データを出力
                        rx_valid <= 1'b1;       // 受信完了
                    end
                    if (tick)
                        tick_cnt <= tick_cnt + 1'b1;
                end
            endcase
        end
    end

endmodule
```

### 18.3.6 UARTテストベンチ

制約付きランダムテストを用いたテストベンチです。送信データをランダムに生成し、ループバック接続（TX出力をRX入力に接続）で正しく受信できることを確認します。

```systemverilog
module tb_uart;

    // パラメータ
    localparam int CLK_FREQ  = 50_000_000;
    localparam int BAUD_RATE = 115200;
    localparam int DATA_BITS = 8;

    // 信号宣言
    logic clk, rst_n;
    logic tx_start;
    logic [DATA_BITS-1:0] tx_data;
    logic tx_serial;        // TX→RXループバック
    logic [DATA_BITS-1:0] rx_data;
    logic rx_valid;
    logic tx_busy;
    logic tick;

    // クロック生成（50MHz = 20ns周期）
    initial clk = 0;
    always #10 clk = ~clk;

    // DUTインスタンス化
    baud_gen #(
        .CLK_FREQ(CLK_FREQ),
        .BAUD_RATE(BAUD_RATE)
    ) u_baud (
        .clk(clk), .rst_n(rst_n), .tick(tick)
    );

    uart_tx #(.DATA_BITS(DATA_BITS)) u_tx (
        .clk(clk), .rst_n(rst_n), .tick(tick),
        .tx_start(tx_start), .tx_data(tx_data),
        .tx(tx_serial), .tx_busy(tx_busy)
    );

    uart_rx #(.DATA_BITS(DATA_BITS)) u_rx (
        .clk(clk), .rst_n(rst_n), .tick(tick),
        .rx(tx_serial),  // ループバック接続
        .rx_data(rx_data), .rx_valid(rx_valid)
    );

    // カバレッジグループ
    covergroup uart_cov @(posedge clk);
        // 送信データの値域カバレッジ
        tx_data_cp: coverpoint tx_data {
            bins zero     = {8'h00};
            bins low      = {[8'h01:8'h3F]};
            bins mid      = {[8'h40:8'hBF]};
            bins high     = {[8'hC0:8'hFE]};
            bins all_ones = {8'hFF};
        }
        // 受信完了のカバレッジ
        rx_valid_cp: coverpoint rx_valid;
    endgroup

    uart_cov cov = new();

    // テストシーケンス
    int pass_count = 0;
    int fail_count = 0;

    task automatic send_and_check(input logic [DATA_BITS-1:0] data);
        // 送信開始
        @(posedge clk);
        tx_data  = data;
        tx_start = 1'b1;
        @(posedge clk);
        tx_start = 1'b0;

        // 受信完了を待機
        wait (rx_valid);
        @(posedge clk);

        // 結果チェック
        if (rx_data === data) begin
            pass_count++;
            $display("[PASS] TX=0x%02h, RX=0x%02h", data, rx_data);
        end else begin
            fail_count++;
            $error("[FAIL] TX=0x%02h, RX=0x%02h", data, rx_data);
        end

        // 次の送信まで待機
        wait (!tx_busy);
        repeat (100) @(posedge clk);
    endtask

    initial begin
        // リセット
        rst_n    = 1'b0;
        tx_start = 1'b0;
        tx_data  = '0;
        repeat (10) @(posedge clk);
        rst_n = 1'b1;
        repeat (10) @(posedge clk);

        // ダイレクトテスト（境界値）
        $display("=== ダイレクトテスト ===");
        send_and_check(8'h00);
        send_and_check(8'hFF);
        send_and_check(8'hA5);
        send_and_check(8'h5A);

        // ランダムテスト
        $display("=== ランダムテスト ===");
        for (int i = 0; i < 50; i++) begin
            logic [DATA_BITS-1:0] random_data;
            assert(std::randomize(random_data));
            send_and_check(random_data);
        end

        // 結果サマリ
        $display("=================================");
        $display("PASS: %0d, FAIL: %0d", pass_count, fail_count);
        $display("=================================");

        if (fail_count == 0)
            $display("ALL TESTS PASSED");
        else
            $fatal(1, "SOME TESTS FAILED");

        $finish;
    end

endmodule
```

### 18.3.7 期待される結果

テストが正常に完了すると、以下のような出力が得られます。

```
=== ダイレクトテスト ===
[PASS] TX=0x00, RX=0x00
[PASS] TX=0xff, RX=0xff
[PASS] TX=0xa5, RX=0xa5
[PASS] TX=0x5a, RX=0x5a
=== ランダムテスト ===
[PASS] TX=0x3c, RX=0x3c
[PASS] TX=0x87, RX=0x87
...（省略）...
=================================
PASS: 54, FAIL: 0
=================================
ALL TESTS PASSED
```

---

## 18.4 プロジェクト2：簡易CPU設計

### 18.4.1 命令セットアーキテクチャ（ISA）

本プロジェクトでは、8ビットの簡易CPUを設計します。教育目的のシンプルなアーキテクチャですが、フェッチ・デコード・実行の基本的なパイプライン概念を含みます。

| 項目 | 仕様 |
|------|------|
| **データ幅** | 8ビット |
| **アドレス幅** | 8ビット（256バイトメモリ） |
| **レジスタ** | 4本の汎用レジスタ（R0～R3） |
| **命令長** | 16ビット固定長 |
| **アーキテクチャ** | ロード/ストア型 |

命令フォーマットは以下の通りです。

```
R型命令（レジスタ間演算）:
[15:12] opcode | [11:10] rd | [9:8] rs1 | [7:6] rs2 | [5:0] reserved

I型命令（即値付き演算・メモリアクセス）:
[15:12] opcode | [11:10] rd | [9:8] rs1 | [7:0] imm
```

![簡易CPU命令フォーマット](/images/systemverilog-complete-guide/ch17_cpu_instruction_format.drawio.png)

命令セットの一覧です。

| オペコード | ニーモニック | 動作 | 型 |
|-----------|------------|------|-----|
| 4'b0000 | NOP | 何もしない | - |
| 4'b0001 | ADD | rd = rs1 + rs2 | R |
| 4'b0010 | SUB | rd = rs1 - rs2 | R |
| 4'b0011 | AND | rd = rs1 & rs2 | R |
| 4'b0100 | OR | rd = rs1 \| rs2 | R |
| 4'b0101 | XOR | rd = rs1 ^ rs2 | R |
| 4'b0110 | LOAD | rd = mem[rs1 + imm] | I |
| 4'b0111 | STORE | mem[rs1 + imm] = rd | I |
| 4'b1000 | ADDI | rd = rs1 + imm | I |
| 4'b1001 | BEQ | if (rd == rs1) PC = imm | I |
| 4'b1010 | BNE | if (rd != rs1) PC = imm | I |
| 4'b1111 | HALT | 実行停止 | - |

### 18.4.2 CPUモジュール構成

CPUは以下のサブモジュールで構成します。

- **cpu_top**: トップモジュール（各モジュールの接続）
- **alu**: 算術論理演算ユニット
- **reg_file**: レジスタファイル
- **control_unit**: 制御ユニット（命令デコード・制御信号生成）
- **program_counter**: プログラムカウンタ
- **inst_mem**: 命令メモリ（ROM）
- **data_mem**: データメモリ（RAM）

![簡易CPUブロック図](/images/systemverilog-complete-guide/ch17_cpu_block_diagram.drawio.png)

### 18.4.3 ALU（算術論理演算ユニット）

```systemverilog
module alu #(
    parameter int WIDTH = 8  // データ幅
)(
    input  logic [WIDTH-1:0] a,         // オペランドA
    input  logic [WIDTH-1:0] b,         // オペランドB
    input  logic [3:0]       alu_op,    // 演算種別
    output logic [WIDTH-1:0] result,    // 演算結果
    output logic             zero_flag  // ゼロフラグ
);

    // ALU演算コード
    localparam logic [3:0] ALU_ADD = 4'b0001;
    localparam logic [3:0] ALU_SUB = 4'b0010;
    localparam logic [3:0] ALU_AND = 4'b0011;
    localparam logic [3:0] ALU_OR  = 4'b0100;
    localparam logic [3:0] ALU_XOR = 4'b0101;

    always_comb begin
        case (alu_op)
            ALU_ADD: result = a + b;     // 加算
            ALU_SUB: result = a - b;     // 減算
            ALU_AND: result = a & b;     // 論理AND
            ALU_OR:  result = a | b;     // 論理OR
            ALU_XOR: result = a ^ b;     // 排他的OR
            default: result = '0;        // デフォルト値
        endcase
    end

    // ゼロフラグ：結果が0ならアサート
    assign zero_flag = (result == '0);

endmodule
```

### 18.4.4 レジスタファイル

```systemverilog
module reg_file #(
    parameter int WIDTH    = 8,  // データ幅
    parameter int NUM_REGS = 4   // レジスタ本数
)(
    input  logic                       clk,
    input  logic                       rst_n,
    // 読み出しポート1
    input  logic [$clog2(NUM_REGS)-1:0] rs1_addr,
    output logic [WIDTH-1:0]            rs1_data,
    // 読み出しポート2
    input  logic [$clog2(NUM_REGS)-1:0] rs2_addr,
    output logic [WIDTH-1:0]            rs2_data,
    // 書き込みポート
    input  logic                        wr_en,
    input  logic [$clog2(NUM_REGS)-1:0] rd_addr,
    input  logic [WIDTH-1:0]            rd_data
);

    // レジスタ配列
    logic [WIDTH-1:0] regs [NUM_REGS];

    // 読み出し（組み合わせ回路）
    assign rs1_data = regs[rs1_addr];
    assign rs2_data = regs[rs2_addr];

    // 書き込み（クロック同期）
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            for (int i = 0; i < NUM_REGS; i++)
                regs[i] <= '0;  // リセット時に全レジスタをクリア
        end else if (wr_en) begin
            regs[rd_addr] <= rd_data;
        end
    end

endmodule
```

### 18.4.5 制御ユニット

```systemverilog
module control_unit (
    input  logic [3:0]  opcode,     // 命令オペコード
    output logic        reg_write,  // レジスタ書き込み許可
    output logic        mem_read,   // メモリ読み出し
    output logic        mem_write,  // メモリ書き込み
    output logic        alu_src,    // ALUソース選択（0:レジスタ, 1:即値）
    output logic        branch,     // 分岐命令フラグ
    output logic        mem_to_reg, // メモリ→レジスタ選択
    output logic [3:0]  alu_op,     // ALU演算種別
    output logic        halt        // 停止信号
);

    always_comb begin
        // デフォルト値
        reg_write  = 1'b0;
        mem_read   = 1'b0;
        mem_write  = 1'b0;
        alu_src    = 1'b0;
        branch     = 1'b0;
        mem_to_reg = 1'b0;
        alu_op     = 4'b0000;
        halt       = 1'b0;

        case (opcode)
            4'b0000: begin // NOP
                // 何もしない
            end
            4'b0001: begin // ADD
                reg_write = 1'b1;
                alu_op    = 4'b0001;
            end
            4'b0010: begin // SUB
                reg_write = 1'b1;
                alu_op    = 4'b0010;
            end
            4'b0011: begin // AND
                reg_write = 1'b1;
                alu_op    = 4'b0011;
            end
            4'b0100: begin // OR
                reg_write = 1'b1;
                alu_op    = 4'b0100;
            end
            4'b0101: begin // XOR
                reg_write = 1'b1;
                alu_op    = 4'b0101;
            end
            4'b0110: begin // LOAD
                reg_write  = 1'b1;
                mem_read   = 1'b1;
                alu_src    = 1'b1;
                mem_to_reg = 1'b1;
                alu_op     = 4'b0001;  // アドレス計算にADD使用
            end
            4'b0111: begin // STORE
                mem_write = 1'b1;
                alu_src   = 1'b1;
                alu_op    = 4'b0001;  // アドレス計算にADD使用
            end
            4'b1000: begin // ADDI
                reg_write = 1'b1;
                alu_src   = 1'b1;
                alu_op    = 4'b0001;
            end
            4'b1001: begin // BEQ
                branch = 1'b1;
                alu_op = 4'b0010;  // 比較にSUB使用
            end
            4'b1010: begin // BNE
                branch = 1'b1;
                alu_op = 4'b0010;
            end
            4'b1111: begin // HALT
                halt = 1'b1;
            end
            default: begin
                // 未定義命令：何もしない
            end
        endcase
    end

endmodule
```

### 18.4.6 CPUトップモジュール

```systemverilog
module cpu_top #(
    parameter int DATA_WIDTH = 8,
    parameter int ADDR_WIDTH = 8
)(
    input  logic clk,
    input  logic rst_n,
    output logic halted  // CPU停止状態
);

    // 内部信号
    logic [15:0]            instruction;
    logic [ADDR_WIDTH-1:0]  pc;
    logic [3:0]             opcode;
    logic [1:0]             rd_addr, rs1_addr, rs2_addr;
    logic [7:0]             imm;

    // 制御信号
    logic reg_write, mem_read, mem_write;
    logic alu_src, branch, mem_to_reg, halt;
    logic [3:0] alu_op;

    // データパス信号
    logic [DATA_WIDTH-1:0] rs1_data, rs2_data;
    logic [DATA_WIDTH-1:0] alu_a, alu_b, alu_result;
    logic                  zero_flag;
    logic [DATA_WIDTH-1:0] mem_data_out;
    logic [DATA_WIDTH-1:0] write_back_data;

    // 命令デコード
    assign opcode   = instruction[15:12];
    assign rd_addr  = instruction[11:10];
    assign rs1_addr = instruction[9:8];
    assign rs2_addr = instruction[7:6];
    assign imm      = instruction[7:0];

    // プログラムカウンタ
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n)
            pc <= '0;
        else if (halt)
            pc <= pc;  // 停止時はPCを保持
        else if (branch && zero_flag && opcode == 4'b1001)
            pc <= imm;  // BEQ: ゼロフラグが立っていれば分岐
        else if (branch && !zero_flag && opcode == 4'b1010)
            pc <= imm;  // BNE: ゼロフラグが立っていなければ分岐
        else
            pc <= pc + 1'b1;
    end

    // 命令メモリ（簡易実装：内部ROM）
    logic [15:0] inst_mem [256];
    assign instruction = inst_mem[pc];

    // ALUソース選択
    assign alu_a = rs1_data;
    assign alu_b = alu_src ? {{(DATA_WIDTH){1'b0}} | imm[DATA_WIDTH-1:0]}
                           : rs2_data;

    // ライトバックデータ選択
    assign write_back_data = mem_to_reg ? mem_data_out : alu_result;

    // 停止信号
    assign halted = halt;

    // サブモジュールのインスタンス化
    reg_file #(
        .WIDTH(DATA_WIDTH), .NUM_REGS(4)
    ) u_reg_file (
        .clk(clk), .rst_n(rst_n),
        .rs1_addr(rs1_addr), .rs1_data(rs1_data),
        .rs2_addr(rs2_addr), .rs2_data(rs2_data),
        .wr_en(reg_write), .rd_addr(rd_addr),
        .rd_data(write_back_data)
    );

    alu #(.WIDTH(DATA_WIDTH)) u_alu (
        .a(alu_a), .b(alu_b),
        .alu_op(alu_op),
        .result(alu_result), .zero_flag(zero_flag)
    );

    control_unit u_ctrl (
        .opcode(opcode),
        .reg_write(reg_write), .mem_read(mem_read),
        .mem_write(mem_write), .alu_src(alu_src),
        .branch(branch), .mem_to_reg(mem_to_reg),
        .alu_op(alu_op), .halt(halt)
    );

    // データメモリ
    logic [DATA_WIDTH-1:0] data_mem [256];

    always_ff @(posedge clk) begin
        if (mem_write)
            data_mem[alu_result] <= rs1_data;
    end

    assign mem_data_out = mem_read ? data_mem[alu_result] : '0;

endmodule
```

### 18.4.7 CPUテストベンチ

テストベンチでは、命令メモリにプログラムをロードし、期待される実行結果と比較します。

```systemverilog
module tb_cpu;

    logic clk, rst_n;
    logic halted;

    // クロック生成
    initial clk = 0;
    always #5 clk = ~clk;

    // DUTインスタンス化
    cpu_top #(
        .DATA_WIDTH(8), .ADDR_WIDTH(8)
    ) u_cpu (
        .clk(clk), .rst_n(rst_n), .halted(halted)
    );

    // テストプログラムのロード
    task automatic load_program();
        // プログラム: R0=5, R1=3を設定し、R2=R0+R1を計算
        // ADDI R0, R0, 5   (R0 = 0 + 5 = 5)
        u_cpu.inst_mem[0] = {4'b1000, 2'b00, 2'b00, 8'd5};
        // ADDI R1, R1, 3   (R1 = 0 + 3 = 3)
        u_cpu.inst_mem[1] = {4'b1000, 2'b01, 2'b01, 8'd3};
        // ADD R2, R0, R1   (R2 = 5 + 3 = 8)
        u_cpu.inst_mem[2] = {4'b0001, 2'b10, 2'b00, 2'b01, 6'b0};
        // SUB R3, R0, R1   (R3 = 5 - 3 = 2)
        u_cpu.inst_mem[3] = {4'b0010, 2'b11, 2'b00, 2'b01, 6'b0};
        // STORE R2, [R0+0] (mem[5] = 8)
        u_cpu.inst_mem[4] = {4'b0111, 2'b10, 2'b00, 8'd0};
        // LOAD R3, [R0+0]  (R3 = mem[5] = 8)
        u_cpu.inst_mem[5] = {4'b0110, 2'b11, 2'b00, 8'd0};
        // HALT
        u_cpu.inst_mem[6] = {4'b1111, 12'b0};
    endtask

    // 結果検証
    task automatic verify_results();
        int errors = 0;

        // R0 = 5
        if (u_cpu.u_reg_file.regs[0] !== 8'd5) begin
            $error("R0: 期待値=5, 実際=%0d", u_cpu.u_reg_file.regs[0]);
            errors++;
        end
        // R1 = 3
        if (u_cpu.u_reg_file.regs[1] !== 8'd3) begin
            $error("R1: 期待値=3, 実際=%0d", u_cpu.u_reg_file.regs[1]);
            errors++;
        end
        // R2 = 8
        if (u_cpu.u_reg_file.regs[2] !== 8'd8) begin
            $error("R2: 期待値=8, 実際=%0d", u_cpu.u_reg_file.regs[2]);
            errors++;
        end

        if (errors == 0)
            $display("[PASS] すべてのレジスタ値が正しいです");
        else
            $error("[FAIL] %0d 個のエラーが検出されました", errors);
    endtask

    // カバレッジ：命令種別の網羅
    covergroup cpu_cov @(posedge clk);
        opcode_cp: coverpoint u_cpu.opcode {
            bins nop   = {4'b0000};
            bins add   = {4'b0001};
            bins sub   = {4'b0010};
            bins and_  = {4'b0011};
            bins or_   = {4'b0100};
            bins xor_  = {4'b0101};
            bins load  = {4'b0110};
            bins store = {4'b0111};
            bins addi  = {4'b1000};
            bins beq   = {4'b1001};
            bins bne   = {4'b1010};
            bins halt  = {4'b1111};
        }
        alu_zero_cp: coverpoint u_cpu.zero_flag;
    endgroup

    cpu_cov cov = new();

    // メインテストシーケンス
    initial begin
        rst_n = 1'b0;
        repeat (5) @(posedge clk);
        rst_n = 1'b1;

        // プログラムロード
        load_program();

        // 実行完了（HALT）を待機
        wait (halted);
        repeat (2) @(posedge clk);

        // 結果検証
        verify_results();

        $display("=== CPUテスト完了 ===");
        $finish;
    end

    // タイムアウト監視
    initial begin
        #100_000;
        $fatal(1, "テストがタイムアウトしました");
    end

endmodule
```

### 18.4.8 期待される結果

```
[PASS] すべてのレジスタ値が正しいです
=== CPUテスト完了 ===
```

レジスタの最終状態は以下の通りです。

| レジスタ | 期待値 | 説明 |
|---------|--------|------|
| R0 | 5 | ADDI R0, R0, 5 |
| R1 | 3 | ADDI R1, R1, 3 |
| R2 | 8 | ADD R2, R0, R1 (5+3) |
| R3 | 8 | LOAD R3, [R0+0] (mem[5]の値) |

---

## 18.5 プロジェクト3：SPIマスターコントローラ

### 18.5.1 SPIプロトコルの概要

**SPI（Serial Peripheral Interface）**は、マスター/スレーブ方式の同期式シリアル通信インターフェースです。4本の信号線で全二重通信を行います。

| 信号名 | 方向 | 説明 |
|--------|------|------|
| **SCLK** | マスター→スレーブ | シリアルクロック |
| **MOSI** | マスター→スレーブ | Master Out Slave In（送信データ） |
| **MISO** | スレーブ→マスター | Master In Slave Out（受信データ） |
| **CS_N** | マスター→スレーブ | チップセレクト（アクティブLow） |

![SPIバス接続図](/images/systemverilog-complete-guide/ch17_spi_bus_connection.drawio.png)

本プロジェクトでは以下の仕様でSPIマスターを設計します。

- **データ幅**: 8ビット
- **クロック極性（CPOL）**: 0または1（設定可能）
- **クロック位相（CPHA）**: 0または1（設定可能）
- **クロック分周比**: 2, 4, 8, 16, 32, 64, 128, 256（設定可能）
- **ビット順序**: MSBファースト

SPIには4つの動作モードがあります。

| モード | CPOL | CPHA | クロックアイドル | データサンプリング |
|--------|------|------|--------------|----------------|
| Mode 0 | 0 | 0 | Low | 立ち上がりエッジ |
| Mode 1 | 0 | 1 | Low | 立ち下がりエッジ |
| Mode 2 | 1 | 0 | High | 立ち下がりエッジ |
| Mode 3 | 1 | 1 | High | 立ち上がりエッジ |

### 18.5.2 SPIマスター RTL設計

```systemverilog
module spi_master #(
    parameter int DATA_WIDTH = 8  // データ幅
)(
    input  logic                   clk,
    input  logic                   rst_n,
    // 制御インターフェース
    input  logic                   start,      // 転送開始
    input  logic [DATA_WIDTH-1:0]  tx_data,    // 送信データ
    output logic [DATA_WIDTH-1:0]  rx_data,    // 受信データ
    output logic                   busy,       // 転送中フラグ
    output logic                   done,       // 転送完了パルス
    // SPI設定
    input  logic                   cpol,       // クロック極性
    input  logic                   cpha,       // クロック位相
    input  logic [7:0]             clk_div,    // クロック分周値
    // SPIバス信号
    output logic                   sclk,       // SPIクロック
    output logic                   mosi,       // マスター出力
    input  logic                   miso,       // マスター入力
    output logic                   cs_n        // チップセレクト
);

    // ステートマシン
    typedef enum logic [2:0] {
        S_IDLE,       // アイドル状態
        S_CS_SETUP,   // CSセットアップ時間
        S_TRANSFER,   // データ転送中
        S_CS_HOLD,    // CSホールド時間
        S_DONE        // 転送完了
    } spi_state_t;

    spi_state_t state, next_state;

    logic [7:0]              clk_cnt;      // クロック分周カウンタ
    logic                    sclk_en;      // SCLKイネーブル
    logic [$clog2(DATA_WIDTH)-1:0] bit_cnt; // ビットカウンタ
    logic [DATA_WIDTH-1:0]   tx_shift;     // 送信シフトレジスタ
    logic [DATA_WIDTH-1:0]   rx_shift;     // 受信シフトレジスタ
    logic                    sclk_int;     // 内部SCLKトグル
    logic                    sample_edge;  // サンプリングエッジ
    logic                    shift_edge;   // シフトエッジ

    // クロック分周
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            clk_cnt  <= '0;
            sclk_en  <= 1'b0;
        end else if (state == S_TRANSFER) begin
            if (clk_cnt == clk_div - 1) begin
                clk_cnt <= '0;
                sclk_en <= 1'b1;
            end else begin
                clk_cnt <= clk_cnt + 1'b1;
                sclk_en <= 1'b0;
            end
        end else begin
            clk_cnt <= '0;
            sclk_en <= 1'b0;
        end
    end

    // SCLKの生成
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n)
            sclk_int <= 1'b0;
        else if (state == S_TRANSFER && sclk_en)
            sclk_int <= ~sclk_int;
        else if (state == S_IDLE)
            sclk_int <= 1'b0;
    end

    // CPOL適用
    assign sclk = cpol ? ~sclk_int : sclk_int;

    // サンプリング/シフトエッジの決定
    assign sample_edge = sclk_en & (cpha ? sclk_int : ~sclk_int);
    assign shift_edge  = sclk_en & (cpha ? ~sclk_int : sclk_int);

    // 状態遷移
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n)
            state <= S_IDLE;
        else
            state <= next_state;
    end

    // 次状態論理
    always_comb begin
        next_state = state;
        case (state)
            S_IDLE: begin
                if (start)
                    next_state = S_CS_SETUP;
            end
            S_CS_SETUP: begin
                next_state = S_TRANSFER;  // 1クロックのセットアップ
            end
            S_TRANSFER: begin
                if (shift_edge && bit_cnt == DATA_WIDTH - 1)
                    next_state = S_CS_HOLD;
            end
            S_CS_HOLD: begin
                next_state = S_DONE;
            end
            S_DONE: begin
                next_state = S_IDLE;
            end
            default: next_state = S_IDLE;
        endcase
    end

    // データパス
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            tx_shift <= '0;
            rx_shift <= '0;
            bit_cnt  <= '0;
            cs_n     <= 1'b1;  // 非選択
            mosi     <= 1'b0;
            rx_data  <= '0;
            busy     <= 1'b0;
            done     <= 1'b0;
        end else begin
            done <= 1'b0;  // デフォルトで非アクティブ
            case (state)
                S_IDLE: begin
                    cs_n <= 1'b1;
                    busy <= 1'b0;
                    if (start) begin
                        tx_shift <= tx_data;
                        busy     <= 1'b1;
                    end
                end
                S_CS_SETUP: begin
                    cs_n <= 1'b0;  // チップセレクトアサート
                    mosi <= tx_shift[DATA_WIDTH-1];  // MSBを出力
                end
                S_TRANSFER: begin
                    // データサンプリング（受信）
                    if (sample_edge)
                        rx_shift <= {rx_shift[DATA_WIDTH-2:0], miso};

                    // データシフト（送信）
                    if (shift_edge) begin
                        tx_shift <= {tx_shift[DATA_WIDTH-2:0], 1'b0};
                        mosi     <= tx_shift[DATA_WIDTH-2];
                        bit_cnt  <= bit_cnt + 1'b1;
                    end
                end
                S_CS_HOLD: begin
                    rx_data <= rx_shift;  // 受信データを確定
                end
                S_DONE: begin
                    cs_n    <= 1'b1;  // チップセレクトデアサート
                    done    <= 1'b1;  // 完了パルス
                    bit_cnt <= '0;
                end
            endcase
        end
    end

endmodule
```

### 18.5.3 SPIスレーブモデル（検証用）

テストベンチで使用するスレーブモデルです。マスターからの送信データをエコーバックします。

```systemverilog
module spi_slave_model #(
    parameter int DATA_WIDTH = 8
)(
    input  logic                   sclk,
    input  logic                   cs_n,
    input  logic                   mosi,
    output logic                   miso,
    input  logic                   cpol,
    input  logic                   cpha,
    // モニタ用出力
    output logic [DATA_WIDTH-1:0]  received_data,
    output logic                   data_valid
);

    logic [DATA_WIDTH-1:0] rx_shift;
    logic [DATA_WIDTH-1:0] tx_shift;
    logic [$clog2(DATA_WIDTH)-1:0] bit_cnt;
    logic sample_clk, shift_clk;

    // CPOL/CPHAに応じたエッジ選択
    assign sample_clk = (cpol ^ cpha) ? ~sclk : sclk;  // サンプリングエッジ
    assign shift_clk  = (cpol ^ cpha) ? sclk : ~sclk;   // シフトエッジ

    // MISO出力（MSBファースト）
    assign miso = cs_n ? 1'bz : tx_shift[DATA_WIDTH-1];

    // 受信処理
    always @(posedge sample_clk or posedge cs_n) begin
        if (cs_n) begin
            bit_cnt    <= '0;
            data_valid <= 1'b0;
        end else begin
            rx_shift <= {rx_shift[DATA_WIDTH-2:0], mosi};
            bit_cnt  <= bit_cnt + 1'b1;
            if (bit_cnt == DATA_WIDTH - 1) begin
                received_data <= {rx_shift[DATA_WIDTH-2:0], mosi};
                data_valid    <= 1'b1;
            end
        end
    end

    // 送信処理（受信データをエコーバック）
    always @(negedge cs_n) begin
        tx_shift <= received_data;  // 前回の受信データを送信
    end

    always @(posedge shift_clk) begin
        if (!cs_n)
            tx_shift <= {tx_shift[DATA_WIDTH-2:0], 1'b0};
    end

endmodule
```

### 18.5.4 SPIテストベンチ

4つのSPIモードすべてをテストし、全二重通信の正しさを確認します。

```systemverilog
module tb_spi;

    // パラメータ
    localparam int DATA_WIDTH = 8;
    localparam int CLK_DIV    = 4;

    // 信号宣言
    logic clk, rst_n;
    logic start;
    logic [DATA_WIDTH-1:0] tx_data, rx_data;
    logic busy, done;
    logic cpol, cpha;
    logic [7:0] clk_div;
    logic sclk, mosi, miso, cs_n;

    // スレーブモデルの信号
    logic [DATA_WIDTH-1:0] slave_rx_data;
    logic slave_data_valid;

    // クロック生成
    initial clk = 0;
    always #5 clk = ~clk;

    // DUTインスタンス化
    spi_master #(.DATA_WIDTH(DATA_WIDTH)) u_master (
        .clk(clk), .rst_n(rst_n),
        .start(start), .tx_data(tx_data), .rx_data(rx_data),
        .busy(busy), .done(done),
        .cpol(cpol), .cpha(cpha), .clk_div(clk_div),
        .sclk(sclk), .mosi(mosi), .miso(miso), .cs_n(cs_n)
    );

    // スレーブモデルインスタンス化
    spi_slave_model #(.DATA_WIDTH(DATA_WIDTH)) u_slave (
        .sclk(sclk), .cs_n(cs_n),
        .mosi(mosi), .miso(miso),
        .cpol(cpol), .cpha(cpha),
        .received_data(slave_rx_data),
        .data_valid(slave_data_valid)
    );

    // カバレッジ
    covergroup spi_cov @(posedge done);
        // SPIモードのカバレッジ
        mode_cp: coverpoint {cpol, cpha} {
            bins mode0 = {2'b00};
            bins mode1 = {2'b01};
            bins mode2 = {2'b10};
            bins mode3 = {2'b11};
        }
        // 送信データの範囲
        tx_data_cp: coverpoint tx_data {
            bins zero     = {8'h00};
            bins low      = {[8'h01:8'h7F]};
            bins high     = {[8'h80:8'hFE]};
            bins all_ones = {8'hFF};
        }
        // モード×データのクロスカバレッジ
        mode_data_cross: cross mode_cp, tx_data_cp;
    endgroup

    spi_cov cov = new();

    // 1回のSPI転送を実行するタスク
    task automatic spi_transfer(
        input  logic [DATA_WIDTH-1:0] data,
        output logic [DATA_WIDTH-1:0] received
    );
        @(posedge clk);
        tx_data = data;
        start   = 1'b1;
        @(posedge clk);
        start = 1'b0;

        // 転送完了を待機
        wait (done);
        @(posedge clk);
        received = rx_data;
    endtask

    // メインテストシーケンス
    int pass_count = 0;
    int fail_count = 0;

    initial begin
        // 初期化
        rst_n   = 1'b0;
        start   = 1'b0;
        tx_data = '0;
        cpol    = 1'b0;
        cpha    = 1'b0;
        clk_div = CLK_DIV;

        repeat (10) @(posedge clk);
        rst_n = 1'b1;
        repeat (5) @(posedge clk);

        // 4つのSPIモードすべてをテスト
        for (int mode = 0; mode < 4; mode++) begin
            cpol = mode[1];
            cpha = mode[0];
            $display("=== SPI Mode %0d (CPOL=%0b, CPHA=%0b) ===",
                     mode, cpol, cpha);

            // ダイレクトテスト
            begin
                logic [DATA_WIDTH-1:0] recv;
                spi_transfer(8'hA5, recv);
                if (slave_rx_data === 8'hA5) begin
                    pass_count++;
                    $display("[PASS] 送信=0xA5, スレーブ受信=0x%02h", slave_rx_data);
                end else begin
                    fail_count++;
                    $error("[FAIL] 送信=0xA5, スレーブ受信=0x%02h", slave_rx_data);
                end

                repeat (20) @(posedge clk);
            end

            // ランダムテスト（各モード10回）
            for (int i = 0; i < 10; i++) begin
                logic [DATA_WIDTH-1:0] random_data, recv;
                assert(std::randomize(random_data));
                spi_transfer(random_data, recv);

                if (slave_rx_data === random_data) begin
                    pass_count++;
                end else begin
                    fail_count++;
                    $error("[FAIL] Mode%0d: 送信=0x%02h, スレーブ受信=0x%02h",
                           mode, random_data, slave_rx_data);
                end
                repeat (20) @(posedge clk);
            end
        end

        // 結果サマリ
        $display("============================");
        $display("PASS: %0d, FAIL: %0d", pass_count, fail_count);
        $display("============================");

        if (fail_count == 0)
            $display("ALL SPI TESTS PASSED");
        else
            $fatal(1, "SOME SPI TESTS FAILED");

        $finish;
    end

endmodule
```

### 18.5.5 期待される結果

```
=== SPI Mode 0 (CPOL=0, CPHA=0) ===
[PASS] 送信=0xA5, スレーブ受信=0xa5
=== SPI Mode 1 (CPOL=0, CPHA=1) ===
[PASS] 送信=0xA5, スレーブ受信=0xa5
=== SPI Mode 2 (CPOL=1, CPHA=0) ===
[PASS] 送信=0xA5, スレーブ受信=0xa5
=== SPI Mode 3 (CPOL=1, CPHA=1) ===
[PASS] 送信=0xA5, スレーブ受信=0xa5
============================
PASS: 44, FAIL: 0
============================
ALL SPI TESTS PASSED
```

---

## 18.6 デバッグ戦略

### 18.6.1 体系的なデバッグアプローチ

プロジェクト演習でバグに遭遇した場合、以下の体系的なアプローチで原因を特定します。これは第15章で学んだデバッグ技法の実践的な適用です。

1. **症状の特定**: テストの失敗メッセージやアサーション違反を正確に把握します
2. **再現性の確認**: 同じ条件で必ず再現するか、ランダムシードに依存するかを確認します
3. **問題の切り分け**: 関連するモジュールを特定し、影響範囲を絞り込みます
4. **波形の確認**: 信号のタイミングと値を波形ビューワで確認します
5. **根本原因の特定**: RTLのロジックやステートマシンの誤りを発見します
6. **修正と回帰テスト**: 修正後、既存のテストがすべて通ることを確認します

![デバッグフロー](/images/systemverilog-complete-guide/ch17_debug_flow.drawio.png)

### 18.6.2 各プロジェクトでの典型的な問題

プロジェクトごとに起こりやすい典型的な問題と、その対処法を以下にまとめます。

**UARTコントローラ**

| 症状 | 想定される原因 | デバッグ方法 |
|------|-------------|------------|
| 受信データが化ける | ボーレート不一致 | 分周カウンタの計算値を確認 |
| スタートビット検出失敗 | サンプリングタイミング異常 | tick信号の周期を波形で確認 |
| データがシフトしている | ビットカウンタのオフバイワン | bit_cntの遷移を波形で追跡 |

**簡易CPU**

| 症状 | 想定される原因 | デバッグ方法 |
|------|-------------|------------|
| 演算結果が不正 | ALUオペコードの割り当てミス | 制御ユニットの出力を確認 |
| 分岐が動作しない | ゼロフラグの接続ミス | ALU→制御の信号を追跡 |
| メモリアクセス異常 | アドレス計算の誤り | ALU結果とメモリアドレスを比較 |

**SPIマスター**

| 症状 | 想定される原因 | デバッグ方法 |
|------|-------------|------------|
| データ化け | CPOL/CPHAの処理ミス | SCLKとMOSI/MISOのタイミング確認 |
| 最初のビットが欠落 | CS→SCLKのセットアップ時間不足 | CSアサートからSCLK開始までの時間確認 |
| 転送が完了しない | ビットカウンタの終了条件ミス | bit_cntの値を波形で監視 |

### 18.6.3 アサーションを活用したデバッグ

各プロジェクトに組み込むべきアサーションの例を示します。アサーションを事前に埋め込んでおくことで、バグの早期発見が可能になります。

```systemverilog
// UARTアサーション例
module uart_assertions (
    input logic clk, rst_n,
    input logic tx, rx,
    input logic tx_busy, rx_valid
);

    // TX信号はリセット解除後、アイドル時にHighであること
    property tx_idle_high;
        @(posedge clk) disable iff (!rst_n)
        !tx_busy |-> tx;
    endproperty
    assert property (tx_idle_high)
        else $error("TX line is not HIGH during idle");

    // rx_validは1クロック幅であること
    property rx_valid_pulse;
        @(posedge clk) disable iff (!rst_n)
        rx_valid |=> !rx_valid;
    endproperty
    assert property (rx_valid_pulse)
        else $error("rx_valid is not a single-cycle pulse");

endmodule
```

```systemverilog
// CPUアサーション例
module cpu_assertions (
    input logic       clk, rst_n,
    input logic [3:0] opcode,
    input logic       reg_write,
    input logic       mem_write,
    input logic       halt
);

    // HALT時にはメモリ書き込みが発生しないこと
    property no_write_on_halt;
        @(posedge clk) disable iff (!rst_n)
        halt |-> !mem_write && !reg_write;
    endproperty
    assert property (no_write_on_halt)
        else $error("Write occurred during HALT");

    // レジスタ書き込みとメモリ書き込みが同時に発生しないこと
    property no_simultaneous_write;
        @(posedge clk) disable iff (!rst_n)
        !(reg_write && mem_write);
    endproperty
    assert property (no_simultaneous_write)
        else $error("Simultaneous register and memory write");

endmodule
```

```systemverilog
// SPIアサーション例
module spi_assertions (
    input logic clk, rst_n,
    input logic cs_n, sclk, busy, done
);

    // CS_Nがデアサート中はSCLKがトグルしないこと
    property no_sclk_without_cs;
        @(posedge clk) disable iff (!rst_n)
        cs_n |-> $stable(sclk);
    endproperty
    assert property (no_sclk_without_cs)
        else $error("SCLK toggled while CS_N is deasserted");

    // doneは1クロック幅のパルスであること
    property done_pulse_width;
        @(posedge clk) disable iff (!rst_n)
        done |=> !done;
    endproperty
    assert property (done_pulse_width)
        else $error("done signal is not a single-cycle pulse");

endmodule
```

---

## 18.7 プロジェクト演習ガイドライン

### 18.7.1 プロジェクトの取り組み方

各プロジェクトに取り組む際は、以下のステップに従うことを推奨します。

1. **仕様書を読む**: まず仕様を完全に理解してから設計を開始してください
2. **ブロック図を描く**: モジュール間のインターフェースを明確にします
3. **段階的に実装する**: 最小限の機能から始めて、徐々に機能を追加します
4. **こまめにテストする**: モジュール単位でテストし、統合前にバグを発見します
5. **波形を確認する**: 期待通りの動作かどうか、必ず波形で確認します

### 18.7.2 検証計画の作成

各プロジェクトで検証計画を作成することが重要です。以下のテンプレートを参考にしてください。

| 検証項目 | テスト種類 | カバレッジ目標 | 優先度 |
|---------|-----------|-------------|--------|
| 基本機能 | ダイレクトテスト | 全パスの実行確認 | 高 |
| 境界値 | ダイレクトテスト | 最小/最大/ゼロ値 | 高 |
| ランダムシナリオ | 制約付きランダム | 機能カバレッジ100% | 中 |
| エラーケース | ダイレクトテスト | エラーハンドリング確認 | 中 |
| コーナーケース | ダイレクト/ランダム | 特殊な組み合わせ | 低 |

### 18.7.3 カバレッジ駆動検証の実践

プロジェクト演習では、カバレッジ駆動検証（CDV: Coverage Driven Verification）のアプローチを採用します。

```systemverilog
// カバレッジ駆動検証のフレームワーク例
class coverage_driven_test;

    // カバレッジ目標
    real target_coverage = 95.0;

    // テストループ
    task run();
        int iteration = 0;
        real current_coverage;

        while (1) begin
            iteration++;

            // ランダムシナリオを生成・実行
            generate_and_run_scenario();

            // カバレッジを確認
            current_coverage = $get_coverage();
            $display("反復%0d: カバレッジ = %.1f%%", iteration, current_coverage);

            // 目標達成で終了
            if (current_coverage >= target_coverage) begin
                $display("カバレッジ目標 %.1f%% を達成しました（%0d回の反復）",
                         target_coverage, iteration);
                break;
            end

            // 上限に達した場合は警告
            if (iteration >= 10000) begin
                $warning("反復上限に達しました。カバレッジ = %.1f%%",
                         current_coverage);
                break;
            end
        end
    endtask

    // シナリオ生成・実行（プロジェクトごとにオーバーライド）
    virtual task generate_and_run_scenario();
    endtask

endclass
```

### 18.7.4 プロジェクト評価基準

各プロジェクトの達成度は、以下の基準で評価します。

| 評価項目 | 配点 | 内容 |
|---------|------|------|
| **RTL設計の正確性** | 30% | 仕様通りに動作するか |
| **コードの可読性** | 15% | 命名規則、コメント、構造化 |
| **テストベンチの品質** | 25% | テストの網羅性、自動チェック機能 |
| **カバレッジ達成率** | 20% | 機能カバレッジ・コードカバレッジの達成度 |
| **アサーションの活用** | 10% | 適切なアサーションの配置 |

### 18.7.5 発展課題

各プロジェクトの基本演習を完了した後、以下の発展課題にも挑戦してみてください。

**UART発展課題**

- **パリティ機能の追加**: 偶数パリティ/奇数パリティの送受信を実装します
- **FIFO統合**: 8段のFIFOを送信/受信パスに組み込みます
- **フロー制御**: RTS/CTSによるハードウェアフロー制御を追加します

**簡易CPU発展課題**

- **パイプライン化**: 2段または3段のパイプラインを実装します
- **命令追加**: シフト演算、ジャンプ命令を追加します
- **割り込み処理**: 単純な割り込みメカニズムを追加します

**SPI発展課題**

- **マルチスレーブ対応**: 複数のCS信号を管理するデコーダを追加します
- **DMA連携**: DMAコントローラとの連携インターフェースを設計します
- **可変データ幅**: 8/16/32ビットの転送幅を動的に切り替え可能にします

---

## 18.8 まとめ

本章では、3つのプロジェクト演習を通じて、SystemVerilogによる設計・検証の実践スキルを学びました。

1. **UARTコントローラ**では、ボーレートジェネレータ・送信モジュール・受信モジュールの設計と、ループバック方式による検証手法を学びました。16倍オーバーサンプリングによるビット中央サンプリングが信頼性の鍵となります。
2. **簡易CPU**では、命令セットアーキテクチャの定義からALU・レジスタファイル・制御ユニットの実装まで、プロセッサ設計の基本フローを体験しました。命令デコードと制御信号の正確な対応付けが重要です。
3. **SPIマスターコントローラ**では、4つのSPIモード（CPOL/CPHAの組み合わせ）に対応した設計と、スレーブモデルを用いた全二重通信の検証を行いました。
4. **デバッグ戦略**として、体系的なアプローチ（症状特定→再現性確認→問題切り分け→波形確認→根本原因特定→回帰テスト）を各プロジェクトに適用する手法を学びました。
5. **アサーションの埋め込み**により、プロトコル違反やタイミング異常を早期に検出できることを確認しました。
6. **カバレッジ駆動検証（CDV）**のアプローチにより、検証の完了基準を定量的に管理する方法を実践しました。

次章では、SystemVerilogのさらなる学習に向けたリソースと、本書全体のまとめを行います。
