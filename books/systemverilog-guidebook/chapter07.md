---
title: "第7章：タスクと関数"
---

# 第7章：タスクと関数

## 7.1 この章で学ぶこと

本章では、SystemVerilogにおけるサブルーチンである**タスク（task）**と**関数（function）**について詳しく解説します。ソフトウェアプログラミング言語における「関数」や「プロシージャ」に相当するこれらの機能は、コードの再利用性と可読性を大幅に向上させます。タスクと関数の違い、引数の受け渡し方法、そして再帰呼び出しにおける `automatic` 指定の重要性を学びます。

---

## 7.2 タスクと関数の違い

SystemVerilogでは、再利用可能なコードブロックを定義するために**タスク（task）**と**関数（function）**の2つの仕組みが用意されています。両者は似ていますが、重要な違いがあります。

### 7.2.1 関数（function）

関数は、**時間経過を含まない**処理を記述するために使います。つまり、`#`（遅延）、`@`（イベント待ち）、`wait` などの時間制御文を関数内で使用することはできません。関数は必ず**戻り値を返す**ことができます（`void` 型の関数を除く）。

```systemverilog
// 関数の基本的な定義
function int add(int a, int b);
    return a + b;
endfunction

// void型関数（戻り値なし）
function void print_value(int val);
    $display("Value = %0d", val);
endfunction
```

関数の特徴をまとめると以下のようになります：

- **時間経過を含めることができない**（`#`, `@`, `wait` は使用不可）
- **戻り値を持てる**（`void` を指定すれば戻り値なしも可）
- **組み合わせ回路の記述**に適している
- `always_comb` ブロック内から呼び出せる

### 7.2.2 タスク（task）

タスクは、**時間経過を含む**処理を記述できます。クロック待ちやイベント待ちを含むテストベンチの処理に広く使われます。タスクは関数とは異なり、戻り値を直接返す構文は持ちませんが、`output` 引数や `ref` 引数を通じて値を返すことは可能です。

```systemverilog
// タスクの基本的な定義
task automatic send_data(input logic [7:0] data);
    @(posedge clk);       // クロックの立ち上がりを待つ
    valid <= 1'b1;
    dout  <= data;
    @(posedge clk);
    valid <= 1'b0;
endtask
```

タスクの特徴をまとめると以下のようになります：

- **時間経過を含むことができる**（`#`, `@`, `wait` が使用可能）
- **戻り値を持たない**（ただし `output`/`ref` 引数で値を返せる）
- **テストベンチの手順記述**に適している
- `always_comb` ブロック内からは呼び出せない

### 7.2.3 比較表

| 特性 | function | task |
|------|----------|------|
| 時間経過（`#`, `@`, `wait`） | 不可 | 可能 |
| 戻り値 | あり（void可） | なし（output/refで代替） |
| 他のタスクの呼び出し | 不可 | 可能 |
| 他の関数の呼び出し | 可能 | 可能 |
| `always_comb` 内での呼び出し | 可能 | 不可 |
| 合成可能性 | 高い | 低い（テストベンチ向け） |

![タスクと関数の違い](/images/systemverilog-complete-guide/ch07_task_vs_function.drawio.png)

---

## 7.3 関数の詳細

### 7.3.1 関数の宣言と呼び出し

関数は `function` キーワードで宣言します。戻り値の型を指定し、`return` 文で値を返します。

```systemverilog
// 戻り値の型を明示した関数
function logic [7:0] max(logic [7:0] a, logic [7:0] b);
    if (a > b)
        return a;
    else
        return b;
endfunction

// 関数の呼び出し
module example;
    logic [7:0] x, y, result;

    initial begin
        x = 8'd100;
        y = 8'd200;
        result = max(x, y);  // result = 200
        $display("Max value = %0d", result);
    end
endmodule
```

### 7.3.2 暗黙の戻り値

関数名自体を変数として使い、値を代入する方法もあります。これはVerilog時代からの構文です。

```systemverilog
function logic [7:0] min(logic [7:0] a, logic [7:0] b);
    if (a < b)
        min = a;    // 関数名に代入 = 戻り値の設定
    else
        min = b;
endfunction
```

### 7.3.3 void型関数

戻り値が不要な場合は `void` 型を指定します。void型関数はタスクに似ていますが、時間経過を含められない点が異なります。

```systemverilog
function void check_range(int value, int min_val, int max_val);
    if (value < min_val || value > max_val)
        $error("Value %0d is out of range [%0d:%0d]", value, min_val, max_val);
endfunction
```

---

## 7.4 タスクの詳細

### 7.4.1 タスクの宣言と呼び出し

タスクは `task` キーワードで宣言します。時間制御文を含められるため、テストベンチでのバス・プロトコルの実装に広く使われます。

```systemverilog
// AXI-Liteの書き込みトランザクションを模倣するタスク
task automatic axi_write(
    input  logic [31:0] addr,
    input  logic [31:0] data
);
    // アドレスフェーズ
    @(posedge aclk);
    awvalid <= 1'b1;
    awaddr  <= addr;
    @(posedge aclk iff awready);
    awvalid <= 1'b0;

    // データフェーズ
    wvalid <= 1'b1;
    wdata  <= data;
    @(posedge aclk iff wready);
    wvalid <= 1'b0;

    // 応答フェーズ
    @(posedge aclk iff bvalid);
    bready <= 1'b1;
    @(posedge aclk);
    bready <= 1'b0;
endtask
```

### 7.4.2 タスクでの値の返却

タスクは関数とは異なり、戻り値を持ちません。値を呼び出し元に返す場合は、`output` 引数を使用します。ただし、タスクでも `return` 文を使用することは可能です。タスクにおける `return` 文は値を返すためではなく、**タスクの早期終了**（途中で処理を打ち切る）ために使用します。エラー条件の検出時などに、残りの処理をスキップして即座にタスクを終了させたい場合に便利です。

```systemverilog
// return文による早期終了の例
task automatic send_packet(input logic [7:0] data, input bit enable);
    if (!enable) return;  // 無効なら即座にタスクを終了

    @(posedge clk);
    valid <= 1'b1;
    dout  <= data;
    @(posedge clk);
    valid <= 1'b0;
endtask
```

`output` 引数を使って値を返す例は以下のとおりです。

```systemverilog
task automatic read_register(
    input  logic [15:0] addr,
    output logic [31:0] data,
    output logic        error
);
    @(posedge clk);
    rd_en   <= 1'b1;
    rd_addr <= addr;
    @(posedge clk iff rd_valid);
    data  = rd_data;
    error = rd_error;
    rd_en <= 1'b0;
endtask

// 呼び出し例
initial begin
    logic [31:0] rdata;
    logic        rerr;
    read_register(16'h0010, rdata, rerr);
    $display("Read data = %h, Error = %b", rdata, rerr);
end
```

---

## 7.5 引数の受け渡し

SystemVerilogでは、タスクと関数の引数に4つの方向（direction）を指定できます。これにより、データの流れを明確にし、意図しない副作用を防ぎます。

### 7.5.1 input（入力引数）

呼び出し側から値を受け取ります。デフォルトの方向です。タスク/関数の開始時に値がコピーされ、内部で変更しても呼び出し元の値は変わりません。

```systemverilog
function void show(input int x);
    $display("x = %0d", x);
    x = 999;  // この変更は呼び出し元に影響しない
endfunction
```

### 7.5.2 output（出力引数）

タスク/関数から呼び出し元へ値を返します。タスク/関数の終了時に値がコピーされます。

```systemverilog
task automatic divide(
    input  int dividend,
    input  int divisor,
    output int quotient,
    output int remainder
);
    #1;  // 何らかの処理時間
    quotient  = dividend / divisor;
    remainder = dividend % divisor;
endtask
```

### 7.5.3 inout（入出力引数）

呼び出し時と終了時の両方で値がコピーされます。入力としても出力としても機能します。

```systemverilog
task automatic swap(inout int a, inout int b);
    int temp;
    temp = a;
    a = b;
    b = temp;
endtask

// 使用例
initial begin
    int x = 10, y = 20;
    swap(x, y);
    $display("x = %0d, y = %0d", x, y);  // x = 20, y = 10
end
```

### 7.5.4 ref（参照渡し）

`ref` は引数を**参照渡し**（pass by reference）で受け取ります。これはC++の参照に相当し、値のコピーを行わず元の変数を直接操作します。大きなデータ構造（配列やオブジェクトなど）を渡す場合にパフォーマンス上の利点があります。

```systemverilog
// ref引数を使って大きな配列を効率的に処理
function automatic int sum_array(ref int arr[]);
    int total = 0;
    foreach (arr[i])
        total += arr[i];
    return total;
endfunction

// 使用例
initial begin
    int data[] = '{1, 2, 3, 4, 5};
    $display("Sum = %0d", sum_array(data));  // Sum = 15
end
```

**重要**: `ref` 引数は `automatic` なタスク/関数でのみ使用できます。また、変更を防ぎたい場合は `const ref` を指定します。

```systemverilog
// const ref: 読み取り専用の参照渡し
function automatic void print_array(const ref int arr[]);
    foreach (arr[i])
        $display("arr[%0d] = %0d", i, arr[i]);
    // arr[0] = 999;  // コンパイルエラー！ const refは変更不可
endfunction
```

### 7.5.5 引数方向の比較

| 方向 | コピータイミング | 用途 | 備考 |
|------|----------------|------|------|
| `input` | 呼び出し時にコピー | 値を受け取る（デフォルト） | 内部変更は呼び出し元に影響しない |
| `output` | 終了時にコピー | 値を返す | 開始時の値は不定 |
| `inout` | 呼び出し時と終了時 | 値の受け取りと返却 | 双方向のデータ移動 |
| `ref` | コピーなし（参照） | 大きなデータの効率的な受け渡し | `automatic` 必須 |

![引数の受け渡し方向](/images/systemverilog-complete-guide/ch07_argument_directions.drawio.png)

---

## 7.6 デフォルト引数値

SystemVerilogでは、関数やタスクの引数にデフォルト値を設定できます。呼び出し時に引数を省略するとデフォルト値が使用されます。

```systemverilog
function automatic void configure(
    input int timeout = 100,
    input int retries = 3,
    input bit verbose = 1'b0
);
    $display("Timeout=%0d, Retries=%0d, Verbose=%0b", timeout, retries, verbose);
endfunction

// 呼び出し例
initial begin
    configure();             // デフォルト値を使用: 100, 3, 0
    configure(200);          // timeout=200, 他はデフォルト
    configure(200, 5);       // timeout=200, retries=5, verbose=デフォルト
    configure(200, 5, 1'b1); // すべて指定
end
```

---

## 7.7 再帰呼び出しと automatic 指定

### 7.7.1 static と automatic の違い

SystemVerilogのタスクと関数は、デフォルトでは**static**（静的）です。ただし、デフォルトのライフタイムはコンテキストによって異なります。**モジュール内で宣言されたサブルーチンはデフォルトで `static`** ですが、**クラス内で宣言されたサブルーチンはデフォルトで `automatic`** です。static なサブルーチンでは、ローカル変数がすべての呼び出しで共有されます。これは再帰呼び出しやマルチスレッド環境で問題を引き起こします。

一方、**automatic** を指定すると、各呼び出しごとに独立したローカル変数領域が確保されます。再帰呼び出しを行う場合は必ず `automatic` を指定する必要があります。

```systemverilog
// ★ staticでの問題例
function int factorial_static(int n);  // デフォルトはstatic
    int result;  // 全呼び出しで共有される！
    if (n <= 1) begin
        result = 1;
    end else begin
        result = n * factorial_static(n - 1);  // 正しく動作しない
    end
    return result;
endfunction

// ★ automaticによる正しい再帰
function automatic int factorial(int n);
    if (n <= 1)
        return 1;
    else
        return n * factorial(n - 1);
endfunction

// 使用例
initial begin
    $display("5! = %0d", factorial(5));  // 5! = 120
end
```

### 7.7.2 automatic 指定が必要な場面

以下の場面では `automatic` 指定が必須または推奨されます：

1. **再帰呼び出し**: ローカル変数が呼び出しごとに独立している必要がある
2. **fork-join 内での呼び出し**: 複数のスレッドが同時に同じタスクを実行する場合
3. **大規模テストベンチ**: 予期しない変数共有によるバグを防ぐ

```systemverilog
// fork-joinでのautomatic指定の重要性
task automatic drive_channel(input int ch_id, input logic [7:0] data);
    $display("[%0t] Channel %0d: Sending 0x%h", $time, ch_id, data);
    @(posedge clk);
    chan_data[ch_id] <= data;
    chan_valid[ch_id] <= 1'b1;
    @(posedge clk);
    chan_valid[ch_id] <= 1'b0;
endtask

initial begin
    fork
        drive_channel(0, 8'hAA);
        drive_channel(1, 8'hBB);
        drive_channel(2, 8'hCC);
    join
end
```

### 7.7.3 再帰の実践例：フィボナッチ数列

```systemverilog
function automatic int fibonacci(int n);
    if (n <= 0) return 0;
    if (n == 1) return 1;
    return fibonacci(n - 1) + fibonacci(n - 2);
endfunction

// 二分探索の例
function automatic int binary_search(
    ref int arr[],
    int target,
    int low,
    int high
);
    int mid;
    if (low > high) return -1;  // 見つからない

    mid = (low + high) / 2;
    if (arr[mid] == target)
        return mid;
    else if (arr[mid] < target)
        return binary_search(arr, target, mid + 1, high);
    else
        return binary_search(arr, target, low, mid - 1);
endfunction
```

---

## 7.8 モジュール内でのタスクと関数の活用

### 7.8.1 組み合わせ回路での関数利用

関数は合成可能な記述にも使用でき、組み合わせ回路の見通しを良くします。

```systemverilog
module alu (
    input  logic [7:0]  a, b,
    input  logic [2:0]  op,
    output logic [7:0]  result,
    output logic        zero_flag
);

    // ALU演算を関数として定義
    function automatic logic [7:0] alu_operation(
        input logic [7:0] operand_a,
        input logic [7:0] operand_b,
        input logic [2:0] operation
    );
        case (operation)
            3'b000: return operand_a + operand_b;   // 加算
            3'b001: return operand_a - operand_b;   // 減算
            3'b010: return operand_a & operand_b;   // AND
            3'b011: return operand_a | operand_b;   // OR
            3'b100: return operand_a ^ operand_b;   // XOR
            3'b101: return operand_a << 1;           // 左シフト
            3'b110: return operand_a >> 1;           // 右シフト
            default: return 8'h00;
        endcase
    endfunction

    assign result    = alu_operation(a, b, op);
    assign zero_flag = (result == 8'h00);

endmodule
```

### 7.8.2 テストベンチでのタスク活用

テストベンチでは、バスプロトコルの操作を再利用可能なタスクとして定義することが一般的です。

```systemverilog
module tb_memory;
    logic        clk, rst_n;
    logic        wr_en, rd_en;
    logic [15:0] addr;
    logic [31:0] wdata, rdata;

    // クロック生成
    initial clk = 0;
    always #5 clk = ~clk;

    // リセットタスク
    task automatic reset_dut();
        rst_n = 1'b0;
        repeat (5) @(posedge clk);
        rst_n = 1'b1;
        @(posedge clk);
        $display("[%0t] Reset complete", $time);
    endtask

    // 書き込みタスク
    task automatic write_mem(input logic [15:0] a, input logic [31:0] d);
        @(posedge clk);
        wr_en <= 1'b1;
        addr  <= a;
        wdata <= d;
        @(posedge clk);
        wr_en <= 1'b0;
        $display("[%0t] Write: addr=0x%h, data=0x%h", $time, a, d);
    endtask

    // 読み出しタスク
    task automatic read_mem(input logic [15:0] a, output logic [31:0] d);
        @(posedge clk);
        rd_en <= 1'b1;
        addr  <= a;
        @(posedge clk);
        d = rdata;
        rd_en <= 1'b0;
        $display("[%0t] Read: addr=0x%h, data=0x%h", $time, a, d);
    endtask

    // テストシナリオ
    initial begin
        reset_dut();
        write_mem(16'h0000, 32'hDEAD_BEEF);
        write_mem(16'h0004, 32'hCAFE_BABE);

        begin
            logic [31:0] read_data;
            read_mem(16'h0000, read_data);
            assert (read_data == 32'hDEAD_BEEF)
                else $error("Data mismatch!");
        end
        $finish;
    end
endmodule
```

---

## 7.9 まとめ

本章では、SystemVerilogのタスクと関数について学びました。重要なポイントを振り返ります。

1. **関数（function）** は時間経過を含まない処理に使い、戻り値を返せる。組み合わせ回路の記述にも利用可能。
2. **タスク（task）** は時間経過を含む処理に使い、テストベンチでのプロトコル操作に最適。
3. **引数の方向**（`input`, `output`, `inout`, `ref`）を適切に指定することで、データの流れを明確にできる。
4. **`ref` 引数**は参照渡しで、大きなデータ構造を効率的に扱える。`automatic` 必須。
5. **`automatic` 指定**は再帰呼び出しやマルチスレッド環境で必須。staticでは変数が共有されるため、予期しない動作の原因になる。
6. **デフォルト引数**を活用することで、柔軟で使いやすいインターフェースを設計できる。

次章では、これらのタスクや関数、型定義をパッケージとしてまとめ、プロジェクト全体で共有する方法について学びます。
