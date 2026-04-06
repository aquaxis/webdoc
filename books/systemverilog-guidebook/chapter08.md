---
title: "第8章：パッケージとコンパイル単位"
---

# 第8章：パッケージとコンパイル単位

## 8.1 この章で学ぶこと

本章では、SystemVerilogにおける**パッケージ（package）**、**スコープ解決演算子（`::`）**、**import文**、そして**コンパイル単位（`$unit`）**について解説します。パッケージは、定数、型定義、関数、タスクなどをまとめて再利用可能な単位として管理するための仕組みです。大規模な設計・検証プロジェクトにおいて、コードの整理と共有のために不可欠な機能です。

---

## 8.2 パッケージ（Package）とは

パッケージは、複数のモジュールやテストベンチで共有したい定義をひとまとめにするための仕組みです。ソフトウェア言語でいうところの「ライブラリ」や「名前空間」に相当します。

パッケージ内に定義できるもの：
- `parameter` / `localparam`（定数）
- `typedef`（型定義）
- `function` / `task`（関数・タスク）
- `class`（クラス）
- `enum`, `struct`（列挙型・構造体）
- 自動変数宣言

### 8.2.1 パッケージの定義

パッケージは `package` ～ `endpackage` キーワードで定義します。

```systemverilog
package bus_pkg;

    // 定数の定義
    parameter int ADDR_WIDTH = 32;
    parameter int DATA_WIDTH = 64;
    parameter int BURST_LEN  = 8;

    // 列挙型の定義
    typedef enum logic [1:0] {
        IDLE   = 2'b00,
        READ   = 2'b01,
        WRITE  = 2'b10,
        ERROR  = 2'b11
    } bus_state_t;

    // 構造体の定義
    typedef struct packed {
        logic [ADDR_WIDTH-1:0] addr;
        logic [DATA_WIDTH-1:0] data;
        logic                  wr_en;
        logic                  rd_en;
    } bus_transaction_t;

    // 関数の定義
    function automatic logic is_aligned(logic [ADDR_WIDTH-1:0] addr, int alignment);
        return (addr % alignment == 0);
    endfunction

endpackage
```

### 8.2.2 パッケージの利用目的

パッケージを使うことで、以下のメリットが得られます：

1. **定義の一元管理**: 定数や型を1か所で管理し、変更時の影響範囲を最小化
2. **名前の衝突防止**: パッケージがスコープを提供し、同名の定義が別パッケージに存在しても問題なし
3. **再利用性の向上**: 複数のプロジェクトで共通パッケージを共有可能
4. **コードの整理**: 関連する定義をグループ化して見通しを良くする

![パッケージの概念](/images/systemverilog-complete-guide/ch08_package_concept.drawio.png)

---

## 8.3 パッケージの利用方法

パッケージに定義された要素を使用するには、**スコープ解決演算子（`::`）** または **`import` 文**を使います。

### 8.3.1 スコープ解決演算子（`::`）

`パッケージ名::要素名` の形式で直接参照します。どのパッケージの要素を使っているかが明確なため、可読性に優れています。

```systemverilog
module bus_master (
    input  logic                           clk,
    input  logic                           rst_n,
    output logic [bus_pkg::ADDR_WIDTH-1:0] addr,
    output logic [bus_pkg::DATA_WIDTH-1:0] wdata,
    input  logic [bus_pkg::DATA_WIDTH-1:0] rdata
);

    bus_pkg::bus_state_t current_state;
    bus_pkg::bus_transaction_t txn;

    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n)
            current_state <= bus_pkg::IDLE;
        else begin
            case (current_state)
                bus_pkg::IDLE:  current_state <= bus_pkg::READ;
                bus_pkg::READ:  current_state <= bus_pkg::WRITE;
                bus_pkg::WRITE: current_state <= bus_pkg::IDLE;
                default:        current_state <= bus_pkg::ERROR;
            endcase
        end
    end

endmodule
```

### 8.3.2 import 文

`import` 文を使うと、パッケージ名の接頭辞なしで要素にアクセスできます。記述が簡潔になりますが、どのパッケージから来た要素かが分かりにくくなる場合もあるため注意が必要です。

#### 個別インポート

特定の要素だけをインポートします。

```systemverilog
module bus_slave;
    import bus_pkg::bus_state_t;        // 型のみインポート
    import bus_pkg::bus_transaction_t;  // 構造体のみインポート
    import bus_pkg::ADDR_WIDTH;         // 定数のみインポート

    bus_state_t state;  // bus_pkg:: なしで使える
    // ...
endmodule
```

#### ワイルドカードインポート

パッケージ内のすべての要素をインポートします。

```systemverilog
module bus_controller;
    import bus_pkg::*;  // bus_pkg内のすべてをインポート

    bus_state_t       state;
    bus_transaction_t txn;
    logic [ADDR_WIDTH-1:0] address;

    initial begin
        if (is_aligned(address, 4))
            $display("Address is 4-byte aligned");
    end
endmodule
```

### 8.3.3 import の注意点

ワイルドカードインポート（`::*`）は便利ですが、以下の点に注意が必要です：

1. **名前の衝突**: 複数のパッケージから同名の要素をインポートするとコンパイルエラーになる
2. **可読性の低下**: 要素の出所が不明確になる
3. **推奨**: 大規模プロジェクトでは個別インポートまたは `::` 演算子を推奨

```systemverilog
// 名前衝突の例
package pkg_a;
    typedef int data_t;
endpackage

package pkg_b;
    typedef logic [7:0] data_t;  // 同じ名前！
endpackage

module conflict_example;
    import pkg_a::*;
    import pkg_b::*;
    // data_t x;  // エラー！どちらのdata_tか不明

    // 解決方法1: 明示的な ::
    pkg_a::data_t x;
    pkg_b::data_t y;

    // 解決方法2: 個別インポートで優先順位を明確化
endmodule
```

---

## 8.4 パッケージの設計パターン

### 8.4.1 プロジェクト共通パッケージ

プロジェクト全体で使用する定数や型をまとめたパッケージは、非常に一般的な設計パターンです。

```systemverilog
package project_pkg;

    // プロジェクト全体の定数
    parameter int CLK_FREQ_MHZ   = 100;
    parameter int RESET_CYCLES   = 10;
    parameter int TIMEOUT_CYCLES = 100000;

    // 共通の型定義
    typedef logic [31:0] addr_t;
    typedef logic [63:0] data_t;
    typedef logic [7:0]  byte_t;

    // エラーレベル
    typedef enum {
        SEVERITY_INFO,
        SEVERITY_WARNING,
        SEVERITY_ERROR,
        SEVERITY_FATAL
    } severity_t;

    // ユーティリティ関数
    function automatic string severity_to_string(severity_t s);
        case (s)
            SEVERITY_INFO:    return "INFO";
            SEVERITY_WARNING: return "WARNING";
            SEVERITY_ERROR:   return "ERROR";
            SEVERITY_FATAL:   return "FATAL";
            default:          return "UNKNOWN";
        endcase
    endfunction

endpackage
```

### 8.4.2 インターフェース固有パッケージ

特定のバスプロトコルに関連する定義をまとめたパッケージも効果的です。

```systemverilog
package axi_pkg;

    // AXIバースト長
    parameter int AXI_MAX_BURST = 256;

    // バーストタイプ
    typedef enum logic [1:0] {
        BURST_FIXED = 2'b00,
        BURST_INCR  = 2'b01,
        BURST_WRAP  = 2'b10
    } axi_burst_t;

    // 応答タイプ
    typedef enum logic [1:0] {
        RESP_OKAY   = 2'b00,
        RESP_EXOKAY = 2'b01,
        RESP_SLVERR = 2'b10,
        RESP_DECERR = 2'b11
    } axi_resp_t;

    // AXIトランザクション
    typedef struct {
        logic [31:0]  addr;
        logic [7:0]   len;
        logic [2:0]   size;
        axi_burst_t   burst;
        logic [31:0]  data[];
    } axi_txn_t;

endpackage
```

### 8.4.3 テストベンチ用パッケージ

テスト環境で必要な型やユーティリティをまとめるパターンです。

```systemverilog
package tb_pkg;

    import project_pkg::*;

    // テスト結果
    typedef enum {
        TEST_PASS,
        TEST_FAIL,
        TEST_SKIP
    } test_result_t;

    // テストカウンタ
    int total_tests  = 0;
    int passed_tests = 0;
    int failed_tests = 0;

    // アサーション関数
    function automatic void check_equal(
        string name,
        logic [63:0] actual,
        logic [63:0] expected
    );
        total_tests++;
        if (actual === expected) begin
            passed_tests++;
            $display("[PASS] %s: got 0x%h", name, actual);
        end else begin
            failed_tests++;
            $error("[FAIL] %s: expected 0x%h, got 0x%h", name, expected, actual);
        end
    endfunction

    // テストサマリ
    function automatic void print_summary();
        $display("==================================");
        $display("  Test Summary");
        $display("  Total:  %0d", total_tests);
        $display("  Passed: %0d", passed_tests);
        $display("  Failed: %0d", failed_tests);
        $display("==================================");
    endfunction

endpackage
```

---

## 8.5 コンパイル単位（$unit）

### 8.5.1 $unit とは

`$unit` は、**コンパイル単位スコープ**（compilation unit scope）と呼ばれる暗黙のスコープを指します。同一コンパイル単位内（通常は同一ファイル内、またはコンパイラが同時に処理するファイル群）のトップレベルに宣言された要素は、`$unit` スコープに属します。

```systemverilog
// ファイルのトップレベルに宣言された変数や型は $unit に属する
typedef logic [7:0] byte_t;      // $unit スコープ
parameter int WIDTH = 16;        // $unit スコープ

module my_module;
    byte_t data;                 // $unit::byte_t を使用
    logic [WIDTH-1:0] bus;       // $unit::WIDTH を使用
endmodule
```

### 8.5.2 $unit のスコープルール

`$unit` に宣言された要素は、同じコンパイル単位内のすべてのモジュール、インターフェース、プログラムブロックから参照できます。ただし、これは**暗黙的なグローバル変数**に似た動作であり、以下の問題を引き起こす可能性があります：

1. **コンパイル順序への依存**: `$unit` の内容はコンパイル単位の定義に依存する
2. **ツール間の非互換性**: コンパイル単位の扱いはツールによって異なる場合がある
3. **意図しない名前衝突**: 異なるファイルで同名の `$unit` 宣言がありうる

```systemverilog
// file_a.sv
typedef enum {RED, GREEN, BLUE} color_t;  // $unit

// file_b.sv
typedef enum {RED, YELLOW, BLUE} light_t; // $unit: RED, BLUEが衝突する可能性！

// ★ これを防ぐためにパッケージを使うべき
package color_pkg;
    typedef enum {RED, GREEN, BLUE} color_t;
endpackage

package light_pkg;
    typedef enum {RED, YELLOW, BLUE} light_t;
endpackage
```

### 8.5.3 $unit を避けるべき理由

`$unit` を使ったグローバル宣言は、小規模なテストでは便利ですが、以下の理由で大規模プロジェクトでは**避けるべき**とされています：

- **移植性の問題**: コンパイル順序やファイルの分割方法によって動作が変わる
- **保守性の低下**: 宣言がどのファイルにあるか追跡しにくい
- **パッケージの優位性**: パッケージは同じ機能を提供しつつ、明示的で安全

**ベストプラクティス**: `$unit` の代わりにパッケージを使用し、`import` または `::` で明示的に参照しましょう。

---

## 8.6 パッケージのコンパイルと依存関係

### 8.6.1 コンパイル順序

パッケージを使用するモジュールよりも**先に**パッケージをコンパイルする必要があります。一般的なコンパイル順序は以下のとおりです：

```
1. パッケージファイル（pkg.sv）
2. インターフェースファイル（if.sv）
3. 設計モジュール（rtl.sv）
4. テストベンチ（tb.sv）
```

コマンドラインでのコンパイル例：

```bash
# VCSの場合
vcs -sverilog bus_pkg.sv axi_pkg.sv top_module.sv tb_top.sv

# Questaの場合
vlog bus_pkg.sv axi_pkg.sv top_module.sv tb_top.sv

# Verilatorの場合
verilator --sv bus_pkg.sv top_module.sv --top-module top_module
```

### 8.6.2 パッケージ間の依存

パッケージが他のパッケージに依存する場合、依存先を先にコンパイルし、`import` で取り込みます。

```systemverilog
// base_pkg.sv - 基底パッケージ
package base_pkg;
    parameter int MAX_SIZE = 256;
    typedef logic [31:0] word_t;
endpackage

// derived_pkg.sv - base_pkgに依存するパッケージ
package derived_pkg;
    import base_pkg::*;  // base_pkgを取り込む

    typedef struct packed {
        word_t  header;   // base_pkg::word_t を使用
        word_t  payload;
        logic [7:0] checksum;
    } packet_t;

    function automatic logic [7:0] calc_checksum(word_t header, word_t payload);
        return header[7:0] ^ payload[7:0];
    endfunction

endpackage
```

### 8.6.3 循環依存の回避

パッケージAがパッケージBをインポートし、パッケージBがパッケージAをインポートする**循環依存**は許可されていません。このような状況では、共通部分を別の基底パッケージに抽出して解決します。

```systemverilog
// NG: 循環依存
// package A; import B::*; endpackage
// package B; import A::*; endpackage

// OK: 共通基底パッケージで解決
package common_pkg;
    typedef logic [31:0] data_t;
endpackage

package pkg_a;
    import common_pkg::*;
    // ...
endpackage

package pkg_b;
    import common_pkg::*;
    // ...
endpackage
```

![パッケージの依存関係](/images/systemverilog-complete-guide/ch08_package_dependency.drawio.png)

---

## 8.7 実践的なパッケージ活用例

### 8.7.1 SoC設計でのパッケージ構成

大規模なSoC（System on Chip）設計では、以下のようなパッケージ構成が一般的です。

```systemverilog
// soc_pkg.sv - SoC全体の共通定義
package soc_pkg;
    // アドレスマップ
    parameter logic [31:0] UART_BASE   = 32'h4000_0000;
    parameter logic [31:0] SPI_BASE    = 32'h4001_0000;
    parameter logic [31:0] GPIO_BASE   = 32'h4002_0000;
    parameter logic [31:0] TIMER_BASE  = 32'h4003_0000;
    parameter logic [31:0] MEM_BASE    = 32'h8000_0000;

    // 割り込み番号
    typedef enum int {
        IRQ_UART  = 0,
        IRQ_SPI   = 1,
        IRQ_GPIO  = 2,
        IRQ_TIMER = 3
    } irq_id_t;

    // バスアクセス関数
    function automatic logic in_range(
        logic [31:0] addr,
        logic [31:0] base,
        int          size
    );
        return (addr >= base) && (addr < base + size);
    endfunction
endpackage
```

### 8.7.2 パッケージとインターフェースの組み合わせ

パッケージで定義した型をインターフェースで使用することで、統一された型安全なデザインが実現できます。

```systemverilog
package mem_pkg;
    parameter int ADDR_W = 20;
    parameter int DATA_W = 32;

    typedef enum logic [2:0] {
        CMD_NOP   = 3'b000,
        CMD_READ  = 3'b001,
        CMD_WRITE = 3'b010,
        CMD_BURST = 3'b011
    } mem_cmd_t;
endpackage

interface mem_if;
    import mem_pkg::*;

    logic [ADDR_W-1:0] addr;
    logic [DATA_W-1:0] wdata;
    logic [DATA_W-1:0] rdata;
    mem_cmd_t          cmd;
    logic              valid;
    logic              ready;

    modport master (
        output addr, wdata, cmd, valid,
        input  rdata, ready
    );

    modport slave (
        input  addr, wdata, cmd, valid,
        output rdata, ready
    );
endinterface
```

---

## 8.8 まとめ

本章では、パッケージとコンパイル単位について学びました。重要なポイントを振り返ります。

1. **パッケージ（package）** は定数、型、関数、クラスなどをまとめて管理する単位であり、コードの再利用性と整理に不可欠。
2. **スコープ解決演算子（`::`）** は出所を明確にし、名前衝突を防ぐ。大規模プロジェクトでの推奨方法。
3. **`import` 文**はワイルドカード（`::*`）と個別指定の2種類がある。便利だが名前衝突に注意。
4. **`$unit`**（コンパイル単位スコープ）はトップレベル宣言のスコープだが、移植性と保守性の問題からパッケージの使用が推奨される。
5. **コンパイル順序**ではパッケージを先にコンパイルする必要がある。循環依存は避けること。
6. **設計パターン**として、プロジェクト共通パッケージ、インターフェース固有パッケージ、テストベンチ用パッケージなどがある。

次章からは第4部「検証編」に入り、SystemVerilogのオブジェクト指向プログラミング機能について学びます。
