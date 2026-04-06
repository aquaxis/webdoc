---
title: "第14章：DPI-C（Direct Programming Interface）"
---

# 第14章：DPI-C（Direct Programming Interface）

## 14.1 この章で学ぶこと

本章では、SystemVerilogと外部プログラミング言語（主にC/C++）を連携させるための**DPI-C（Direct Programming Interface - C）**を中心に、高度なトピックを包括的に解説します。DPI-Cを使うと、SystemVerilogからC関数を呼び出したり、逆にCからSystemVerilogのタスク・関数を呼び出したりすることが可能になります。レガシーな**PLI（Programming Language Interface）**との比較、共有ライブラリを用いたビルドフロー、パラメタライズドクラスやファクトリパターンなどの高度な言語機能、そしてシミュレーション性能を最適化するためのテクニックについても学びます。

- **DPI-Cの基本**: `import` / `export` 宣言、データ型マッピング、`pure` / `context` 属性
- **オープン配列**: C側から動的にSV配列を操作する方法
- **レガシーPLIの概要**: TF/ACCルーチンとVPIへの移行パス
- **C/C++統合のビルドフロー**: 共有ライブラリの作成とシミュレータへのリンク
- **高度な言語機能**: パラメタライズドクラス、型パラメータ、ファクトリパターン
- **性能最適化**: コンパイル戦略とシミュレーション高速化の実践

---

## 14.2 DPI-Cの基本概念

### 14.2.1 DPI-Cとは

**DPI-C（Direct Programming Interface - C）**は、IEEE 1800規格で定義されたSystemVerilogとC言語間の標準インターフェースです。従来のPLI/VPIと比較して、はるかにシンプルかつ高性能な連携を実現します。

- **シンプルな宣言**: `import "DPI-C"` / `export "DPI-C"` で関数を宣言するだけで連携可能です
- **直接呼び出し**: PLIのようなラッパー層を介さず、C関数を直接呼び出します
- **高い性能**: 関数呼び出しのオーバーヘッドが最小限に抑えられています
- **双方向通信**: SVからCを呼ぶ（import）だけでなく、CからSVを呼ぶ（export）ことも可能です

![DPI-Cのアーキテクチャ](/images/systemverilog-complete-guide/ch14_dpi_architecture.drawio.png)

### 14.2.2 import宣言（CをSVから呼び出す）

`import "DPI-C"` 宣言を使うと、C言語で実装された関数をSystemVerilogから呼び出すことができます。

```systemverilog
// C関数をSystemVerilogにインポート
import "DPI-C" function int c_add(input int a, input int b);
import "DPI-C" function string get_env_var(input string name);
import "DPI-C" function void c_print_message(input string msg);

module dpi_import_example;
    initial begin
        // C関数を通常のSV関数のように呼び出す
        $display("c_add(10, 20) = %0d", c_add(10, 20));       // 30
        $display("HOME = %s", get_env_var("HOME"));
        c_print_message("Hello from SystemVerilog!");
    end
endmodule
```

対応するC側の実装は以下のとおりです。

```c
#include <stdio.h>
#include <stdlib.h>
#include "svdpi.h"  // DPI-C用ヘッダ

int c_add(int a, int b) { return a + b; }

const char* get_env_var(const char* name) { return getenv(name); }

void c_print_message(const char* msg) { printf("[C] %s\n", msg); }
```

### 14.2.3 export宣言（SVをCから呼び出す）

`export "DPI-C"` 宣言を使うと、SystemVerilogで定義した関数やタスクをC言語から呼び出すことができます。

```systemverilog
module dpi_export_example;
    export "DPI-C" function sv_factorial;
    export "DPI-C" task sv_wait_clocks;

    function int sv_factorial(int n);
        if (n <= 1) return 1;
        else return n * sv_factorial(n - 1);
    endfunction

    task sv_wait_clocks(int num_clocks);
        repeat (num_clocks) @(posedge clk);
    endtask

    // context属性が必要（内部でexport関数を呼ぶため）
    import "DPI-C" context task c_run_test();

    reg clk = 0;
    always #5 clk = ~clk;

    initial begin
        c_run_test();
        $finish;
    end
endmodule
```

```c
#include <stdio.h>
#include "svdpi.h"

extern int sv_factorial(int n);       // SVからエクスポートされた関数
extern void sv_wait_clocks(int n);    // SVからエクスポートされたタスク

void c_run_test() {
    printf("[C] sv_factorial(5) = %d\n", sv_factorial(5));  // 120
    sv_wait_clocks(10);  // SVタスクを呼び出して時間を消費
}
```

### 14.2.4 import関数とimportタスクの違い

| 特性 | import function | import task |
|------|----------------|-------------|
| 時間消費 | 不可（ゼロ時間で実行） | 可能（シミュレーション時間を消費できる） |
| C側でのexport呼び出し | exportされたfunctionのみ | exportされたtask/function両方 |
| `pure` 属性 | 付与可能 | 付与不可 |
| `context` 属性 | 必要に応じて付与 | 常にcontext扱い |

---

## 14.3 データ型マッピング

### 14.3.1 基本データ型の対応関係

SystemVerilogとCの間でデータを受け渡すには、型の対応関係を正確に理解する必要があります。

| SystemVerilog型 | C型 | ビット幅 | 備考 |
|----------------|-----|---------|------|
| `byte` | `char` | 8 | 符号付き |
| `shortint` | `short int` | 16 | 符号付き |
| `int` | `int` | 32 | 符号付き |
| `longint` | `long long` | 64 | 符号付き |
| `real` | `double` | 64 | IEEE 754倍精度 |
| `shortreal` | `float` | 32 | IEEE 754単精度 |
| `string` | `const char*` | - | ヌル終端文字列 |
| `chandle` | `void*` | - | 不透明ポインタ |
| `bit` | `svBit` | 1 | 2値（0, 1） |
| `logic` | `svLogic` | 1 | 4値（0, 1, X, Z） |

### 14.3.2 2値型と4値型

SystemVerilog特有の**2値型（two-state）**と**4値型（four-state）**の区別は、DPI-Cでのデータ受け渡しにおいて重要です。

```c
#include "svdpi.h"

// 2値型（bit）: svBitVecVal を使用（32ビット単位のunsigned int配列）
void process_bit_vector(const svBitVecVal* bv, int width) {
    for (int i = 0; i < (width + 31) / 32; i++)
        printf("word[%d] = 0x%08x\n", i, bv[i]);
}

// 4値型（logic）: svLogicVecVal を使用
// aval: 値ビット, bval: 制御ビット
// (a=0,b=0)->0, (a=1,b=0)->1, (a=0,b=1)->Z, (a=1,b=1)->X
void process_logic_vector(const svLogicVecVal* lv, int width) {
    for (int i = 0; i < (width + 31) / 32; i++)
        printf("word[%d]: aval=0x%08x, bval=0x%08x\n",
               i, lv[i].aval, lv[i].bval);
}
```

```systemverilog
import "DPI-C" function void process_bit_vector(
    input bit [127:0] bv, input int width
);
import "DPI-C" function void process_logic_vector(
    input logic [127:0] lv, input int width
);

module data_type_example;
    bit   [127:0] b_data = 128'hDEAD_BEEF_CAFE_BABE_1234_5678_9ABC_DEF0;
    logic [127:0] l_data = 128'hFFFF_XXXX_ZZZZ_0000_1111_2222_3333_4444;
    initial begin
        process_bit_vector(b_data, 128);
        process_logic_vector(l_data, 128);
    end
endmodule
```

### 14.3.3 構造体のマッピング

`packed`構造体はビットベクタとして、`unpacked`構造体はC構造体として渡すことができます。

```systemverilog
typedef struct packed {
    bit [7:0]  opcode;
    bit [15:0] address;
    bit [7:0]  data;
} packed_cmd_t;  // ビットベクタとして扱われる

typedef struct {
    int    id;
    real   timestamp;
    string name;
} unpacked_info_t;  // C構造体に対応

import "DPI-C" function void process_packed_cmd(input packed_cmd_t cmd);
import "DPI-C" function void process_unpacked_info(input unpacked_info_t info);
```

![DPI-Cデータ型マッピング](/images/systemverilog-complete-guide/ch14_dpi_type_mapping.drawio.png)

---

## 14.4 pure関数とcontext関数

### 14.4.1 pure属性

`pure` 属性は、関数が**副作用を持たない**ことをシミュレータに伝えるための修飾子です。`pure` 関数はグローバル変数やI/Oに触れず、同じ引数に対して常に同じ結果を返す必要があります。

```systemverilog
import "DPI-C" pure function int c_max(input int a, input int b);
import "DPI-C" pure function real c_sqrt(input real x);
```

```c
#include <math.h>
int c_max(int a, int b) { return (a > b) ? a : b; }
double c_sqrt(double x) { return sqrt(x); }
```

`pure` 関数を宣言すると、シミュレータは結果のキャッシュや不要な呼び出しの削除などの最適化を適用できます。

### 14.4.2 context属性

`context` 属性は、関数が**シミュレーションコンテキストにアクセスする**ことをシミュレータに伝えます。エクスポートされたSV関数/タスクの呼び出しやVPI関数の使用に必要です。

```systemverilog
export "DPI-C" function sv_get_time;
function longint sv_get_time();
    return $time;
endfunction

import "DPI-C" context function void c_log_with_time(input string msg);
```

```c
extern long long sv_get_time();  // SVからエクスポートされた関数

void c_log_with_time(const char* msg) {
    long long sim_time = sv_get_time();
    printf("[%lld] %s\n", sim_time, msg);
}
```

### 14.4.3 属性の比較

| 属性 | 副作用 | SVコンテキスト | 最適化 | 用途 |
|------|--------|--------------|--------|------|
| `pure` | なし | アクセス不可 | 最大限 | 純粋な計算関数 |
| デフォルト | 許容 | アクセス不可 | 一部可能 | I/O操作等を含む関数 |
| `context` | 許容 | アクセス可能 | 制限あり | SV export呼び出し、VPI利用 |

---

## 14.5 オープン配列

### 14.5.1 オープン配列とは

**オープン配列（Open Array）**は、配列のサイズを宣言時に固定せず、実行時に決定する仕組みです。異なるサイズの配列を同一のC関数で処理できます。

```systemverilog
import "DPI-C" function void c_sort_array(inout int arr []);
import "DPI-C" function int c_find_max(input int arr []);

module open_array_example;
    int data5 [5]  = '{30, 10, 50, 20, 40};
    int data10[10] = '{9, 3, 7, 1, 5, 8, 2, 6, 4, 0};
    initial begin
        c_sort_array(data5);   // 同じC関数を異なるサイズの配列に適用
        $display("Max of data10: %0d", c_find_max(data10));
    end
endmodule
```

### 14.5.2 C側でのオープン配列操作

```c
#include "svdpi.h"

void c_sort_array(svOpenArrayHandle arr) {
    int lo = svLow(arr, 1);
    int hi = svHigh(arr, 1);
    // バブルソート
    for (int i = lo; i <= hi; i++) {
        for (int j = i + 1; j <= hi; j++) {
            int *pi = (int*)svGetArrElemPtr1(arr, i);
            int *pj = (int*)svGetArrElemPtr1(arr, j);
            if (*pi > *pj) { int tmp = *pi; *pi = *pj; *pj = tmp; }
        }
    }
}

int c_find_max(const svOpenArrayHandle arr) {
    int lo = svLow(arr, 1), hi = svHigh(arr, 1);
    int max_val = *(int*)svGetArrElemPtr1(arr, lo);
    for (int i = lo + 1; i <= hi; i++) {
        int val = *(int*)svGetArrElemPtr1(arr, i);
        if (val > max_val) max_val = val;
    }
    return max_val;
}
```

### 14.5.3 オープン配列API一覧

| API関数 | 説明 |
|---------|------|
| `svSize(h, dim)` | 指定次元のサイズを返す |
| `svLow(h, dim)` / `svHigh(h, dim)` | 下限/上限インデックスを返す |
| `svLeft(h, dim)` / `svRight(h, dim)` | 左端/右端インデックスを返す |
| `svDimensions(h)` | 配列の次元数を返す |
| `svGetArrElemPtr1(h, i)` | 1次元配列の要素ポインタを返す |
| `svGetArrElemPtr2(h, i, j)` | 2次元配列の要素ポインタを返す |

---

## 14.6 PLI（Programming Language Interface）の概要

### 14.6.1 PLIの歴史的背景

**PLI（Programming Language Interface）**は、Verilog HDLの初期から存在するC言語インターフェースです。現在はDPI-CおよびVPIに置き換えられていますが、レガシーコードの保守のために基本を押さえておくことは重要です。

| 世代 | インターフェース | 規格 | 状態 |
|------|----------------|------|------|
| 第1世代 | PLI 1.0 TF/ACC | IEEE 1364-1995 | 非推奨 |
| 第2世代 | VPI（PLI 2.0） | IEEE 1364-2001 | 現行規格 |
| 現行 | DPI-C | IEEE 1800-2005以降 | 推奨 |

### 14.6.2 TF/ACCルーチンの概要

TFルーチンはシステムタスク引数へのアクセス、ACCルーチンは設計オブジェクトへのアクセスを提供していました。

```c
#include "veriuser.h"  // TFルーチン用ヘッダ（レガシー）

// TFルーチンの例（非推奨）
int my_display_calltf(int user_data, int reason) {
    int value = tf_getp(1);       // 第1引数を整数として取得
    io_printf("Value = %d\n", value);
    return 0;
}
```

```c
#include "acc_user.h"  // ACCルーチン用ヘッダ（レガシー）

// ACCルーチンの例（非推奨）
void read_signal_value() {
    handle sig;
    acc_initialize();
    sig = acc_handle_by_name("top.dut.data_out", NULL);
    io_printf("data_out = %s\n", acc_fetch_value(sig, "%b", NULL));
    acc_close();
}
```

### 14.6.3 PLIからVPI/DPI-Cへの移行

| PLI 1.0 操作 | VPI対応 | DPI-C対応 |
|-------------|---------|----------|
| `tf_getp()` / `tf_putp()` | `vpi_get_value()` / `vpi_put_value()` | 関数引数で直接渡す |
| `acc_handle_by_name()` | `vpi_handle_by_name()` | context関数内でVPIを使用 |
| `acc_fetch_value()` | `vpi_get_value()` | 関数の戻り値で受け取る |
| カスタムシステムタスク | `vpi_register_systf()` | import宣言で同等機能を実現 |

移行時の判断基準は以下のとおりです。

- **単純な関数呼び出し**: DPI-Cを使用（最もシンプル）
- **シミュレーション内部オブジェクトへのアクセス**: VPIを使用（次章で詳述）
- **カスタムシステムタスクの登録**: VPIを使用

---

## 14.7 C/C++統合のビルドフロー

### 14.7.1 共有ライブラリの作成

DPI-Cで使用するC/C++コードは、共有ライブラリ（`.so` / `.dll`）としてコンパイルし、シミュレータにリンクします。

![DPI-Cビルドフロー](/images/systemverilog-complete-guide/ch14_dpi_build_flow.drawio.png)

```c
// ファイル: dpi_functions.c
#include <stdio.h>
#include <math.h>
#include "svdpi.h"

double c_sin(double x)  { return sin(x); }
double c_cos(double x)  { return cos(x); }

int c_popcount(const svBitVecVal* bv) {
    unsigned int v = bv[0];
    int count = 0;
    while (v) { count += v & 1; v >>= 1; }
    return count;
}
```

```bash
# ステップ1: 共有ライブラリにコンパイル
gcc -shared -fPIC -o dpi_functions.so dpi_functions.c -lm

# ステップ2: シミュレータごとのリンク方法
vcs -sverilog -LDFLAGS "-L. -ldpi_functions" top.sv     # VCS
vsim -sv_lib dpi_functions top                            # Questa/ModelSim
xrun -sv top.sv -sv_lib dpi_functions.so                  # Xcelium
```

### 14.7.2 C++との統合

C++コードを使用する場合は、`extern "C"` ブロックでC言語リンケージを指定する必要があります。

```c
// ファイル: dpi_cpp_functions.cpp
#include <string>
#include <vector>
#include <algorithm>
#include "svdpi.h"

extern "C" {

const char* c_to_upper(const char* str) {
    static std::string result;
    result = str;
    std::transform(result.begin(), result.end(), result.begin(), ::toupper);
    return result.c_str();
}

int c_median(const svOpenArrayHandle arr) {
    int size = svSize(arr, 1), lo = svLow(arr, 1);
    std::vector<int> vec;
    for (int i = lo; i < lo + size; i++)
        vec.push_back(*(int*)svGetArrElemPtr1(arr, i));
    std::sort(vec.begin(), vec.end());
    return vec[size / 2];
}

}  // extern "C"
```

### 14.7.3 実用的なディレクトリ構成

```
project/
├── rtl/                    # 設計ソース
│   └── design.sv
├── tb/                     # テストベンチ
│   ├── top_tb.sv
│   └── dpi_imports.svh     # DPI-Cインポート宣言をまとめたヘッダ
├── c_src/                  # C/C++ソース
│   ├── dpi_functions.c
│   └── Makefile
├── lib/                    # ビルド成果物
│   └── libdpi.so
└── sim/                    # シミュレーション実行ディレクトリ
    └── Makefile
```

DPI-Cインポート宣言は一箇所にまとめると管理が容易になります。

```systemverilog
// ファイル: tb/dpi_imports.svh
`ifndef DPI_IMPORTS_SVH
`define DPI_IMPORTS_SVH

import "DPI-C" pure function int    c_add(input int a, input int b);
import "DPI-C" pure function real   c_sqrt(input real x);
import "DPI-C" pure function int    c_popcount(input bit [31:0] bv);
import "DPI-C" context function void c_log_with_time(input string msg);

`endif
```

---

## 14.8 パラメタライズドクラスと高度な言語機能

### 14.8.1 パラメタライズドクラス

SystemVerilogでは、クラスにパラメータを持たせることで汎用的で再利用性の高い設計が可能です。

```systemverilog
class Fifo #(type T = int, int DEPTH = 16);
    T queue[$];

    function bit push(T item);
        if (queue.size() >= DEPTH) return 0;
        queue.push_back(item);
        return 1;
    endfunction

    function bit pop(output T item);
        if (queue.size() == 0) return 0;
        item = queue.pop_front();
        return 1;
    endfunction

    function int size();       return queue.size();        endfunction
    function bit is_empty();   return (queue.size() == 0); endfunction
    function bit is_full();    return (queue.size() >= DEPTH); endfunction
endclass
```

異なる型やサイズでインスタンス化する例です。

```systemverilog
module parameterized_class_example;
    Fifo #(int, 8)          int_fifo;      // int型、深さ8
    Fifo #(real, 32)         real_fifo;     // real型、深さ32
    Fifo #(bit [63:0], 256)  wide_fifo;    // 64ビット型、深さ256

    typedef struct {
        bit [31:0] addr;
        bit [31:0] data;
        bit        write;
    } bus_txn_t;
    Fifo #(bus_txn_t, 64) txn_fifo;        // 構造体型、深さ64

    initial begin
        int val;
        int_fifo = new();
        int_fifo.push(42);
        int_fifo.push(100);
        int_fifo.pop(val);
        $display("Popped: %0d", val);  // 42
    end
endmodule
```

### 14.8.2 型パラメータの活用

型パラメータを使うと、コンテナクラスやユーティリティクラスを型に依存しない形で実装できます。

```systemverilog
class Pair #(type T1 = int, type T2 = int);
    T1 first;
    T2 second;
    function new(T1 f, T2 s);
        first = f; second = s;
    endfunction
    function void display();
        $display("Pair: (%p, %p)", first, second);
    endfunction
endclass

module type_param_example;
    initial begin
        Pair #(int, string) p1 = new(42, "hello");
        Pair #(real, bit [7:0]) p2 = new(3.14, 8'hFF);
        p1.display();
        p2.display();
    end
endmodule
```

### 14.8.3 ファクトリパターン

**ファクトリパターン**は、オブジェクト生成を柔軟に制御するデザインパターンです。テストケースごとに異なるトランザクションを動的に生成する際に活用されます。

```systemverilog
class Transaction;
    rand bit [31:0] addr;
    rand bit [31:0] data;
    rand bit        write;

    virtual function void display(string prefix = "");
        $display("%sTXN: addr=0x%08h, data=0x%08h, wr=%0b",
                 prefix, addr, data, write);
    endfunction

    virtual function Transaction clone();
        Transaction t = new();
        t.addr = this.addr; t.data = this.data; t.write = this.write;
        return t;
    endfunction
endclass

class BurstTransaction extends Transaction;
    rand int unsigned burst_len;
    constraint c_burst { burst_len inside {[1:16]}; }

    virtual function void display(string prefix = "");
        super.display(prefix);
        $display("%s  burst_len=%0d", prefix, burst_len);
    endfunction
endclass

class TransactionFactory;
    static Transaction prototypes[string];

    static function void register_type(string name, Transaction proto);
        prototypes[name] = proto;
    endfunction

    static function Transaction create(string name);
        if (!prototypes.exists(name))
            $fatal(1, "Unknown type: %s", name);
        return prototypes[name].clone();
    endfunction
endclass

module factory_example;
    initial begin
        Transaction t = new();
        BurstTransaction b = new();
        TransactionFactory::register_type("basic", t);
        TransactionFactory::register_type("burst", b);

        t = TransactionFactory::create("burst");
        t.randomize();
        t.display("[BURST] ");
    end
endmodule
```

![ファクトリパターンのクラス図](/images/systemverilog-complete-guide/ch14_factory_pattern.drawio.png)

---

## 14.9 性能最適化

### 14.9.1 コンパイル戦略

シミュレーションの性能はコンパイル方法によって大きく変わります。

| 戦略 | 説明 | メリット | デメリット |
|------|------|---------|----------|
| **インクリメンタル** | 変更ファイルのみ再コンパイル | ビルド時間短縮 | 初回は全体と同等 |
| **2パス** | 解析と最適化を分離 | 深い最適化 | コンパイル時間長い |
| **並列** | 複数ファイルを並列コンパイル | マルチコア活用 | ライセンス制約あり |

```bash
# VCSでの最適化コンパイル例
vcs -sverilog -O3 -j8 -partcomp -full64 -f filelist.f -o simv

# Xceliumでの最適化コンパイル例
xrun -sv -access +r -64bit -elaborate -f filelist.f
```

### 14.9.2 シミュレーション高速化のテクニック

**1. 2値型の活用**: テストベンチでX/Z検出が不要な場合は `bit` を使用します。

```systemverilog
// 性能向上: 4値型（logic）ではなく2値型（bit）を使用
class FastTransaction;
    bit [31:0] addr;    // logic ではなく bit
    bit [31:0] data;
    bit        valid;
endclass
```

**2. DPI-Cによる計算オフロード**: 計算量の多い処理をC関数に移すことで大幅な高速化が見込めます。

```systemverilog
// C側のテーブル駆動CRCは SV側のループ実装より高速
import "DPI-C" pure function int c_crc32_fast(
    input byte unsigned data[], input int len
);
```

```c
int c_crc32_fast(const svOpenArrayHandle data, int len) {
    static unsigned int table[256];
    static int init = 0;
    if (!init) {
        for (unsigned int i = 0; i < 256; i++) {
            unsigned int c = i;
            for (int j = 0; j < 8; j++)
                c = (c >> 1) ^ ((c & 1) ? 0xEDB88320 : 0);
            table[i] = c;
        }
        init = 1;
    }
    unsigned int crc = 0xFFFFFFFF;
    for (int i = 0; i < len; i++) {
        unsigned char* p = (unsigned char*)svGetArrElemPtr1(data, i);
        crc = table[(crc ^ *p) & 0xFF] ^ (crc >> 8);
    }
    return ~crc;
}
```

**3. 不要なアクティビティの削減**:

```systemverilog
`ifdef DEBUG
    `define DBG_DISPLAY(msg) $display msg
`else
    `define DBG_DISPLAY(msg)
`endif
```

### 14.9.3 DPI-Cの性能に関する注意点

- **`pure` 関数を積極的に使う**: シミュレータによる最適化が可能になります
- **`context` は必要な場合のみ使う**: コンテキスト保存・復元のオーバーヘッドが発生します
- **バッチ処理を心がける**: データは配列としてまとめて渡します
- **C側のメモリ管理に注意**: `malloc`/`free` の呼び出し頻度を最小限に抑えます

---

## 14.10 実践例：DPI-Cを活用したリファレンスモデル

### 14.10.1 Cリファレンスモデルとの連携

RTLの出力をCで記述したリファレンスモデルと比較する手法は、ハードウェア検証の基本です。

```systemverilog
import "DPI-C" function void c_encrypt_init(input bit [127:0] key);
import "DPI-C" function void c_encrypt_block(
    input  bit [127:0] plaintext,
    output bit [127:0] ciphertext
);

module encrypt_checker (
    input  logic        clk,
    input  logic        rst_n,
    input  logic        valid_in,
    input  logic [127:0] key,
    input  logic [127:0] plaintext,
    input  logic        valid_out,
    input  logic [127:0] ciphertext
);
    bit [127:0] expected;
    bit         key_loaded = 0;

    always_ff @(posedge clk) begin
        if (!rst_n)
            key_loaded <= 0;
        else if (valid_in && !key_loaded) begin
            c_encrypt_init(key);       // リファレンスモデルの初期化
            key_loaded <= 1;
        end
    end

    always_ff @(posedge clk) begin
        if (valid_out) begin
            c_encrypt_block(plaintext, expected);
            assert (ciphertext == expected)
                else $error("[%0t] Mismatch! DUT=%h, Ref=%h",
                            $time, ciphertext, expected);
        end
    end
endmodule
```

```c
#include <string.h>
#include "svdpi.h"

static unsigned char ref_key[16];

void c_encrypt_init(const svBitVecVal* key) {
    memcpy(ref_key, key, 16);
}

void c_encrypt_block(const svBitVecVal* plaintext, svBitVecVal* ciphertext) {
    unsigned char pt[16], ct[16];
    memcpy(pt, plaintext, 16);
    // 簡易暗号化（実際のプロジェクトではAES等を使用）
    for (int round = 0; round < 10; round++) {
        for (int i = 0; i < 16; i++)
            ct[i] = pt[i] ^ ref_key[i] ^ (unsigned char)round;
        memcpy(pt, ct, 16);
    }
    memcpy(ciphertext, ct, 16);
}
```

### 14.10.2 バイナリファイルI/O

SystemVerilog標準のファイルI/Oでは対応しきれない操作を、DPI-Cの `chandle` を使って実現します。

```systemverilog
import "DPI-C" function chandle c_fopen(input string filename, input string mode);
import "DPI-C" function void    c_fclose(input chandle fp);
import "DPI-C" function int     c_fread_bytes(
    input chandle fp, inout byte unsigned buf[], input int count
);
import "DPI-C" function int     c_fwrite_bytes(
    input chandle fp, input byte unsigned buf[], input int count
);

module file_io_example;
    chandle fp;
    byte unsigned wbuf[256], rbuf[256];

    initial begin
        foreach (wbuf[i]) wbuf[i] = i;      // テストデータ生成
        fp = c_fopen("data.bin", "wb");
        c_fwrite_bytes(fp, wbuf, 256);
        c_fclose(fp);

        fp = c_fopen("data.bin", "rb");
        c_fread_bytes(fp, rbuf, 256);
        c_fclose(fp);

        foreach (rbuf[i])
            assert (rbuf[i] == i) else $error("Mismatch at %0d", i);
        $display("File I/O test passed!");
    end
endmodule
```

---

## 14.11 DPI-Cのデバッグとトラブルシューティング

### 14.11.1 よくある問題と対処法

| 問題 | 原因 | 対処法 |
|------|------|--------|
| リンクエラー（undefined symbol） | 関数名の不一致、`extern "C"` の欠落 | 関数名とリンケージの確認 |
| セグメンテーションフォルト | NULLポインタ、配列境界外アクセス | GDBでC側をデバッグ |
| 値の不一致 | データ型マッピングの誤り | `svBitVecVal`/`svLogicVecVal` の使い分けを確認 |
| メモリリーク | C側のメモリ解放漏れ | Valgrindでメモリ分析 |

### 14.11.2 デバッグ手法

```c
// デバッグ用ログマクロ
#ifdef DPI_DEBUG
    #define DPI_LOG(fmt, ...) fprintf(stderr, "[DPI] " fmt "\n", ##__VA_ARGS__)
#else
    #define DPI_LOG(fmt, ...)
#endif

int c_process_data(const svBitVecVal* data, int width) {
    DPI_LOG("called: width=%d, data[0]=0x%08x", width, data[0]);
    int result = 0;
    for (int i = 0; i < (width + 31) / 32; i++) result ^= data[i];
    return result;
}
```

### 14.11.3 関数名のマングリング制御

C側の関数名をSystemVerilog側と異なる名前にマッピングすることも可能です。

```systemverilog
// C側では c_internal_add が呼ばれるが、SV側では my_add として使用
import "DPI-C" c_internal_add = function int my_add(input int a, input int b);
```

この機能は、既存のCライブラリの関数名がSystemVerilogの予約語と衝突する場合などに有用です。

---

## 14.12 まとめ

本章では、SystemVerilogの高度なトピックとして、DPI-Cを中心にC/C++との連携手法と高度な言語機能について学びました。

1. **DPI-C**は `import "DPI-C"` / `export "DPI-C"` により、SystemVerilogとC言語間の双方向関数呼び出しを実現する標準インターフェースです。
2. **データ型マッピング**では、`int`・`real`・`string` などの基本型に加え、`svBitVecVal`（2値）・`svLogicVecVal`（4値）によるビットベクタの受け渡しが可能です。
3. **`pure` 属性**は副作用のない関数に付与し最適化を可能にします。**`context` 属性**はSVエクスポート関数やVPIを呼び出す関数に必要です。
4. **オープン配列**を使うことで、サイズの異なる配列を同一のC関数で処理できます。
5. **PLI（TF/ACC）**はレガシーインターフェースであり、新規開発ではDPI-CまたはVPIを使用すべきです。
6. **共有ライブラリ**としてC/C++コードをコンパイルし、シミュレータにリンクするビルドフローの理解が実践では不可欠です。
7. **パラメタライズドクラス**と**ファクトリパターン**により、汎用的で柔軟なクラス設計が実現できます。
8. **性能最適化**では、2値型の活用、DPI-Cによる計算オフロード、不要なアクティビティの削減が有効です。

次章では、VPI（Verilog Procedural Interface）を使ったシミュレーション内部オブジェクトへのアクセスと、実践的なデバッグ技法について学びます。
