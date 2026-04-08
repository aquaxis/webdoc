---
title: "第 8 章：パッケージとコンパイル単位"
---

# 第 8 章：パッケージとコンパイル単位

## 8.1 この章で学ぶこと

本章では、 SystemVerilog における**パッケージ（package）**、**スコープ解決演算子（`::`）**、**import 文**、そして**コンパイル単位（`$unit`）**について解説する。パッケージは、定数、型定義、関数、タスクなどをまとめて再利用可能な単位として管理するための仕組みである。大規模な設計・検証プロジェクトにおいて、コードの整理と共有のために不可欠な機能である。

---

## 8.2 パッケージ（Package）とは

パッケージは、複数のモジュールやテストベンチで共有したい定義をひとまとめにするための仕組みである。ソフトウェア言語でいうところの「ライブラリ」や「名前空間」に相当する。

パッケージ内に定義できるもの：
- `parameter` / `localparam`（定数）
- `typedef`（型定義）
- `function` / `task`（関数・タスク）
- `class`（クラス）
- `enum`, `struct`（列挙型・構造体）
- 自動変数宣言

### 8.2.1 パッケージの定義

パッケージは `package` ～ `endpackage` キーワードで定義する。

**リスト8.1: パッケージの定義**

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

パッケージを使うことで、以下のメリットが得られる：

1. **定義の一元管理**: 定数や型を 1 か所で管理し、変更時の影響範囲を最小化
2. **名前の衝突防止**: パッケージがスコープを提供し、同名の定義が別パッケージに存在しても問題なし
3. **再利用性の向上**: 複数のプロジェクトで共通パッケージを共有可能
4. **コードの整理**: 関連する定義をグループ化して見通しを良くする

![図8.1: パッケージの概念](/images/systemverilog-guidebook/ch08_package_concept.drawio.png)

---

## 8.3 パッケージの利用方法

パッケージに定義された要素を使用するには、**スコープ解決演算子（`::`）** または **`import` 文**を用いる。

### 8.3.1 スコープ解決演算子（`::`）

`パッケージ名::要素名` の形式で直接参照する。どのパッケージの要素を使っているかが明確なため、可読性に優れている。

**リスト8.2: スコープ解決演算子によるパッケージ要素の参照**

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

`import` 文を使うと、パッケージ名の接頭辞なしで要素にアクセスできる。記述が簡潔になるが、どのパッケージから来た要素かが分かりにくくなる場合もあるため注意が必要である。

#### 個別インポート

特定の要素だけをインポートする。

**リスト8.3: 個別インポート**

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

パッケージ内のすべての要素をインポートする。

**リスト8.4: ワイルドカードインポート**

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

ワイルドカードインポート（`::*`）は便利であるが、以下の点に注意が必要である：

1. **名前の衝突**: 複数のパッケージから同名の要素をインポートするとコンパイルエラーになる
2. **可読性の低下**: 要素の出所が不明確になる
3. **推奨**: 大規模プロジェクトでは個別インポートまたは `::` 演算子を推奨

**リスト8.5: 名前衝突の回避**

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

![図8.2: パッケージのインポート方法比較](/images/systemverilog-guidebook/ch08_import_methods.drawio.png)

---

## 8.4 パッケージの設計パターン

### 8.4.1 プロジェクト共通パッケージ

プロジェクト全体で使用する定数や型をまとめたパッケージは、非常に一般的な設計パターンである。

**リスト8.6: プロジェクト共通パッケージ**

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

特定のバスプロトコルに関連する定義をまとめたパッケージも効果的である。

**リスト8.7: インターフェース固有パッケージ**

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

テスト環境で必要な型やユーティリティをまとめるパターンである。

**リスト8.8: テストベンチ用パッケージ**

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

`$unit` は、**コンパイル単位スコープ**（compilation unit scope）と呼ばれる暗黙のスコープを指す。同一コンパイル単位内（通常は同一ファイル内、またはコンパイラが同時に処理するファイル群）のトップレベルに宣言された要素は、`$unit` スコープに属する。

**リスト8.9: $unitスコープの例**

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

`$unit` に宣言された要素は、同じコンパイル単位内のすべてのモジュール、インターフェース、プログラムブロックから参照できる。ただし、これは**暗黙的なグローバル変数**に似た動作であり、以下の問題を引き起こす可能性がある：

1. **コンパイル順序への依存**: `$unit` の内容はコンパイル単位の定義に依存する
2. **ツール間の非互換性**: コンパイル単位の扱いはツールによって異なる場合がある
3. **意図しない名前衝突**: 異なるファイルで同名の `$unit` 宣言がありうる

**リスト8.10: $unitの問題点とパッケージによる解決**

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

![図8.3: $unit（コンパイル単位）のスコープ](/images/systemverilog-guidebook/ch08_unit_scope.drawio.png)

### 8.5.3 $unit を避けるべき理由

`$unit` を使ったグローバル宣言は、小規模なテストでは便利であるが、以下の理由で大規模プロジェクトでは**避けるべき**とされている：

- **移植性の問題**: コンパイル順序やファイルの分割方法によって動作が変わる
- **保守性の低下**: 宣言がどのファイルにあるか追跡しにくい
- **パッケージの優位性**: パッケージは同じ機能を提供しつつ、明示的で安全

**ベストプラクティス**: `$unit` の代わりにパッケージを使用し、`import` または `::` で明示的に参照すること。

![図8.4: 名前解決の優先順位（スコープルール）](/images/systemverilog-guidebook/ch08_name_resolution.drawio.png)

---

## 8.6 パッケージのコンパイルと依存関係

### 8.6.1 コンパイル順序

パッケージを使用するモジュールよりも**先に**パッケージをコンパイルする必要がある。一般的なコンパイル順序は以下のとおりである：

```
1. パッケージファイル（pkg.sv）
2. インターフェースファイル（if.sv）
3. 設計モジュール（rtl.sv）
4. テストベンチ（tb.sv）
```

コマンドラインでのコンパイル例：

**リスト8.11: パッケージのコンパイル順序**

```bash
# VCSの場合
vcs -sverilog bus_pkg.sv axi_pkg.sv top_module.sv tb_top.sv

# Questaの場合
vlog bus_pkg.sv axi_pkg.sv top_module.sv tb_top.sv

# Verilatorの場合
verilator --sv bus_pkg.sv top_module.sv --top-module top_module
```

### 8.6.2 パッケージ間の依存

パッケージが他のパッケージに依存する場合、依存先を先にコンパイルし、`import` で取り込む。

**リスト8.12: パッケージ間の依存関係**

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

パッケージ A がパッケージ B をインポートし、パッケージ B がパッケージ A をインポートする**循環依存**は許可されていない。このような状況では、共通部分を別の基底パッケージに抽出して解決する。

**リスト8.13: 循環依存の回避**

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

![図8.5: パッケージの依存関係](/images/systemverilog-guidebook/ch08_package_dependency.drawio.png)

---

## 8.7 実践的なパッケージ活用例

### 8.7.1 SoC 設計でのパッケージ構成

大規模な SoC（System on Chip）設計では、以下のようなパッケージ構成が一般的である。

**リスト8.14: SoC設計でのパッケージ構成例**

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

パッケージで定義した型をインターフェースで使用することで、統一された型安全なデザインが実現できる。

**リスト8.15: パッケージとインターフェースの組み合わせ**

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

### 8.7.4 名前空間管理のベストプラクティス

大規模プロジェクトでは、名前の衝突や可読性の低下を防ぐために、体系的な名前空間管理が不可欠である。

**命名規則の策定**

一貫した命名規則により、名前の衝突を防ぎ、コードの可読性を向上させる。

```systemverilog
// プロジェクト共通のプレフィックス
package project_common_pkg;
    // 定数: PREFIX_CONST_NAME
    localparam int PROJECT_MAX_WIDTH = 32;

    // 型: prefix_suffix_t
    typedef logic [PROJECT_MAX_WIDTH-1:0] project_data_t;

    // 関数: prefix_verb_noun
    function automatic void project_reset_all();
        // ...
    endfunction
endpackage
```

**パッケージの階層化**

パッケージを階層的に構成し、依存関係を明確にする。

```systemverilog
// 基本型パッケージ（依存なし）
package types_pkg;
    typedef logic [31:0] word_t;
    typedef logic [7:0]  byte_t;
endpackage

// 定数パッケージ（types_pkgに依存）
package consts_pkg;
    import types_pkg::*;
    localparam word_t MAX_ADDR = 32'hFFFF_FFFF;
endpackage

// ユーティリティパッケージ（types_pkg, consts_pkgに依存）
package utils_pkg;
    import types_pkg::*;
    import consts_pkg::*;

    function automatic word_t add_words(word_t a, word_t b);
        return a + b;
    endfunction
endpackage
```

**名前衝突の回避**

複数のパッケージから同じ名前をインポートした場合の対処法。

```systemverilog
// 名前衝突の回避方法

// 方法1: スコープ解決演算子を使用（推奨）
import pkg_a::*;
import pkg_b::*;

// 衝突する名前は明示的に指定
logic value = pkg_a::counter;  // pkg_aから
logic other  = pkg_b::counter;  // pkg_bから

// 方法2: 個別インポートで選択
import pkg_a::counter;       // pkg_aのcounterを使用
// pkg_b::counterはインポートしない

// 方法3: エイリアスを使用
typedef pkg_a::status_t status_a_t;
typedef pkg_b::status_t status_b_t;
```

**推奨事項まとめ**

| 推奨事項 | 理由 |
|---------|------|
| 一貫したプレフィックス | 名前衝突を防止 |
| パッケージごとの責務分離 | 依存関係の明確化 |
| `::*`より個別インポート | 使用する名前を明示 |
| `$unit`の使用を最小限に | 移植性と保守性の確保 |
| 循環依存の回避 | コンパイルエラーの防止 |

---

## 8.8 まとめ

本章では、パッケージとコンパイル単位について学んだ。重要なポイントを振り返る。

1. **パッケージ（package）** は定数、型、関数、クラスなどをまとめて管理する単位であり、コードの再利用性と整理に不可欠。
2. **スコープ解決演算子（`::`）** は出所を明確にし、名前衝突を防ぐ。大規模プロジェクトでの推奨方法。
3. **`import` 文**はワイルドカード（`::*`）と個別指定の 2 種類がある。便利だが名前衝突に注意。
4. **`$unit`**（コンパイル単位スコープ）はトップレベル宣言のスコープだが、移植性と保守性の問題からパッケージの使用が推奨される。
5. **コンパイル順序**ではパッケージを先にコンパイルする必要がある。循環依存は避けること。
6. **設計パターン**として、プロジェクト共通パッケージ、インターフェース固有パッケージ、テストベンチ用パッケージなどがある。

次章からは第 4 部「検証編」に入り、 SystemVerilog のオブジェクト指向プログラミング機能について学ぶ。
