---
title: "第3章：基本データ型とリテラル"
---

# 第3章：基本データ型とリテラル

## 3.1 この章で学ぶこと

SystemVerilogを使いこなすうえで、データ型の理解は避けて通れない最重要テーマです。本章では、SystemVerilogが提供する豊富なデータ型の体系を基礎から丁寧に解説します。

具体的には、以下の内容を扱います。

- **4値論理と2値論理**の違いと使い分け
- **符号付き・符号なし整数型**の種類と特性
- **ユーザー定義型**（typedef、enum、struct、union）の活用方法
- **定数**（parameter、localparam、const、specparam）の宣言と用途
- **リテラル表現**（数値、文字列、時間）の記法

これらを正しく理解することで、ハードウェア設計と検証の両方で、安全かつ効率的なコードを書けるようになります。

![SystemVerilogデータ型の分類体系](/images/systemverilog-complete-guide/ch03_type_hierarchy.drawio.png)

---

## 3.2 4値論理と2値論理

### 3.2.1 4値論理とは

ハードウェアの世界では、信号の状態は単純な0と1だけでは表現しきれません。SystemVerilogの4値論理では、以下の4つの状態を扱います。

| 値 | 意味 | 用途 |
|----|------|------|
| `0` | 論理Low | 確定した低電位状態 |
| `1` | 論理High | 確定した高電位状態 |
| `x`（または`X`） | 不定値（Unknown） | 初期化されていない、または衝突している状態 |
| `z`（または`Z`） | ハイインピーダンス | 駆動されていない（トライステート）状態 |

`x`はシミュレーション開始直後のレジスタ値や、複数のドライバが同一信号を異なる値で駆動しているときに現れます。`z`はバスのトライステート制御などで意図的に使用されます。

### 3.2.2 logic型

`logic`型はSystemVerilogで最も基本的かつ重要なデータ型です。4値論理（0, 1, x, z）を扱い、Verilog時代の`reg`型と`wire`型の多くの場面を置き換えることができます。

```systemverilog
// logic型の基本宣言
logic        a;          // 1ビットの信号
logic [7:0]  data;       // 8ビットの信号
logic [31:0] address;    // 32ビットの信号

// logic型への代入
initial begin
    a = 1'b0;
    data = 8'hFF;
    address = 32'hDEAD_BEEF;
end
```

### 3.2.3 reg型からlogic型への移行理由

Verilog時代には、`reg`型と`wire`型を状況に応じて使い分ける必要がありました。`reg`は`always`ブロック内での代入先として、`wire`は`assign`文や接続先として使用されました。しかし、`reg`という名前が「レジスタ（フリップフロップ）」を連想させるにもかかわらず、実際には組み合わせ回路でも使用されるため、初学者にとって混乱の原因となっていました。

SystemVerilogの`logic`型は、この問題を解決します。`logic`型はドライバが1つだけの場合、`reg`と`wire`のどちらの文脈でも使用できます。

```systemverilog
// Verilog時代の記述（冗長で混乱しやすい）
module old_style(
    input  wire        clk,
    input  wire        rst_n,
    input  wire  [7:0] data_in,
    output reg   [7:0] data_out
);
    reg [7:0] temp;  // always内で使うのでreg
    wire [7:0] sum;  // assign文で使うのでwire

    assign sum = data_in + temp;

    always @(posedge clk or negedge rst_n) begin
        if (!rst_n)
            data_out <= 8'h00;
        else
            data_out <= sum;
    end
endmodule

// SystemVerilogでの記述（logicに統一）
module new_style(
    input  logic       clk,
    input  logic       rst_n,
    input  logic [7:0] data_in,
    output logic [7:0] data_out
);
    logic [7:0] temp;
    logic [7:0] sum;

    assign sum = data_in + temp;

    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n)
            data_out <= 8'h00;
        else
            data_out <= sum;
    end
endmodule
```

### 3.2.4 wire型との関係

`logic`型は万能ではありません。**複数のドライバ**が存在する場合には、依然として`wire`型を使用する必要があります。これは、トライステートバスや双方向信号など、複数のソースが同一の信号を駆動するケースに該当します。

```systemverilog
// 複数ドライバが必要な場合はwireを使用
wire [7:0] shared_bus;

// 複数のモジュールが同じバスを駆動
assign shared_bus = (enable_a) ? data_a : 8'bz;
assign shared_bus = (enable_b) ? data_b : 8'bz;
```

### 3.2.5 bit型（2値論理）

`bit`型は2値論理（0と1のみ）を扱うデータ型です。`x`や`z`の状態を持たないため、シミュレーション速度が向上する場合があります。主に検証環境（テストベンチ）で使用されます。

```systemverilog
// bit型の宣言
bit        flag;        // 1ビット（初期値は0）
bit [7:0]  counter;     // 8ビットカウンタ
bit [31:0] test_data;   // 32ビットテストデータ
```

> **注意**: `bit`型は初期値が`0`であるのに対し、`logic`型の初期値は`x`（不定）です。この違いにより、`bit`型をRTL設計に使用すると、初期化忘れのバグを検出できなくなるリスクがあります。RTL設計には`logic`型を使用し、`bit`型は検証環境に限定することを推奨します。

### 3.2.6 4値型と2値型の比較

| 特性 | 4値型（logic） | 2値型（bit） |
|------|----------------|--------------|
| 取りうる値 | 0, 1, x, z | 0, 1 |
| 初期値 | `x`（不定） | `0` |
| 主な用途 | RTL設計 | テストベンチ |
| x/z検出 | 可能 | 不可能 |
| シミュレーション速度 | 標準 | 高速（場合による） |

![4値論理と2値論理の信号状態遷移](/images/systemverilog-complete-guide/ch03_4state_vs_2state.drawio.png)

---

## 3.3 符号付き・符号なし整数型

### 3.3.1 2値整数型

SystemVerilogは、C言語に似た2値整数型を提供します。これらはシミュレーション専用で、合成対象とはならないことが多いです。

| 型名 | ビット幅 | 符号 | 値の範囲 |
|------|----------|------|----------|
| `byte` | 8ビット | 符号付き | -128 ~ 127 |
| `shortint` | 16ビット | 符号付き | -32,768 ~ 32,767 |
| `int` | 32ビット | 符号付き | -2,147,483,648 ~ 2,147,483,647 |
| `longint` | 64ビット | 符号付き | -2^63 ~ 2^63-1 |

```systemverilog
// 2値整数型の使用例
byte     char_data;      // 8ビット符号付き
shortint sensor_val;     // 16ビット符号付き
int      loop_count;     // 32ビット符号付き
longint  timestamp;      // 64ビット符号付き

initial begin
    char_data  = 8'd100;
    sensor_val = -16'd500;
    loop_count = 1000;
    timestamp  = 64'd123456789012;

    $display("byte: %0d, shortint: %0d", char_data, sensor_val);
    $display("int: %0d, longint: %0d", loop_count, timestamp);
end
```

### 3.3.2 4値整数型

4値論理に基づく整数型も存在します。代表的なのは`integer`型です。

| 型名 | ビット幅 | 符号 | 論理 | 備考 |
|------|----------|------|------|------|
| `integer` | 32ビット | 符号付き | 4値 | Verilogから引き継がれた型 |
| `time` | 64ビット | 符号なし | 4値 | シミュレーション時間の格納用 |

```systemverilog
// integer型（4値）とint型（2値）の違い
integer idx_4state;   // 4値、初期値はx
int     idx_2state;   // 2値、初期値は0

initial begin
    // 初期値の違いを確認
    $display("integer初期値: %h", idx_4state);  // xxxxxxxx
    $display("int初期値: %h", idx_2state);       // 00000000
end
```

### 3.3.3 整数型一覧表

以下に、SystemVerilogの全整数型を一覧にまとめます。

| 型名 | ビット幅 | 符号 | 論理 | 初期値 | 主な用途 |
|------|----------|------|------|--------|----------|
| `bit` | 1（可変） | 符号なし | 2値 | 0 | テストベンチ |
| `logic` | 1（可変） | 符号なし | 4値 | x | RTL設計 |
| `reg` | 1（可変） | 符号なし | 4値 | x | 旧Verilog互換 |
| `byte` | 8 | 符号付き | 2値 | 0 | 文字・小データ |
| `shortint` | 16 | 符号付き | 2値 | 0 | 中規模データ |
| `int` | 32 | 符号付き | 2値 | 0 | ループカウンタ等 |
| `longint` | 64 | 符号付き | 2値 | 0 | タイムスタンプ等 |
| `integer` | 32 | 符号付き | 4値 | x | Verilog互換 |
| `time` | 64 | 符号なし | 4値 | x | シミュレーション時間 |

### 3.3.4 signed/unsigned修飾子

デフォルトの符号属性を変更したい場合は、`signed`または`unsigned`修飾子を使用します。

```systemverilog
// signed/unsigned修飾子の使用例
logic signed [7:0]   signed_data;    // 符号付き8ビット（-128 ~ 127）
logic unsigned [7:0] unsigned_data;  // 符号なし8ビット（0 ~ 255）
bit signed [15:0]    signed_val;     // 符号付き16ビット
int unsigned         uint_val;       // 符号なし32ビット

initial begin
    signed_data   = 8'hFF;  // -1 として解釈
    unsigned_data = 8'hFF;  // 255 として解釈

    $display("signed:   %0d", signed_data);    // -1
    $display("unsigned: %0d", unsigned_data);   // 255
end
```

> **重要**: 符号付きと符号なしの型を混在させた演算は、意図しない結果を招くことがあります。SystemVerilogでは、演算に関わるオペランドのうち1つでも符号なしであれば、全体が符号なし演算として扱われます。

---

## 3.4 ユーザー定義型

### 3.4.1 typedef（型の別名定義）

`typedef`を使用すると、既存の型に分かりやすい名前を付けることができます。コードの可読性が向上し、型の変更も一箇所で済むようになります。

```systemverilog
// typedefの基本的な使い方
typedef logic [7:0]  byte_t;
typedef logic [15:0] halfword_t;
typedef logic [31:0] word_t;
typedef logic [63:0] dword_t;

module alu(
    input  word_t  operand_a,
    input  word_t  operand_b,
    output word_t  result
);
    assign result = operand_a + operand_b;
endmodule
```

`typedef`による型名には、慣例として`_t`サフィックスを付けることが多いです。これにより、ユーザー定義型であることが一目でわかります。

### 3.4.2 enum（列挙型）

`enum`は、名前付き定数の集合を定義する型です。ステートマシンの状態定義に特に有効であり、数値をそのまま使用するよりもコードの意図が明確になります。

```systemverilog
// 基本的なenum宣言
typedef enum logic [2:0] {
    IDLE   = 3'b000,
    FETCH  = 3'b001,
    DECODE = 3'b010,
    EXEC   = 3'b011,
    WRITE  = 3'b100
} cpu_state_t;

// ステートマシンでの活用例
module fsm_example(
    input  logic clk,
    input  logic rst_n,
    input  logic start,
    output logic busy
);
    cpu_state_t current_state, next_state;

    // 状態レジスタ
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n)
            current_state <= IDLE;
        else
            current_state <= next_state;
    end

    // 次状態論理
    always_comb begin
        next_state = current_state;  // デフォルト：状態維持
        case (current_state)
            IDLE:   if (start) next_state = FETCH;
            FETCH:  next_state = DECODE;
            DECODE: next_state = EXEC;
            EXEC:   next_state = WRITE;
            WRITE:  next_state = IDLE;
            default: next_state = IDLE;
        endcase
    end

    // 出力論理
    assign busy = (current_state != IDLE);
endmodule
```

`enum`にはいくつかの便利なメソッドが用意されています。

```systemverilog
cpu_state_t state;

initial begin
    state = IDLE;
    $display("名前: %s", state.name());    // "IDLE"
    state = state.next();                   // FETCH
    state = state.prev();                   // IDLE
    $display("最初: %s", state.first().name()); // "IDLE"
    $display("最後: %s", state.last().name());  // "WRITE"
end
```

### 3.4.3 struct（構造体）

`struct`は、複数のフィールドをひとまとめにしたデータ型を定義します。関連するデータをグループ化することで、コードの整理と可読性の向上に役立ちます。

#### packed struct

`packed`修飾子を付けた構造体は、フィールドがビット単位で連続的に配置されます。ビットベクタとして扱えるため、RTL設計で特に有用です。

```systemverilog
// packed structの定義
typedef struct packed {
    logic [15:0] addr;    // ビット[31:16]
    logic [7:0]  data;    // ビット[15:8]
    logic [3:0]  cmd;     // ビット[7:4]
    logic [3:0]  status;  // ビット[3:0]
} packet_t;

module packet_handler(
    input  logic    clk,
    input  packet_t pkt_in,
    output packet_t pkt_out
);
    always_ff @(posedge clk) begin
        pkt_out <= pkt_in;
    end

    // packed structはビットベクタとしてもアクセス可能
    // pkt_in 全体は32ビットのベクタとして扱える
    logic [31:0] raw_bits;
    assign raw_bits = pkt_in;  // 自動変換
endmodule
```

#### unpacked struct

`packed`修飾子を付けない構造体は、フィールドのメモリ配置がシミュレータに依存します。テストベンチやモデリングで使用します。

```systemverilog
// unpacked structの定義
typedef struct {
    string  name;
    int     id;
    bit     active;
    real    score;
} student_t;

initial begin
    student_t s;
    s.name   = "Tanaka";
    s.id     = 1001;
    s.active = 1;
    s.score  = 95.5;

    $display("学生: %s (ID: %0d) スコア: %0.1f", s.name, s.id, s.score);
end
```

### 3.4.4 union（共用体）

`union`は、同じメモリ領域を複数の異なる型で共有します。ハードウェア設計では、同一のビットパターンを異なる視点で解釈する場面で有用です。

```systemverilog
// 基本的なunion
typedef union packed {
    logic [31:0] raw;
    struct packed {
        logic [7:0]  opcode;
        logic [4:0]  rs1;
        logic [4:0]  rs2;
        logic [4:0]  rd;
        logic [8:0]  reserved;
    } fields;
} instruction_t;

initial begin
    instruction_t instr;
    instr.raw = 32'hAB_CDE_F01;

    $display("Opcode: %h", instr.fields.opcode);
    $display("RS1:    %h", instr.fields.rs1);
end
```

#### tagged union

通常の`union`では、どのフィールドが有効かを追跡するのはプログラマの責任です。`tagged union`は、型安全なアクセスを提供し、不正なフィールドへのアクセスを防止します。

```systemverilog
// tagged unionの定義
typedef union tagged {
    void    Invalid;
    int     Value;
    string  Error;
} result_t;

initial begin
    result_t r;

    // tagged unionの使用
    r = tagged Value 42;

    // パターンマッチで安全にアクセス
    case (r) matches
        tagged Invalid:    $display("無効");
        tagged Value .v:   $display("値: %0d", v);
        tagged Error .msg: $display("エラー: %s", msg);
    endcase
end
```

> **注意**: `tagged union`のサポート状況はシミュレータによって異なります。使用前にツールの対応状況を確認してください。

![構造体と共用体のメモリレイアウト比較](/images/systemverilog-complete-guide/ch03_struct_union_layout.drawio.png)

---

## 3.5 定数

SystemVerilogには、用途に応じた複数の定数定義手段があります。

### 3.5.1 parameter

`parameter`は、モジュールをパラメータ化するために使用します。インスタンス化時に値を上書きできるため、汎用的なモジュール設計に不可欠です。

```systemverilog
// parameterによるモジュールのパラメータ化
module fifo #(
    parameter int DATA_WIDTH = 8,
    parameter int DEPTH      = 16
)(
    input  logic                  clk,
    input  logic                  rst_n,
    input  logic [DATA_WIDTH-1:0] wr_data,
    output logic [DATA_WIDTH-1:0] rd_data,
    output logic                  full,
    output logic                  empty
);
    localparam int ADDR_WIDTH = $clog2(DEPTH);

    logic [DATA_WIDTH-1:0] mem [0:DEPTH-1];
    logic [ADDR_WIDTH:0]   wr_ptr, rd_ptr;

    // ... FIFO実装 ...
endmodule

// インスタンス化時にパラメータを変更
module top;
    fifo #(.DATA_WIDTH(32), .DEPTH(64)) u_fifo_32x64 (
        .clk(clk), .rst_n(rst_n),
        .wr_data(data_32), .rd_data(out_32),
        .full(full), .empty(empty)
    );

    fifo #(.DATA_WIDTH(16), .DEPTH(128)) u_fifo_16x128 (
        .clk(clk), .rst_n(rst_n),
        .wr_data(data_16), .rd_data(out_16),
        .full(full2), .empty(empty2)
    );
endmodule
```

### 3.5.2 localparam

`localparam`は、モジュール内部でのみ使用するローカル定数を定義します。`parameter`と異なり、インスタンス化時に外部から上書きすることはできません。他のパラメータから導出された値など、変更されるべきでない定数に適しています。

```systemverilog
module memory_controller #(
    parameter int ADDR_WIDTH = 20,
    parameter int DATA_WIDTH = 32
);
    // localparamはparameterから導出した定数に使う
    localparam int BYTE_LANES   = DATA_WIDTH / 8;
    localparam int MEM_SIZE     = 2 ** ADDR_WIDTH;
    localparam int BURST_LENGTH = 8;

    // 外部から変更できないことが保証される
    // MEM_SIZEやBURST_LENGTHはモジュール内でのみ有効
endmodule
```

### 3.5.3 const

`const`は、変数を定数として宣言します。`parameter`や`localparam`がエラボレーション時（コンパイル時）に値が確定するのに対し、`const`はシミュレーション実行時に一度だけ値が設定され、以降は変更不可となります。

```systemverilog
// constの使用例
class transaction;
    const int MAX_RETRY = 3;    // クラス内定数

    function new();
        // constフィールドはコンストラクタで初期化可能
    endfunction
endclass

// 関数内でのconst使用
function automatic void process_data();
    const int THRESHOLD = 100;
    // THRESHOLDは関数内で変更不可
endfunction
```

### 3.5.4 specparam

`specparam`は、タイミング仕様を記述するための特殊なパラメータです。`specify`ブロック内で使用し、遅延時間などのタイミング情報を定義します。

```systemverilog
module buf_cell(input logic a, output logic y);
    // specparamはspecifyブロック内で使用
    specify
        specparam tPLH = 1.2;  // 立ち上がり伝搬遅延 [ns]
        specparam tPHL = 0.8;  // 立ち下がり伝搬遅延 [ns]
        specparam tR   = 0.3;  // 立ち上がり遷移時間 [ns]
        specparam tF   = 0.2;  // 立ち下がり遷移時間 [ns]

        (a => y) = (tPLH, tPHL);
    endspecify

    assign y = a;
endmodule
```

### 3.5.5 定数の使い分けまとめ

| 定数の種類 | 確定タイミング | 上書き可否 | 主な用途 |
|-----------|---------------|-----------|---------|
| `parameter` | エラボレーション時 | インスタンス化時に可 | モジュールのパラメータ化 |
| `localparam` | エラボレーション時 | 不可 | 導出定数、内部定数 |
| `const` | シミュレーション実行時 | 不可（初期化後） | クラス定数、関数内定数 |
| `specparam` | エラボレーション時 | SDF注釈で可 | タイミング仕様 |

---

## 3.6 リテラル表現

### 3.6.1 数値リテラル

SystemVerilogの数値リテラルは、`<ビット幅>'<基数><値>` という形式で記述します。

| 基数指定 | 進数 | 使用可能な文字 | 例 |
|----------|------|---------------|-----|
| `'b` / `'B` | 2進 | 0, 1, x, z | `8'b1010_0101` |
| `'o` / `'O` | 8進 | 0-7, x, z | `8'o245` |
| `'d` / `'D` | 10進 | 0-9 | `8'd165` |
| `'h` / `'H` | 16進 | 0-9, a-f, A-F, x, z | `8'hA5` |

```systemverilog
// 数値リテラルの各種表現
logic [7:0] val;

initial begin
    // 同じ値（165）を異なる基数で表現
    val = 8'b1010_0101;   // 2進数（アンダースコアで区切り可能）
    val = 8'o245;         // 8進数
    val = 8'd165;         // 10進数
    val = 8'hA5;          // 16進数

    // ビット幅の省略（最低32ビットとして扱われる）
    val = 'hA5;           // 幅なし16進数

    // 符号付きリテラル
    val = 8'shFF;         // 符号付き（-1として解釈）

    // 特殊値リテラル
    val = 8'bxxxx_xxxx;  // 全ビット不定
    val = 8'bzzzz_zzzz;  // 全ビットハイインピーダンス
    val = '0;             // 全ビット0（ビット幅自動推論）
    val = '1;             // 全ビット1
    val = 'x;             // 全ビットx
    val = 'z;             // 全ビットz
end
```

> **補足**: アンダースコア（`_`）は数値リテラル内で自由に使用でき、可読性を向上させます。`32'hDEAD_BEEF`のように桁区切りとして活用するとよいでしょう。

### 3.6.2 文字列リテラル

文字列リテラルはダブルクォーテーションで囲みます。SystemVerilogでは`string`型が用意されており、可変長の文字列を扱えます。

```systemverilog
// 文字列リテラル
string greeting = "Hello, SystemVerilog!";
string file_path = "/data/test_vectors.txt";

// エスケープシーケンス
string with_newline = "Line1\nLine2";
string with_tab     = "Col1\tCol2";
string with_quote   = "He said \"Hello\"";
string with_bslash  = "path\\to\\file";

// 文字列メソッド
initial begin
    string s = "SystemVerilog";

    $display("長さ: %0d", s.len());        // 13
    $display("大文字: %s", s.toupper());    // SYSTEMVERILOG
    $display("小文字: %s", s.tolower());    // systemverilog
    $display("部分文字列: %s", s.substr(0, 5)); // System

    // 文字列の比較
    if (s == "SystemVerilog")
        $display("一致");

    // 文字列の結合
    string full = {s, " ", "2024"};  // "SystemVerilog 2024"
end
```

### 3.6.3 時間リテラル

時間リテラルは、シミュレーションにおける遅延時間を指定する際に使用します。数値の後に時間単位を付けて記述します。

```systemverilog
// 時間リテラルの使用例
module timing_example;
    // タイムスケールの宣言
    timeunit 1ns;
    timeprecision 1ps;

    logic clk;
    logic data;

    // 時間リテラルを使った遅延
    initial begin
        clk = 0;
        forever #5ns clk = ~clk;  // 5ns周期でクロック生成
    end

    initial begin
        data = 0;
        #10ns;       // 10ナノ秒待機
        data = 1;
        #100us;      // 100マイクロ秒待機
        data = 0;
        #1ms;        // 1ミリ秒待機
        $finish;
    end
endmodule
```

利用可能な時間単位は以下の通りです。

| 単位 | 意味 | 記法例 |
|------|------|--------|
| `s` | 秒 | `1s` |
| `ms` | ミリ秒 | `10ms` |
| `us` | マイクロ秒 | `100us` |
| `ns` | ナノ秒 | `5ns` |
| `ps` | ピコ秒 | `250ps` |
| `fs` | フェムト秒 | `100fs` |

---

## 3.7 まとめ

本章では、SystemVerilogの基本データ型とリテラルについて体系的に解説しました。以下に重要なポイントを整理します。

**4値論理と2値論理**
- `logic`型（4値）はRTL設計の標準型であり、Verilog時代の`reg`/`wire`を多くの場面で置き換えます
- `bit`型（2値）はテストベンチでの使用に適していますが、初期化忘れを検出できないリスクがあります
- 複数ドライバが存在する場合は`wire`型を使用します

**整数型**
- 2値整数型（`byte`, `shortint`, `int`, `longint`）は主にテストベンチで使用します
- 4値整数型（`integer`, `time`）はVerilogとの互換性を保ちます
- `signed`/`unsigned`修飾子でデフォルトの符号属性を変更できます

**ユーザー定義型**
- `typedef`で型に別名を付けて可読性を向上させます
- `enum`は状態定義やコマンド定義に活用します（ステートマシン設計で特に有効です）
- `packed struct`はRTL設計で、`unpacked struct`はテストベンチで活用します
- `union`は同一ビットパターンの多面的な解釈に使用します

**定数**
- `parameter`はモジュールのパラメータ化に使い、インスタンス化時に変更可能です
- `localparam`はモジュール内部の導出定数に使い、外部から変更不可です
- `const`は実行時定数として、クラスや関数内で使用します
- `specparam`はタイミング仕様の記述に特化しています

**リテラル**
- 数値リテラルは`<幅>'<基数><値>`の形式で記述します
- アンダースコアで桁区切りが可能です
- `'0`, `'1`, `'x`, `'z`で全ビット一括指定ができます
- `string`型は豊富なメソッドを持っています
- 時間リテラルで遅延値を直感的に記述できます

次章（第4章）では、配列と集合について解説します。動的配列、連想配列、キューなど、SystemVerilogの強力なデータ構造を学びます。
