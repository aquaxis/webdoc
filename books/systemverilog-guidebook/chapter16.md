---
title: "第16章：デバッグ技法"
---

# 第16章：デバッグ技法

## この章で学ぶこと

デバッグはVLSI設計フローにおいて最も多くの時間を費やす工程の一つです。本章では、$display系タスクによるテキストベースのデバッグから、波形解析、アサーションベースのデバッグ、UVMのレポーティング機能まで、体系的なデバッグ手法を解説します。効率的なデバッグ環境の構築方法についても学びます。

---

## 16.1 デバッグの基本技法

### 16.1.1 $display系タスクの効果的な使い方

SystemVerilogの `$display` 系タスクは、最も基本的でありながら最も頻繁に使用されるデバッグ手段です。各タスクの特性を理解し、適切に使い分けることが重要です。

```systemverilog
module display_examples;
    logic [7:0]  data;
    logic [31:0] addr;
    logic        clk;

    // $display: 実行時即座に表示（改行あり）
    initial begin
        data = 8'hFF;
        addr = 32'h1000;
        $display("Time=%0t: data=0x%h, addr=0x%h", $time, data, addr);
        // 出力: Time=0: data=0xff, addr=00001000
    end

    // $write: 改行なしで表示
    initial begin
        for (int i = 0; i < 4; i++) begin
            $write("%0d ", i);
        end
        $write("\n");  // 手動で改行
        // 出力: 0 1 2 3
    end

    // $displayh, $displayb, $displayo: デフォルト基数指定
    initial begin
        #10;
        data = 8'b1010_0101;
        $displayh("Hex: ", data);     // Hex: a5
        $displayb("Bin: ", data);     // Bin: 10100101
        $displayo("Oct: ", data);     // Oct: 245
    end
endmodule
```

フォーマット指定子の一覧です。

| 指定子 | 説明 | 例 |
|--------|------|-----|
| `%d` / `%0d` | 10進数（`%0d`はゼロ埋めなし） | `255` |
| `%h` / `%H` | 16進数 | `ff` |
| `%b` / `%B` | 2進数 | `11111111` |
| `%o` / `%O` | 8進数 | `377` |
| `%s` | 文字列 | `"hello"` |
| `%t` | 時間（`$timeformat`に従う） | `100.000ns` |
| `%m` | 階層パス名 | `top.dut.u_ctrl` |
| `%e` / `%f` / `%g` | 浮動小数点 | `3.14` |

`$timeformat` でタイムスタンプの表示形式をカスタマイズできます。

```systemverilog
module time_format_example;
    initial begin
        // $timeformat(単位, 精度, サフィックス, 最小幅)
        $timeformat(-9, 3, " ns", 12);  // ナノ秒、小数3桁

        #100.5;
        $display("Time = %t", $time);
        // 出力: Time =  100.500 ns
    end
endmodule
```

### 16.1.2 $monitor, $strobe の活用

`$monitor` と `$strobe` は、`$display` とは異なるタイミングで実行される特殊な表示タスクです。

```systemverilog
module monitor_strobe_demo;
    logic       clk;
    logic [7:0] count;

    // クロック生成
    initial clk = 0;
    always #5 clk = ~clk;

    // カウンタ
    always @(posedge clk) begin
        count <= count + 1;
    end

    initial begin
        count = 0;

        // $monitor: 引数のいずれかが変化するたびに自動表示
        // プロセス全体で1つだけアクティブ
        $monitor("MON  [%0t] count=%0d", $time, count);

        // $strobe: 現在のタイムステップの最後に表示
        // NBAの結果が反映された後の値を見られる
        forever @(posedge clk) begin
            $strobe("STRB [%0t] count=%0d", $time, count);
            $display("DISP [%0t] count=%0d", $time, count);
        end
    end

    initial #100 $finish;
endmodule
```

`$display` と `$strobe` の違いは重要です。

```systemverilog
module display_vs_strobe;
    logic [7:0] data;

    initial begin
        data = 8'h00;

        // $displayはアクティブ領域で実行される
        // $strobeはリアクティブ領域（タイムステップの最後）で実行される
        fork
            begin
                data <= 8'hFF;  // NBA
                $display("$display: data=%h", data);  // 00 (古い値)
                $strobe("$strobe:  data=%h", data);    // ff (新しい値)
            end
        join
    end
endmodule
```

| タスク | 実行タイミング | 複数登録 | 改行 |
|--------|-------------|---------|------|
| `$display` | 呼び出し時（アクティブ領域） | 制限なし | あり |
| `$write` | 呼び出し時（アクティブ領域） | 制限なし | なし |
| `$strobe` | タイムステップの最後（リアクティブ領域） | 制限なし | あり |
| `$monitor` | 引数が変化した時 | 1つのみ | あり |

### 16.1.3 ファイルへのログ出力

大規模なシミュレーションでは、コンソール出力だけでなくファイルにログを記録することが不可欠です。

```systemverilog
module file_logging;
    logic       clk;
    logic [7:0] data;
    logic       valid;

    // ファイルハンドル
    integer log_fd;
    integer csv_fd;

    initial begin
        // ファイルオープン
        log_fd = $fopen("simulation.log", "w");
        csv_fd = $fopen("trace.csv", "w");

        if (log_fd == 0) begin
            $display("ERROR: ログファイルを開けませんでした");
            $finish;
        end

        // CSVヘッダ出力
        $fdisplay(csv_fd, "Time,Data,Valid");

        // ログヘッダ
        $fdisplay(log_fd, "=== Simulation Log ===");
        $fdisplay(log_fd, "Start time: %0t", $time);
    end

    // データのログ記録
    always @(posedge clk) begin
        if (valid) begin
            $fdisplay(log_fd, "[%0t] Data received: 0x%h", $time, data);
            $fdisplay(csv_fd, "%0t,%h,%b", $time, data, valid);
        end
    end

    // シミュレーション終了時にファイルクローズ
    final begin
        $fdisplay(log_fd, "=== End of Log ===");
        $fdisplay(log_fd, "End time: %0t", $time);
        $fclose(log_fd);
        $fclose(csv_fd);
    end
endmodule
```

マルチチャネルディスクリプタ（MCD）を使えば、複数の出力先に同時にログを書き出すこともできます。

```systemverilog
module multi_channel_log;
    integer fd1, fd2, mcd;

    initial begin
        fd1 = $fopen("debug.log");
        fd2 = $fopen("error.log");

        // MCDを使って複数ファイルとSTDOUTに同時出力
        mcd = fd1 | fd2 | 1;  // 1はSTDOUT

        $fdisplay(mcd, "This message goes to both files and stdout");

        $fclose(fd1);
        $fclose(fd2);
    end
endmodule
```

### 16.1.4 条件付きデバッグメッセージ（verbosityレベル）

大規模プロジェクトでは、デバッグメッセージの出力レベルを制御する仕組みが重要です。

```systemverilog
// デバッグverbosityパッケージ
package debug_pkg;
    typedef enum int {
        VERB_NONE  = 0,
        VERB_ERROR = 1,
        VERB_WARN  = 2,
        VERB_INFO  = 3,
        VERB_DEBUG = 4,
        VERB_TRACE = 5
    } verbosity_t;

    // グローバルverbosityレベル
    verbosity_t global_verbosity = VERB_INFO;

    // デバッグ出力関数
    function void dbg_print(verbosity_t level, string msg);
        if (level <= global_verbosity) begin
            string prefix;
            case (level)
                VERB_ERROR: prefix = "ERROR";
                VERB_WARN:  prefix = "WARN ";
                VERB_INFO:  prefix = "INFO ";
                VERB_DEBUG: prefix = "DEBUG";
                VERB_TRACE: prefix = "TRACE";
                default:    prefix = "?????";
            endcase
            $display("[%0t] [%s] %s", $time, prefix, msg);
        end
    endfunction

    function void dbg_error(string msg);
        dbg_print(VERB_ERROR, msg);
    endfunction

    function void dbg_warn(string msg);
        dbg_print(VERB_WARN, msg);
    endfunction

    function void dbg_info(string msg);
        dbg_print(VERB_INFO, msg);
    endfunction

    function void dbg_debug(string msg);
        dbg_print(VERB_DEBUG, msg);
    endfunction

    function void dbg_trace(string msg);
        dbg_print(VERB_TRACE, msg);
    endfunction
endpackage

// 使用例
module controller;
    import debug_pkg::*;

    logic [3:0] state;

    always @(posedge clk) begin
        case (state)
            4'h0: begin
                dbg_trace("IDLE状態");
                if (start) begin
                    dbg_info("処理を開始します");
                    state <= 4'h1;
                end
            end
            4'h1: begin
                dbg_debug($sformatf("処理中: data=0x%h", data));
                if (error_flag) begin
                    dbg_error("エラーが検出されました");
                    state <= 4'hF;
                end else if (done) begin
                    dbg_info("処理が完了しました");
                    state <= 4'h0;
                end
            end
            default: begin
                dbg_warn($sformatf("未定義の状態: %0d", state));
            end
        endcase
    end
endmodule
```

### 16.1.5 `ifdef/`ifndef によるデバッグコードの制御

コンパイルディレクティブを使って、デバッグコードの有効/無効を切り替えることができます。

```systemverilog
module alu (
    input  logic        clk,
    input  logic        rst_n,
    input  logic [31:0] op_a,
    input  logic [31:0] op_b,
    input  logic [3:0]  alu_op,
    output logic [31:0] result,
    output logic        zero_flag
);

    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            result    <= '0;
            zero_flag <= '0;
        end else begin
            case (alu_op)
                4'b0000: result <= op_a + op_b;
                4'b0001: result <= op_a - op_b;
                4'b0010: result <= op_a & op_b;
                4'b0011: result <= op_a | op_b;
                4'b0100: result <= op_a ^ op_b;
                default: result <= '0;
            endcase
            zero_flag <= (result == '0);

            `ifdef DEBUG_ALU
                $display("[%0t] ALU: op_a=0x%h, op_b=0x%h, op=%b, result=0x%h",
                    $time, op_a, op_b, alu_op, result);
            `endif

            `ifdef ASSERT_ON
                assert (alu_op <= 4'b0100)
                    else $error("Invalid ALU operation: %b", alu_op);
            `endif
        end
    end

    // デバッグ用の追加ロジック（合成時は除外）
    `ifdef DEBUG_ALU
        // 演算回数カウンタ
        integer op_count = 0;
        always @(posedge clk) begin
            if (rst_n) op_count <= op_count + 1;
        end

        // 定期的な統計表示
        always @(posedge clk) begin
            if (op_count > 0 && op_count % 1000 == 0) begin
                $display("[%0t] ALU Statistics: %0d operations performed",
                    $time, op_count);
            end
        end
    `endif

endmodule
```

コンパイル時にフラグを指定してデバッグモードを制御します。

```systemverilog
// デバッグモードでコンパイル
// vcs -sverilog +define+DEBUG_ALU+ASSERT_ON design.sv testbench.sv

// リリースモード（デバッグコードなし）
// vcs -sverilog design.sv testbench.sv
```

複数レベルのデバッグを条件付きコンパイルで制御するパターンも有用です。

```systemverilog
// デバッグレベルのマクロ定義
`ifdef DEBUG_LEVEL_3
    `define DEBUG_LEVEL_2
    `define DEBUG_LEVEL_1
`elsif DEBUG_LEVEL_2
    `define DEBUG_LEVEL_1
`endif

module my_module;
    always @(posedge clk) begin
        `ifdef DEBUG_LEVEL_1
            // 基本的なトランザクション情報
            if (valid) $display("[%0t] Transaction: addr=0x%h", $time, addr);
        `endif

        `ifdef DEBUG_LEVEL_2
            // 詳細なデータダンプ
            if (valid) begin
                $display("[%0t] Data payload:", $time);
                for (int i = 0; i < 16; i++)
                    $display("  [%0d] = 0x%h", i, data_array[i]);
            end
        `endif

        `ifdef DEBUG_LEVEL_3
            // サイクルごとの全状態表示
            $display("[%0t] state=%s, count=%0d, fifo_level=%0d",
                $time, state.name(), count, fifo_level);
        `endif
    end
endmodule
```

---

## 16.2 波形デバッグ

### 16.2.1 VCDファイルの出力

**VCD（Value Change Dump）**は、IEEE 1364で標準化された波形データフォーマットです。ほぼすべてのシミュレータとビューワがサポートしています。

```systemverilog
module testbench;
    logic       clk, rst_n;
    logic [7:0] data;
    logic       valid, ready;

    // DUT のインスタンス化
    my_design dut (.*);

    // クロック生成
    initial clk = 0;
    always #5 clk = ~clk;

    // VCDダンプ設定
    initial begin
        // ダンプファイル名の指定
        $dumpfile("wave.vcd");

        // ダンプ対象の指定
        // $dumpvars(レベル, スコープ)
        // レベル0: 指定スコープ以下のすべての階層
        $dumpvars(0, testbench);

        // 特定の信号のみダンプすることも可能
        // $dumpvars(0, dut.u_ctrl.state);
        // $dumpvars(0, dut.u_datapath);
    end

    // ダンプの制御
    initial begin
        // 最初の100nsはダンプしない
        $dumpoff;
        #100;
        $dumpon;

        // 1000ns後にダンプを停止してファイルをフラッシュ
        #1000;
        $dumpoff;
        $dumpflush;

        // 再度ダンプを開始
        #500;
        $dumpon;
    end

    // テストシーケンス
    initial begin
        rst_n = 0;
        data  = 0;
        valid = 0;
        #20 rst_n = 1;

        repeat (100) begin
            @(posedge clk);
            data  <= $urandom;
            valid <= $urandom_range(0, 1);
        end

        #100 $finish;
    end
endmodule
```

`$dumpvars` のレベル指定による制御は以下の通りです。

| レベル | 説明 |
|--------|------|
| 0 | 指定スコープ以下のすべての階層の信号をダンプ |
| 1 | 指定スコープ直下の信号のみダンプ |
| 2 | 指定スコープから2階層分の信号をダンプ |
| N | 指定スコープからN階層分の信号をダンプ |

### 16.2.2 FSDBファイルの出力

**FSDB（Fast Signal Database）**は、Synopsys社が定義した高性能な波形フォーマットです。VCDに比べて圧縮率が高く、大規模設計での使用に適しています。

```systemverilog
module testbench;
    // ... 設計のインスタンス化 ...

    initial begin
        // FSDBダンプの設定（Synopsys VCSの場合）
        // +fsdbfile+wave.fsdb をコマンドラインオプションで指定する方法もある
        $fsdbDumpfile("wave.fsdb");
        $fsdbDumpvars(0, testbench);

        // 特定のスコープのみ
        // $fsdbDumpvars(0, testbench.dut);
        // $fsdbDumpvars("+all");  // SVA、パワーなども含む
    end

    // ダンプの制御
    initial begin
        // ダンプの一時停止と再開
        #1000;
        $fsdbDumpoff;
        #500;
        $fsdbDumpon;
    end

    // FSDB固有の機能
    initial begin
        // トランザクションのダンプ
        // $fsdbDumpSVA;             // SVAのダンプ
        // $fsdbDumpMDA;             // メモリ配列のダンプ
    end
endmodule
```

VCDとFSDBの比較は以下の通りです。

| 比較項目 | VCD | FSDB |
|---------|-----|------|
| **規格** | IEEE 1364標準 | Synopsys独自 |
| **ファイルサイズ** | 大きい（テキスト形式） | 小さい（バイナリ圧縮） |
| **書き込み速度** | 遅い | 高速 |
| **読み込み速度** | 遅い | 高速 |
| **対応ビューワ** | GTKWave, Verdi, DVEなど | Verdi, DVE |
| **SVAサポート** | なし | あり |
| **トランザクション** | なし | あり |
| **大規模設計** | 不向き | 最適 |

### 16.2.3 波形ビューワの活用

波形ビューワは、信号の時間変化をグラフィカルに表示し、設計の動作を直感的に理解するための不可欠なツールです。

![波形デバッグフロー](/images/systemverilog-complete-guide/ch15_waveform_debug.drawio.png)

主要な波形ビューワの特徴を以下にまとめます。

| ビューワ | 提供元 | 対応フォーマット | 特徴 |
|---------|--------|----------------|------|
| **Verdi** | Synopsys | FSDB, VCD | 高機能、ソースコード連携、SVAデバッグ |
| **DVE** | Synopsys | VPD, VCD | VCSに付属、基本的な波形表示 |
| **SimVision** | Cadence | SHM, VCD | Xceliumに付属、トランザクションデバッグ |
| **Visualizer** | Siemens EDA | WLF, VCD | Questaに付属、高速読み込み |
| **GTKWave** | オープンソース | VCD, FST, LXT | 無償、軽量、CI/CD向き |

波形ビューワを効率的に使うためのポイントをまとめます。

1. **信号グループの活用**: 関連する信号をグループ化して保存しておきます
2. **マーカーの設定**: 重要な時間点にマーカーを設定し、素早くジャンプできるようにします
3. **バスの基数設定**: アドレスやデータバスは16進数表示が見やすいです
4. **アナログ表示**: カウンタやFIFOレベルなどはアナログ波形で傾向を把握します
5. **カーソル間の時間計測**: 2つのカーソルを使ってイベント間の時間を測定します

### 16.2.4 波形解析のテクニック

波形デバッグを効率的に行うためのテクニックを紹介します。

**テクニック1: ブックマーク信号の追加**

特定のイベントを素早く特定するため、テストベンチにブックマーク用の信号を追加します。

```systemverilog
module testbench;
    // デバッグ用のブックマーク信号
    string test_phase;
    integer error_count;
    logic   error_flag;

    // テスト中にブックマークを設定
    task run_test();
        test_phase = "RESET";
        reset_dut();

        test_phase = "CONFIG";
        configure_dut();

        test_phase = "TRAFFIC";
        send_traffic();

        test_phase = "CHECK";
        check_results();

        test_phase = "DONE";
    endtask

    // エラー発生箇所のマーキング
    task check_data(logic [31:0] actual, logic [31:0] expected);
        if (actual !== expected) begin
            error_flag  = 1;
            error_count = error_count + 1;
            $display("ERROR at %0t: actual=0x%h, expected=0x%h",
                $time, actual, expected);
            #1 error_flag = 0;  // パルスとして可視化
        end
    endtask
endmodule
```

**テクニック2: プロトコル信号のまとめ表示**

```systemverilog
// AXIバスの信号をグループ化するためのインターフェース
interface axi_debug_if (
    input logic aclk,
    input logic aresetn
);
    // 書き込みチャネルの状態をまとめた信号
    logic [2:0] wr_state;  // 0:IDLE, 1:ADDR, 2:DATA, 3:RESP

    // 読み出しチャネルの状態をまとめた信号
    logic [2:0] rd_state;  // 0:IDLE, 1:ADDR, 2:DATA

    // トランザクションカウンタ
    integer wr_txn_count;
    integer rd_txn_count;

    // 未完了トランザクション数
    integer outstanding_wr;
    integer outstanding_rd;
endinterface
```

### 16.2.5 ダンプ範囲の制御とパフォーマンス

波形ダンプはシミュレーション速度に大きな影響を与えます。適切なダンプ範囲の制御が重要です。

```systemverilog
module dump_control;
    // 方法1: 時間範囲の制御
    initial begin
        $dumpfile("wave.vcd");
        $dumpvars(0, testbench);

        // 問題が発生する前後のみダンプ
        $dumpoff;

        // 問題発生予想時刻の手前でダンプ開始
        #9000;
        $dumpon;
        #2000;  // 2000ns分だけダンプ
        $dumpoff;
    end

    // 方法2: 条件付きダンプ制御
    always @(posedge clk) begin
        if (error_detected) begin
            $dumpon;
            // エラー後100サイクル分ダンプして停止
            repeat (100) @(posedge clk);
            $dumpoff;
            $dumpflush;
        end
    end

    // 方法3: 階層の限定
    initial begin
        $dumpfile("targeted.vcd");
        // トップレベルの信号のみ（1階層）
        $dumpvars(1, testbench);
        // 問題のあるモジュール以下を詳細にダンプ
        $dumpvars(0, testbench.dut.u_problematic_module);
    end
endmodule
```

ダンプ最適化のガイドラインをまとめます。

| 手法 | 速度向上 | トレードオフ |
|------|---------|------------|
| ダンプ階層の制限 | 高 | 他階層の信号が見えない |
| 時間範囲の制限 | 高 | 問題の時刻を事前に知る必要あり |
| FSDBの使用 | 中 | ツール依存 |
| ダンプ無効化 | 最高 | 波形が全く取れない |
| ダンプレベルの制限 | 中 | 深い階層の信号が見えない |

---

## 16.3 アサーションベースデバッグ

### 16.3.1 アサーション失敗の解析

アサーションが失敗した場合の効果的なデバッグ手順を解説します。

```systemverilog
module axi_slave_assertions (
    input logic        aclk,
    input logic        aresetn,
    input logic        awvalid,
    input logic        awready,
    input logic [31:0] awaddr,
    input logic [7:0]  awlen,
    input logic        wvalid,
    input logic        wready,
    input logic        wlast,
    input logic        bvalid,
    input logic        bready
);

    // アサーション1: リセット中はvalidがアサートされない
    property p_no_valid_during_reset;
        @(posedge aclk)
        !aresetn |-> !awvalid && !wvalid;
    endproperty

    a_no_valid_during_reset: assert property (p_no_valid_during_reset)
        else $error("[%0t] AXI Protocol Violation: Valid asserted during reset",
            $time);

    // アサーション2: awvalidがアサートされたらawreadyまで保持
    property p_awvalid_stable;
        @(posedge aclk) disable iff (!aresetn)
        awvalid && !awready |=> awvalid;
    endproperty

    a_awvalid_stable: assert property (p_awvalid_stable)
        else $error("[%0t] AXI Violation: awvalid deasserted before awready",
            $time);

    // アサーション3: wlastの正確性
    sequence s_write_burst;
        awvalid && awready ##1
        (wvalid && wready)[*1:$] ##0 wlast;
    endsequence

    // カバレッジポイント：アサーションが実際にアクティベートされたことを確認
    c_write_burst: cover property (
        @(posedge aclk) disable iff (!aresetn)
        s_write_burst
    );

    // デバッグ情報の付加
    // アサーション失敗時に関連信号を表示
    a_awvalid_stable_debug: assert property (p_awvalid_stable)
    else begin
        $error("[%0t] ASSERTION FAILED: a_awvalid_stable", $time);
        $display("  awvalid=%b, awready=%b", awvalid, awready);
        $display("  awaddr=0x%h, awlen=%0d", awaddr, awlen);
        $display("  Context: state=%s", dut.state.name());
    end

endmodule
```

アサーション失敗のデバッグ手順は以下の通りです。

1. **失敗メッセージの確認**: 時刻、アサーション名、失敗原因を特定します
2. **波形での確認**: 失敗時刻付近の関連信号を波形ビューワで確認します
3. **アサーションの分解**: 複雑なアサーションは部品に分解して、どの部分が不成立かを特定します
4. **前提条件の確認**: `disable iff` や前提条件（antecedent）が正しいかを検証します
5. **修正と再テスト**: RTLまたはアサーションを修正し、再度テストを実行します

### 16.3.2 SVAデバッグのベストプラクティス

SVA（SystemVerilog Assertions）を効果的にデバッグするためのベストプラクティスを紹介します。

```systemverilog
// ベストプラクティス1: 名前付きアサーションとラベル
// 悪い例
assert property (@(posedge clk) req |-> ##[1:3] ack);

// 良い例：名前とメッセージ付き
a_req_ack_handshake: assert property (
    @(posedge clk) disable iff (!rst_n)
    req |-> ##[1:3] ack
) else $error("Request not acknowledged within 3 cycles at %0t", $time);

// ベストプラクティス2: 複雑なプロパティの段階的な検証
// ステップ1: 基本シーケンスの確認
sequence s_basic_req;
    req && !busy;
endsequence

// ステップ2: レスポンスシーケンスの確認
sequence s_basic_ack;
    ##[1:3] ack;
endsequence

// ステップ3: 組み合わせ
property p_req_ack;
    @(posedge clk) disable iff (!rst_n)
    s_basic_req |-> s_basic_ack;
endproperty

// ベストプラクティス3: カバレッジとの組み合わせ
// アサーションがアクティブになっているか確認
c_req_ack_fired: cover property (
    @(posedge clk) disable iff (!rst_n)
    s_basic_req ##0 s_basic_ack
);

// ベストプラクティス4: デバッグ用ヘルパー信号
logic [31:0] dbg_cycle_count;
always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n) dbg_cycle_count <= 0;
    else        dbg_cycle_count <= dbg_cycle_count + 1;
end

// アサーション失敗時のサイクル番号も表示
a_fifo_overflow: assert property (
    @(posedge clk) disable iff (!rst_n)
    fifo_wr && !fifo_full
) else $error("FIFO overflow at cycle %0d (time %0t)",
    dbg_cycle_count, $time);
```

### 16.3.3 bind を使ったアサーションの挿入

`bind` 構文を使うと、RTLコードを一切変更せずにアサーションモジュールを挿入できます。これはデバッグにおいて非常に有用です。

```systemverilog
// アサーションモジュール（別ファイル）
module fifo_assertions #(
    parameter DEPTH = 16,
    parameter WIDTH = 8
) (
    input logic             clk,
    input logic             rst_n,
    input logic             wr_en,
    input logic             rd_en,
    input logic [WIDTH-1:0] wr_data,
    input logic [WIDTH-1:0] rd_data,
    input logic             full,
    input logic             empty,
    input logic [$clog2(DEPTH):0] count
);

    // オーバーフロー検出
    a_no_overflow: assert property (
        @(posedge clk) disable iff (!rst_n)
        wr_en |-> !full
    ) else $error("[%0t] FIFO Overflow: write while full", $time);

    // アンダーフロー検出
    a_no_underflow: assert property (
        @(posedge clk) disable iff (!rst_n)
        rd_en |-> !empty
    ) else $error("[%0t] FIFO Underflow: read while empty", $time);

    // カウント整合性
    a_count_range: assert property (
        @(posedge clk) disable iff (!rst_n)
        count >= 0 && count <= DEPTH
    ) else $error("[%0t] FIFO count out of range: %0d", $time, count);

    // fullフラグの整合性
    a_full_consistent: assert property (
        @(posedge clk) disable iff (!rst_n)
        full == (count == DEPTH)
    ) else $error("[%0t] Full flag inconsistent with count", $time);

    // emptyフラグの整合性
    a_empty_consistent: assert property (
        @(posedge clk) disable iff (!rst_n)
        empty == (count == 0)
    ) else $error("[%0t] Empty flag inconsistent with count", $time);

endmodule

// bindによるアサーション挿入（テストベンチのトップレベルまたは別ファイル）
bind sync_fifo fifo_assertions #(
    .DEPTH(FIFO_DEPTH),
    .WIDTH(DATA_WIDTH)
) u_fifo_assert (
    .clk     (clk),
    .rst_n   (rst_n),
    .wr_en   (wr_en),
    .rd_en   (rd_en),
    .wr_data (wr_data),
    .rd_data (rd_data),
    .full    (full),
    .empty   (empty),
    .count   (count)
);
```

`bind` を使う利点は以下の通りです。

- RTLソースコードに手を加える必要がありません
- アサーションファイルをバージョン管理で別管理できます
- コンパイル時に `bind` ファイルを含めるかどうかで有効/無効を切り替えられます
- 複数のインスタンスに一括で適用できます

### 16.3.4 フォーマル検証との連携

アサーションはシミュレーションだけでなく、フォーマル検証ツールでも使用できます。

```systemverilog
// フォーマル検証を意識したアサーション記述
module formal_friendly_assertions (
    input logic       clk,
    input logic       rst_n,
    input logic [3:0] state,
    input logic       req,
    input logic       ack,
    input logic       grant
);

    // assume: 環境の制約（フォーマルでは入力制約として機能）
    a_valid_state: assume property (
        @(posedge clk) disable iff (!rst_n)
        state inside {4'h0, 4'h1, 4'h2, 4'h3, 4'h4}
    );

    // assert: 検証対象のプロパティ
    a_mutual_exclusion: assert property (
        @(posedge clk) disable iff (!rst_n)
        grant |-> ack
    );

    // フォーマル検証用のリセットシーケンス
    a_reset_init: assert property (
        @(posedge clk)
        !rst_n |-> state == 4'h0
    );

    // cover: 到達可能性の確認
    c_full_sequence: cover property (
        @(posedge clk) disable iff (!rst_n)
        req ##[1:5] grant ##[1:3] ack ##1 !req
    );

    // フォーマル検証でのライブネスプロパティ
    // （いずれ必ず到達することの証明）
    a_liveness_grant: assert property (
        @(posedge clk) disable iff (!rst_n)
        req |-> s_eventually grant
    );

endmodule
```

---

## 16.4 高度なデバッグ技法

### 16.4.1 トランザクションレベルのデバッグ

信号レベルのデバッグだけでなく、トランザクションレベルでの理解がバグの特定を大幅に効率化します。

```systemverilog
// トランザクション記録クラス
class transaction_logger #(type T = uvm_sequence_item);
    // ログファイル
    integer log_fd;
    string  log_name;
    int     txn_count;

    function new(string name = "txn_log");
        log_name = name;
        txn_count = 0;
        log_fd = $fopen({name, ".log"}, "w");
        $fdisplay(log_fd, "=== Transaction Log: %s ===", name);
    endfunction

    function void log_transaction(T txn, string phase = "");
        txn_count++;
        $fdisplay(log_fd, "[%0t] TXN #%0d %s:", $time, txn_count, phase);
        $fdisplay(log_fd, "  %s", txn.sprint());
        $fdisplay(log_fd, "---");
    endfunction

    function void close();
        $fdisplay(log_fd, "=== Total transactions: %0d ===", txn_count);
        $fclose(log_fd);
    endfunction
endclass

// トランザクション比較とミスマッチ検出
class transaction_checker;
    // 期待値キュー
    mailbox #(axi_txn) expected_q;
    integer mismatch_count;
    integer match_count;
    integer log_fd;

    function new();
        expected_q = new();
        mismatch_count = 0;
        match_count = 0;
        log_fd = $fopen("checker.log", "w");
    endfunction

    task check(axi_txn actual);
        axi_txn expected;
        if (expected_q.try_get(expected)) begin
            if (!actual.compare(expected)) begin
                mismatch_count++;
                $fdisplay(log_fd, "[%0t] MISMATCH #%0d:", $time, mismatch_count);
                $fdisplay(log_fd, "  Expected: %s", expected.sprint());
                $fdisplay(log_fd, "  Actual  : %s", actual.sprint());
                // 差分の詳細表示
                show_diff(expected, actual);
            end else begin
                match_count++;
            end
        end else begin
            $fdisplay(log_fd, "[%0t] ERROR: Unexpected transaction:", $time);
            $fdisplay(log_fd, "  %s", actual.sprint());
        end
    endtask

    function void show_diff(axi_txn exp, axi_txn act);
        if (exp.addr !== act.addr)
            $fdisplay(log_fd, "  DIFF addr: 0x%h vs 0x%h", exp.addr, act.addr);
        if (exp.data !== act.data)
            $fdisplay(log_fd, "  DIFF data: 0x%h vs 0x%h", exp.data, act.data);
        if (exp.resp !== act.resp)
            $fdisplay(log_fd, "  DIFF resp: %s vs %s", exp.resp.name(), act.resp.name());
    endfunction
endclass
```

### 16.4.2 UVMデバッグ機能

UVM（Universal Verification Methodology）には、強力なデバッグ・レポーティング機能が組み込まれています。

```systemverilog
class my_driver extends uvm_driver #(my_txn);
    `uvm_component_utils(my_driver)

    function new(string name, uvm_component parent);
        super.new(name, parent);
    endfunction

    task run_phase(uvm_phase phase);
        my_txn txn;

        forever begin
            seq_item_port.get_next_item(txn);

            // UVMレポーティングマクロの使い分け
            `uvm_info("DRV", $sformatf("Driving transaction: addr=0x%h",
                txn.addr), UVM_MEDIUM)

            `uvm_info("DRV_DETAIL", $sformatf("Full txn: %s",
                txn.sprint()), UVM_HIGH)

            // ドライブ処理
            drive_item(txn);

            // エラー検出時
            if (txn.error) begin
                `uvm_warning("DRV", $sformatf(
                    "Transaction error detected: addr=0x%h, resp=%s",
                    txn.addr, txn.resp.name()))
            end

            seq_item_port.item_done();
        end
    endtask
endclass
```

UVMのverbosityレベルは以下の通りです。

| レベル | 定数 | 値 | 用途 |
|--------|------|-----|------|
| なし | `UVM_NONE` | 0 | 常に表示 |
| 低 | `UVM_LOW` | 100 | 基本的な進捗情報 |
| 中 | `UVM_MEDIUM` | 200 | 標準的なデバッグ情報（デフォルト） |
| 高 | `UVM_HIGH` | 300 | 詳細なデバッグ情報 |
| 完全 | `UVM_FULL` | 400 | すべての詳細 |
| デバッグ | `UVM_DEBUG` | 500 | 開発者向けの最詳細情報 |

verbosityの制御はコマンドラインから行えます。

```systemverilog
// コマンドライン例（VCS）:
// simv +UVM_VERBOSITY=UVM_HIGH
// simv +UVM_VERBOSITY=UVM_DEBUG

// 特定コンポーネントのverbosityを変更
// simv +uvm_set_verbosity=uvm_test_top.env.agent.driver,DRV,UVM_DEBUG,run

// テストベンチ内からの制御
class my_test extends uvm_test;
    function void end_of_elaboration_phase(uvm_phase phase);
        // 特定コンポーネントのverbosityを上げる
        uvm_component drv;
        drv = uvm_top.find("*.driver");
        if (drv != null) begin
            drv.set_report_verbosity_level(UVM_DEBUG);
        end

        // 特定IDのメッセージをファイルに出力
        env.agent.monitor.set_report_severity_action_hier(
            UVM_INFO, UVM_DISPLAY | UVM_LOG);
        env.agent.monitor.set_report_default_file_hier(
            monitor_log_fd);
    endfunction
endclass
```

UVMのレポートキャッチャーを使ったカスタムフィルタリングも強力です。

```systemverilog
class error_catcher extends uvm_report_catcher;
    int error_count;
    string error_log[$];

    function new(string name = "error_catcher");
        super.new(name);
        error_count = 0;
    endfunction

    function action_e catch();
        if (get_severity() == UVM_ERROR) begin
            error_count++;
            error_log.push_back($sformatf("[%0t] %s: %s",
                $time, get_id(), get_message()));

            // 特定のエラーを警告にダウングレード
            if (get_id() == "KNOWN_ISSUE_123") begin
                set_severity(UVM_WARNING);
                set_action(UVM_DISPLAY);
                return THROW;
            end
        end
        return THROW;  // 通常通り処理を続行
    endfunction

    function void report_summary();
        $display("=== Error Summary: %0d errors ===", error_count);
        foreach (error_log[i]) begin
            $display("  %s", error_log[i]);
        end
    endfunction
endclass
```

### 16.4.3 メモリリーク・ハンドルリークの検出

SystemVerilogのクラスベース検証環境では、メモリリークが問題になることがあります。

```systemverilog
// メモリ使用量トラッカー
class memory_tracker;
    static int object_count[string];  // クラス名ごとのオブジェクト数
    static int total_allocated;
    static int total_freed;

    static function void track_alloc(string class_name);
        if (object_count.exists(class_name))
            object_count[class_name]++;
        else
            object_count[class_name] = 1;
        total_allocated++;
    endfunction

    static function void track_free(string class_name);
        if (object_count.exists(class_name))
            object_count[class_name]--;
        total_freed++;
    endfunction

    static function void report();
        $display("=== Memory Tracker Report ===");
        $display("Total allocated: %0d", total_allocated);
        $display("Total freed:     %0d", total_freed);
        $display("Potential leaks: %0d", total_allocated - total_freed);
        $display("--- Per-class breakdown ---");
        foreach (object_count[cls]) begin
            if (object_count[cls] > 0)
                $display("  %s: %0d objects still alive", cls, object_count[cls]);
        end
        $display("============================");
    endfunction
endclass

// 使用例：トラッキング対象のクラス
class tracked_transaction extends uvm_sequence_item;
    `uvm_object_utils(tracked_transaction)

    function new(string name = "tracked_transaction");
        super.new(name);
        memory_tracker::track_alloc("tracked_transaction");
    endfunction

    // Note: SystemVerilogにはデストラクタがないため、
    // 明示的なクリーンアップメソッドを使用
    function void cleanup();
        memory_tracker::track_free("tracked_transaction");
    endfunction
endclass
```

メモリリークを防ぐためのベストプラクティスは以下の通りです。

- 使い終わったオブジェクトへの参照を `null` にして、ガベージコレクションを促進します
- キューや連想配列に蓄積されるオブジェクトのサイズを定期的に監視します
- `uvm_pool` や `uvm_queue` の使用時は、不要なエントリを削除します
- スコーピングを意識し、ローカル変数のオブジェクトはスコープ外で自動解放されることを利用します

### 16.4.4 パフォーマンスプロファイリング

シミュレーション速度の最適化は大規模プロジェクトにおいて重要です。

```systemverilog
// シミュレーション性能計測モジュール
module perf_counter;
    // 計測用変数
    real    start_time;
    real    end_time;
    integer cycle_count;
    real    cycles_per_second;

    // ウォールクロック時間の計測
    initial begin
        start_time = $realtime;
        cycle_count = 0;
    end

    // サイクルカウント
    always @(posedge clk) begin
        cycle_count <= cycle_count + 1;
    end

    // 定期的なパフォーマンスレポート
    always @(posedge clk) begin
        if (cycle_count > 0 && cycle_count % 100000 == 0) begin
            end_time = $realtime;
            if (end_time > start_time) begin
                cycles_per_second = cycle_count / ((end_time - start_time) / 1e9);
                $display("[PERF] %0d cycles, %.1f kcycles/sec",
                    cycle_count, cycles_per_second / 1000.0);
            end
        end
    end

    // 終了時のサマリ
    final begin
        end_time = $realtime;
        $display("=== Performance Summary ===");
        $display("Total cycles: %0d", cycle_count);
    end
endmodule
```

パフォーマンスのボトルネックとなる一般的な原因を以下にまとめます。

| ボトルネック | 影響度 | 対策 |
|------------|--------|------|
| 大量の`$display`出力 | 高 | verbosityレベルで制御 |
| 全信号の波形ダンプ | 高 | ダンプ範囲を限定 |
| 巨大な連想配列 | 中 | サイズ制限、定期クリーンアップ |
| 複雑なSVA | 中 | 不要なアサーションの無効化 |
| DPI-Cの頻繁な呼び出し | 中 | バッチ処理化 |
| ゼロ遅延ループ | 高 | ロジックの見直し |
| 大きなメモリモデル | 高 | スパースメモリの使用 |

### 16.4.5 自動回帰テストのデバッグ戦略

CI/CD環境での大量テスト実行時に、効率的にデバッグを進めるための戦略を解説します。

![デバッグ戦略フロー](/images/systemverilog-complete-guide/ch15_debug_strategy.drawio.png)

```systemverilog
// テスト結果の自動記録
class test_result_recorder;
    string test_name;
    integer result_fd;

    // 構造化されたテスト結果の出力
    function void record_result(
        string status,  // PASS, FAIL, ERROR
        string message,
        int    error_count,
        int    warning_count
    );
        // JSON形式で結果を記録
        result_fd = $fopen({test_name, "_result.json"}, "w");
        $fdisplay(result_fd, "{");
        $fdisplay(result_fd, "  \"test_name\": \"%s\",", test_name);
        $fdisplay(result_fd, "  \"status\": \"%s\",", status);
        $fdisplay(result_fd, "  \"message\": \"%s\",", message);
        $fdisplay(result_fd, "  \"error_count\": %0d,", error_count);
        $fdisplay(result_fd, "  \"warning_count\": %0d,", warning_count);
        $fdisplay(result_fd, "  \"sim_time\": %0t,", $time);
        $fdisplay(result_fd, "  \"timestamp\": \"%s\"", get_timestamp());
        $fdisplay(result_fd, "}");
        $fclose(result_fd);
    endfunction

    function string get_timestamp();
        // シミュレーション終了時刻をフォーマット
        return $sformatf("%0t", $time);
    endfunction
endclass

// シード値の記録（ランダムテストの再現性）
module seed_logger;
    initial begin
        integer seed;
        string seed_file;

        // 現在のシード値を取得
        seed = $urandom;

        // シード値をファイルに記録
        seed_file = "random_seed.log";
        $system($sformatf("echo 'Seed: %0d' >> %s", seed, seed_file));

        $display("Random seed: %0d", seed);
    end
endmodule
```

自動回帰テストのデバッグでは、以下の戦略が効果的です。

1. **失敗の分類**: エラーメッセージをパターンで分類し、同一原因の失敗をグループ化します
2. **最小再現セット**: 失敗を再現する最小限のテスト条件を特定します
3. **シード値管理**: ランダムテストのシード値を記録し、失敗の再現を可能にします
4. **差分解析**: 成功時と失敗時のログを比較して差分を特定します
5. **段階的な絞り込み**: verbosityを段階的に上げて、問題の箇所を特定します

---

## 16.5 デバッグ環境の構築

### 16.5.1 Makefileベースのデバッグ環境

効率的なデバッグのためには、統一されたビルド・実行環境が不可欠です。

```c
// Makefile 例
// =========================================
// デバッグ環境用 Makefile
// =========================================

// SIMULATOR = vcs
// TOP_MODULE = testbench
// DESIGN_FILES = rtl/design.sv rtl/submodule.sv
// TB_FILES = tb/testbench.sv tb/monitor.sv
// VPI_SRC = vpi/my_vpi.c
// VPI_LIB = my_vpi.so

// # コンパイルオプション
// COMPILE_OPTS = -sverilog -timescale=1ns/1ps
// DEBUG_OPTS = +define+DEBUG_ON +define+ASSERT_ON
// WAVE_OPTS = -debug_access+all

// # デフォルトターゲット
// all: compile

// # コンパイル（通常モード）
// compile:
//     $(SIMULATOR) $(COMPILE_OPTS) $(DESIGN_FILES) $(TB_FILES) -o simv

// # コンパイル（デバッグモード）
// compile_debug:
//     $(SIMULATOR) $(COMPILE_OPTS) $(DEBUG_OPTS) $(WAVE_OPTS) \
//         $(DESIGN_FILES) $(TB_FILES) -o simv_debug

// # VPIライブラリのビルド
// vpi_lib: $(VPI_SRC)
//     gcc -shared -fPIC -o $(VPI_LIB) $(VPI_SRC) \
//         -I$(VCS_HOME)/include

// # シミュレーション実行（通常）
// run: compile
//     ./simv +seed=random

// # シミュレーション実行（波形付き）
// run_wave: compile_debug
//     ./simv_debug +fsdbfile+wave.fsdb +seed=random

// # シミュレーション実行（VPI付き）
// run_vpi: compile_debug vpi_lib
//     ./simv_debug +vpi -load ./$(VPI_LIB) +seed=random

// # 波形ビューワの起動
// verdi:
//     verdi -ssf wave.fsdb -sv $(DESIGN_FILES) $(TB_FILES) &

// # ログの解析
// analyze_log:
//     @echo "=== Error Summary ==="
//     @grep -c "ERROR" simulation.log || echo "No errors"
//     @echo "=== Warning Summary ==="
//     @grep -c "WARNING" simulation.log || echo "No warnings"

// # クリーンアップ
// clean:
//     rm -rf simv* csrc *.fsdb *.vcd *.log DVEfiles verdiLog

// .PHONY: all compile compile_debug run run_wave run_vpi verdi analyze_log clean
```

### 16.5.2 スクリプトによる自動ログ解析

シミュレーションログの自動解析は、大量のテスト結果を効率的に処理するために必要です。以下にPythonスクリプトの例を示します。

```c
// ログ解析の基本的なアプローチ:
//
// 1. エラーパターンの抽出
//    - "ERROR", "FATAL", "FAIL" 等のキーワード検索
//    - タイムスタンプとコンテキスト情報の紐付け
//
// 2. 統計情報の生成
//    - エラー種別ごとの出現回数
//    - 最初のエラー発生時刻
//    - テスト成功/失敗率
//
// 3. レポート生成
//    - HTML/テキスト形式のサマリレポート
//    - 失敗テストの一覧
```

SystemVerilog側でも、ログ解析しやすい構造化出力を心がけます。

```systemverilog
// 構造化ログ出力クラス
class structured_logger;
    static integer log_fd;
    static string  component_name;

    static function void init(string name, string filename);
        component_name = name;
        log_fd = $fopen(filename, "w");
        log_header();
    endfunction

    static function void log_header();
        $fdisplay(log_fd, "TIMESTAMP,LEVEL,COMPONENT,ID,MESSAGE");
    endfunction

    static function void log_msg(
        string level,
        string id,
        string message
    );
        // CSV形式で出力（後処理しやすい）
        $fdisplay(log_fd, "%0t,%s,%s,%s,%s",
            $time, level, component_name, id, message);

        // コンソールにも表示
        if (level == "ERROR" || level == "FATAL") begin
            $display("[%0t] [%s] [%s] %s: %s",
                $time, level, component_name, id, message);
        end
    endfunction

    static function void close();
        $fclose(log_fd);
    endfunction
endclass

// 使用例
module scoreboard;
    initial begin
        structured_logger::init("SCOREBOARD", "scoreboard.csv");
    end

    task check_result(logic [31:0] actual, logic [31:0] expected);
        if (actual === expected) begin
            structured_logger::log_msg("INFO", "CHECK",
                $sformatf("PASS: 0x%h == 0x%h", actual, expected));
        end else begin
            structured_logger::log_msg("ERROR", "CHECK",
                $sformatf("FAIL: actual=0x%h, expected=0x%h",
                    actual, expected));
        end
    endtask

    final begin
        structured_logger::close();
    end
endmodule
```

### 16.5.3 CI/CDパイプラインでのデバッグ

継続的インテグレーション環境でのデバッグは、対話的なデバッグとは異なるアプローチが必要です。

```systemverilog
// CI/CD向けのテスト制御モジュール
module ci_test_control;
    // 環境変数からテスト設定を読み込む
    string test_name;
    string wave_enable;
    int    max_cycles;
    int    verbosity;

    initial begin
        // プラスアーグからパラメータを取得
        if (!$value$plusargs("TEST_NAME=%s", test_name))
            test_name = "default_test";

        if (!$value$plusargs("MAX_CYCLES=%d", max_cycles))
            max_cycles = 100000;

        if (!$value$plusargs("VERBOSITY=%d", verbosity))
            verbosity = 2;  // INFO レベル

        $display("=== CI Test Configuration ===");
        $display("Test Name:  %s", test_name);
        $display("Max Cycles: %0d", max_cycles);
        $display("Verbosity:  %0d", verbosity);
        $display("=============================");
    end

    // タイムアウト検出
    integer cycle_count = 0;
    always @(posedge clk) begin
        cycle_count <= cycle_count + 1;
        if (cycle_count >= max_cycles) begin
            $display("TIMEOUT: Test exceeded %0d cycles", max_cycles);
            $display("TEST_RESULT: TIMEOUT");
            $finish;
        end
    end

    // テスト結果の標準出力（CI/CDパーサーで解析しやすい形式）
    function void report_pass();
        $display("TEST_RESULT: PASS");
        $display("TEST_NAME: %s", test_name);
        $display("CYCLES: %0d", cycle_count);
    endfunction

    function void report_fail(string reason);
        $display("TEST_RESULT: FAIL");
        $display("TEST_NAME: %s", test_name);
        $display("FAIL_REASON: %s", reason);
        $display("FAIL_CYCLE: %0d", cycle_count);
        $display("FAIL_TIME: %0t", $time);
    endfunction
endmodule
```

CI/CDでのデバッグワークフローは以下のようになります。

1. **初回実行**: 波形ダンプなしで高速実行します
2. **失敗検出**: テスト結果を自動解析し、失敗テストを特定します
3. **再実行**: 失敗テストのみ、波形ダンプ付きで再実行します
4. **アーティファクト保存**: 波形ファイル、ログファイルをCI/CDのアーティファクトとして保存します
5. **ローカルデバッグ**: 開発者がアーティファクトをダウンロードしてローカルで解析します

### 16.5.4 チーム開発でのデバッグ共有

チーム開発では、デバッグ情報の共有が生産性に大きく影響します。

```systemverilog
// デバッグ設定ファイル（チーム共通）
// debug_config.svh

// デバッグレベル定義
`define DBG_LEVEL_BASIC  1
`define DBG_LEVEL_DETAIL 2
`define DBG_LEVEL_TRACE  3

// モジュール別デバッグイネーブル
// 使い方: +define+DBG_MODULE_CTRL で個別に有効化
`ifdef DBG_MODULE_CTRL
    `define DBG_CTRL_LEVEL `DBG_LEVEL_DETAIL
`else
    `define DBG_CTRL_LEVEL 0
`endif

`ifdef DBG_MODULE_DATAPATH
    `define DBG_DP_LEVEL `DBG_LEVEL_DETAIL
`else
    `define DBG_DP_LEVEL 0
`endif

// 共通デバッグマクロ
`define DBG_MSG(level, module_level, msg) \
    if (level <= module_level) \
        $display("[%0t] [%m] %s", $time, msg)

`define DBG_BASIC(module_level, msg)  `DBG_MSG(1, module_level, msg)
`define DBG_DETAIL(module_level, msg) `DBG_MSG(2, module_level, msg)
`define DBG_TRACE(module_level, msg)  `DBG_MSG(3, module_level, msg)
```

波形設定の共有も重要です。

```systemverilog
// 波形ビューワの設定をスクリプトで生成
// signal_list.tcl（Verdi用）

// # AXI Write Channel
// addSignal -h 20 /testbench/dut/aclk
// addSignal /testbench/dut/awvalid
// addSignal /testbench/dut/awready
// addSignal -radix hex /testbench/dut/awaddr
// addSignal -radix unsigned /testbench/dut/awlen
// addGroup "Write Data"
// addSignal /testbench/dut/wvalid
// addSignal /testbench/dut/wready
// addSignal -radix hex /testbench/dut/wdata
// addSignal /testbench/dut/wlast
// addGroup "Write Response"
// addSignal /testbench/dut/bvalid
// addSignal /testbench/dut/bready
// addSignal /testbench/dut/bresp
```

デバッグ共有のベストプラクティスをまとめます。

| 項目 | 推奨事項 |
|------|---------|
| デバッグマクロ | チーム共通のヘッダファイルに定義し、バージョン管理します |
| 波形設定 | 信号グループ設定をスクリプト化し、リポジトリに格納します |
| アサーション | `bind` ファイルとして独立管理し、段階的に追加します |
| ログフォーマット | 構造化フォーマット（CSV/JSON）を統一し、解析ツールを共有します |
| デバッグ手順書 | よくある問題とその対処法をWikiやドキュメントに記録します |
| VPIツール | 共有ライブラリとしてビルドし、チームで利用できるようにします |

---

## 16.6 まとめ

本章では、VPI（Verilog Procedural Interface）とSystemVerilogにおける各種デバッグ技法を学びました。主要なポイントを振り返ります。

**VPIについて**:
- VPIはC言語からシミュレーション内部にアクセスするための標準インターフェースです
- `vpiHandle` によるオブジェクトモデルと、一貫した `vpi_` プレフィックスのAPI体系を持ちます
- カスタムシステムタスクの作成、信号モニタリング、設計階層の走査が可能です
- DPI-Cとは目的が異なり、シミュレーション制御にはVPIが適しています

**デバッグ基本技法**:
- `$display`, `$monitor`, `$strobe` はそれぞれ異なるタイミングで動作するため、使い分けが重要です
- ファイルログ出力とverbosityレベルの制御により、効率的なデバッグが可能になります
- `ifdef` ディレクティブでデバッグコードの有効/無効を切り替えられます

**波形デバッグ**:
- VCDは標準フォーマットで広く対応していますが、大規模設計にはFSDBが適しています
- ダンプ範囲の制御はシミュレーション速度に直結するため、適切な設定が必要です
- ブックマーク信号やデバッグ用インターフェースを活用して波形解析を効率化できます

**アサーションベースデバッグ**:
- SVAとbind構文を組み合わせることで、RTLを変更せずにアサーションを追加できます
- フォーマル検証ツールとの連携も視野に入れたアサーション記述が望ましいです

**高度なデバッグ**:
- UVMのレポーティング機能は、大規模検証環境での標準的なデバッグ手段です
- CI/CDパイプラインでの自動テストには、構造化されたログ出力と再現性の確保が重要です
- チーム開発では、デバッグ設定やツールの共有が生産性向上の鍵となります

---

## 演習問題

**問題1**: VPIを使って、指定したモジュール内のすべてのレジスタ（`reg`/`logic`）の名前、ビット幅、現在値を一覧表示するシステムタスク `$list_regs(module_path)` を実装してください。C言語のソースコード全体を記述し、`vlog_startup_routines` の定義も含めてください。

**問題2**: 以下の `$display` と `$strobe` の出力結果を予測してください。

```systemverilog
module quiz;
    logic [7:0] a, b;

    initial begin
        a = 8'h10;
        b = 8'h20;

        a <= 8'hAA;
        b <= a + b;

        $display("[display] a=%h, b=%h", a, b);
        $strobe("[strobe]  a=%h, b=%h", a, b);

        #1;
        $display("[next]    a=%h, b=%h", a, b);
    end
endmodule
```

**問題3**: 以下の仕様を持つ `sync_fifo` モジュールに対して、`bind` を使って挿入するアサーションモジュールを設計してください。検証すべきプロパティは以下の通りです。
- FIFO満杯時の書き込みが発生しないこと
- FIFO空時の読み出しが発生しないこと
- 書き込みデータと読み出しデータの順序が保証されること（FIFO順序性）
- リセット後にFIFOが空状態になること

**問題4**: VCDダンプのパフォーマンス最適化について、以下の状況でどのようなダンプ設定が最適か、理由とともに説明してください。
1. 初回デバッグ時（問題箇所が不明）
2. 問題箇所が特定のモジュール内と判明している場合
3. 問題が特定の時間範囲で発生することが分かっている場合
4. CI/CDパイプラインでの回帰テスト時

**問題5**: UVMの `uvm_report_catcher` を拡張して、以下の機能を持つカスタムレポートキャッチャーを実装してください。
- 同一IDのエラーが10回以上繰り返された場合、以降のメッセージを抑制する
- 抑制されたメッセージの数をシミュレーション終了時にレポートする
- 特定のIDリスト（既知のバグ）に該当するエラーを警告にダウングレードする

**問題6**: チーム開発を想定し、以下の要件を満たすデバッグ環境を設計してください。Makefileのターゲット構成、デバッグマクロの定義、ログ解析の仕組みを含めた全体アーキテクチャを記述してください。
- 3段階のデバッグレベル（basic, detail, trace）をサポート
- モジュール別のデバッグ有効化/無効化が可能
- テスト結果をJSON形式で出力
- 失敗テストの自動再実行（波形付き）をサポート
