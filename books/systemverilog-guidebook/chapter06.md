---
title: "第6章：モジュールとインターフェース"
---

# 第6章：モジュールとインターフェース

## 6.1 この章で学ぶこと

SystemVerilogにおける設計の基本単位は**モジュール（module）**です。モジュールは、ハードウェアの機能ブロックを表現し、それらを階層的に組み合わせることでシステム全体を構築します。しかし、設計規模が大きくなるにつれて、モジュール間の信号接続は複雑化し、保守性の低下を招きます。この問題を解決するのが**インターフェース（interface）**です。

本章では、以下の内容を扱います。

- **モジュールの基本構造**とポート宣言の方法
- **名前付きポート接続**、**ドット接続**、**ワイルドカード接続**によるインスタンス化の手法
- **パラメータ化されたモジュール**による再利用性の向上
- **generate文**による条件付き・繰り返しインスタンス化
- **インターフェース**の基本概念と構文
- **modport**による入出力方向の制御
- インターフェース内での**タスク・関数の定義**
- **パラメータ化されたインターフェース**の活用
- **バスプロトコル（AXIなど）**へのインターフェース適用例

これらを理解することで、大規模なハードウェア設計においても、見通しの良い階層構造と効率的な信号接続を実現できるようになります。

![モジュールとインターフェースの関係](/images/systemverilog-complete-guide/ch06_module_interface_overview.drawio.png)

---

## 6.2 モジュールの基本構造

### 6.2.1 モジュールとポート宣言

モジュールは `module` キーワードで始まり `endmodule` で終わります。モジュールのポート（入出力端子）は、外部との信号のやりとりを定義します。SystemVerilogでは、ポートの方向と型をポートリスト内に直接記述する**ANSI形式**が推奨されます。

```systemverilog
// ANSI形式のポート宣言（推奨）
module adder (
    input  logic [7:0] a,
    input  logic [7:0] b,
    output logic [8:0] sum
);
    assign sum = a + b;
endmodule
```

旧来のVerilog形式（非ANSI形式）では、ポート名のリストとポート宣言が分離していました。この形式は可読性が低いため、新規設計では使用しないことを推奨します。

```systemverilog
// 非ANSI形式（旧来のVerilog形式、非推奨）
module adder(a, b, sum);
    input  [7:0] a;
    input  [7:0] b;
    output [8:0] sum;

    assign sum = a + b;
endmodule
```

### 6.2.2 モジュールのインスタンス化

モジュールを使用するには、上位モジュール内でインスタンス化します。インスタンス化とは、モジュールの「実体」を生成し、信号を接続する操作です。

```systemverilog
module top (
    input  logic        clk,
    input  logic        rst_n,
    input  logic [7:0]  data_a,
    input  logic [7:0]  data_b,
    output logic [8:0]  result
);
    // adderモジュールのインスタンス化
    adder u_adder (
        .a   (data_a),
        .b   (data_b),
        .sum (result)
    );
endmodule
```

ここで `u_adder` はインスタンス名であり、同じモジュールを複数回インスタンス化する場合はそれぞれ異なる名前を付ける必要があります。

![モジュールの階層構造](/images/systemverilog-complete-guide/ch06_module_hierarchy.drawio.png)

---

## 6.3 ポート接続の方法

SystemVerilogでは、モジュールのポートを接続する方法が複数用意されています。設計の規模や状況に応じて使い分けることで、コードの簡潔さと安全性を両立できます。

### 6.3.1 名前付きポート接続（.name(signal)形式）

最も基本的で安全な接続方法です。ポート名と接続する信号を明示的に指定します。ポートの順序に依存しないため、ポートの追加や並び替えがあっても安全に対応できます。

```systemverilog
module processor (
    input  logic        clk,
    input  logic        rst_n,
    input  logic [31:0] instr,
    output logic [31:0] addr,
    output logic        mem_we
);
    // ... 内部実装 ...
endmodule

module soc_top (
    input  logic        sys_clk,
    input  logic        sys_rst_n
);
    logic [31:0] instruction;
    logic [31:0] mem_addr;
    logic        write_enable;

    // 名前付きポート接続
    processor u_proc (
        .clk    (sys_clk),
        .rst_n  (sys_rst_n),
        .instr  (instruction),
        .addr   (mem_addr),
        .mem_we (write_enable)
    );
endmodule
```

### 6.3.2 ドット接続（.name形式）

上位モジュールの信号名とポート名が同一の場合、信号名の指定を省略できます。これを**暗黙の名前付きポート接続**と呼びます。異なる名前の信号は、通常の名前付きポート接続で個別に指定します。

```systemverilog
module soc_top (
    input  logic        clk,      // processorのポート名と同じ
    input  logic        rst_n     // processorのポート名と同じ
);
    logic [31:0] instr;           // processorのポート名と同じ
    logic [31:0] addr;            // processorのポート名と同じ
    logic        mem_we;          // processorのポート名と同じ

    // ドット接続（.name形式）
    // 全てのポート名と信号名が一致しているため省略可能
    processor u_proc (
        .clk,          // .clk(clk) と同等
        .rst_n,        // .rst_n(rst_n) と同等
        .instr,        // .instr(instr) と同等
        .addr,         // .addr(addr) と同等
        .mem_we        // .mem_we(mem_we) と同等
    );
endmodule
```

この形式は、信号名の命名規則が統一されたプロジェクトで特に有効です。ただし、信号名の不一致に気づきにくいという側面があるため、コードレビュー時には注意が必要です。

### 6.3.3 ワイルドカード接続（.*形式）

`.*` を使用すると、ポート名と同名の信号が自動的に接続されます。一致しないポートのみを明示的に記述すればよいです。

```systemverilog
module soc_top (
    input  logic        sys_clk,
    input  logic        sys_rst_n
);
    logic [31:0] instr;
    logic [31:0] addr;
    logic        mem_we;

    // ワイルドカード接続
    // instr, addr, mem_weは同名なので自動接続
    // clk, rst_nは名前が異なるので明示的に指定
    processor u_proc (
        .clk   (sys_clk),
        .rst_n (sys_rst_n),
        .*                    // 残りは同名の信号に自動接続
    );
endmodule
```

ワイルドカード接続は非常に便利ですが、意図しない接続が発生するリスクがあります。また、コードを読む際にどの信号が接続されているかが一見して分かりにくいです。チームの方針に応じて使用を判断すべきです。

### 6.3.4 各接続方法の比較

| 接続方法 | 記法 | 利点 | 注意点 |
|----------|------|------|--------|
| 名前付き | `.name(signal)` | 最も明確で安全 | 記述量が多い |
| ドット | `.name` | 同名時に簡潔 | 名前不一致に気づきにくい |
| ワイルドカード | `.*` | 最も簡潔 | 意図しない接続のリスク |
| 順序ベース | `(sig1, sig2, ...)` | 短い記述 | 順序依存で極めて危険 |

---

## 6.4 パラメータ化されたモジュール

### 6.4.1 parameterによる汎用化

`parameter` を使用すると、モジュールのビット幅やその他の特性を外部から設定できます。これにより、同じモジュールを異なる構成で再利用できます。

```systemverilog
module counter #(
    parameter int WIDTH = 8,          // カウンタのビット幅
    parameter bit WRAP  = 1'b1        // ラップアラウンドの有無
)(
    input  logic              clk,
    input  logic              rst_n,
    input  logic              enable,
    output logic [WIDTH-1:0]  count
);
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n)
            count <= '0;
        else if (enable) begin
            if (WRAP || count != {WIDTH{1'b1}})
                count <= count + 1'b1;
        end
    end
endmodule
```

### 6.4.2 パラメータ化されたモジュールのインスタンス化

パラメータは `#(...)` 構文でインスタンス化時にオーバーライドします。

```systemverilog
module system_top (
    input  logic        clk,
    input  logic        rst_n
);
    logic        cnt8_en, cnt16_en;
    logic [7:0]  cnt8_val;
    logic [15:0] cnt16_val;

    // 8ビットカウンタ（デフォルトパラメータを使用）
    counter u_cnt8 (
        .clk,
        .rst_n,
        .enable (cnt8_en),
        .count  (cnt8_val)
    );

    // 16ビットカウンタ（パラメータをオーバーライド）
    counter #(
        .WIDTH (16),
        .WRAP  (1'b0)
    ) u_cnt16 (
        .clk,
        .rst_n,
        .enable (cnt16_en),
        .count  (cnt16_val)
    );
endmodule
```

---

## 6.5 generate文による条件付きインスタンス化

`generate` 文を使用すると、条件に応じてモジュールのインスタンス化やロジックの生成を制御できます。コンパイル時に評価されるため、合成可能な条件分岐として機能します。

### 6.5.1 for-generateによる繰り返しインスタンス化

```systemverilog
module parallel_adders #(
    parameter int NUM_ADDERS = 4,
    parameter int WIDTH      = 8
)(
    input  logic [WIDTH-1:0] a [NUM_ADDERS],
    input  logic [WIDTH-1:0] b [NUM_ADDERS],
    output logic [WIDTH:0]   sum [NUM_ADDERS]
);
    // generate forによる複数インスタンスの生成
    for (genvar i = 0; i < NUM_ADDERS; i++) begin : gen_adder
        adder #(.WIDTH(WIDTH)) u_adder (
            .a   (a[i]),
            .b   (b[i]),
            .sum (sum[i])
        );
    end
endmodule
```

`begin : gen_adder` のようにブロックにラベルを付けることは必須です。このラベルは、階層パス（例：`gen_adder[0].u_adder`）でインスタンスを参照する際に使用されます。

### 6.5.2 if-generateによる条件付きインスタンス化

```systemverilog
module fifo_wrapper #(
    parameter int  DEPTH    = 16,
    parameter int  WIDTH    = 8,
    parameter bit  USE_BRAM = 1'b0    // BRAM使用の有無
)(
    input  logic             clk,
    input  logic             rst_n,
    input  logic             wr_en,
    input  logic [WIDTH-1:0] wr_data,
    input  logic             rd_en,
    output logic [WIDTH-1:0] rd_data,
    output logic             full,
    output logic             empty
);
    // パラメータに応じて異なる実装を選択
    if (USE_BRAM) begin : gen_bram_fifo
        bram_fifo #(
            .DEPTH (DEPTH),
            .WIDTH (WIDTH)
        ) u_fifo (.*);
    end else begin : gen_reg_fifo
        reg_fifo #(
            .DEPTH (DEPTH),
            .WIDTH (WIDTH)
        ) u_fifo (.*);
    end
endmodule
```

### 6.5.3 case-generateによる多分岐インスタンス化

```systemverilog
module multiplier #(
    parameter int WIDTH = 8,
    parameter string IMPL = "BASIC"   // 実装方式の選択
)(
    input  logic [WIDTH-1:0]   a,
    input  logic [WIDTH-1:0]   b,
    output logic [2*WIDTH-1:0] product
);
    case (IMPL)
        "BASIC": begin : gen_basic
            assign product = a * b;
        end
        "BOOTH": begin : gen_booth
            booth_multiplier #(.WIDTH(WIDTH)) u_mult (
                .a, .b, .product
            );
        end
        "WALLACE": begin : gen_wallace
            wallace_multiplier #(.WIDTH(WIDTH)) u_mult (
                .a, .b, .product
            );
        end
        default: begin : gen_default
            assign product = a * b;
        end
    endcase
endmodule
```

![generate文による条件付きインスタンス化](/images/systemverilog-complete-guide/ch06_generate_block.drawio.png)

---

## 6.6 インターフェース（Interface）

### 6.6.1 インターフェースが解決する問題

設計規模が大きくなると、モジュール間で多数の信号をやりとりする必要が生じます。たとえば、プロセッサとメモリコントローラの間には、アドレス、データ、制御信号など、数十本の信号が存在することがあります。これらの信号を個別のポートとして宣言し、階層の各レベルで一つ一つ接続していくのは、極めて煩雑でエラーの温床となります。

この問題は**信号のバケツリレー問題**と呼ばれます。中間のモジュールが自身では使用しない信号を、上位から下位へ単にパススルーするだけのために、すべてのポートに宣言しなければならない状況です。

```systemverilog
// バケツリレー問題の例：信号を一つ一つ渡していく
module top;
    logic        clk, rst_n;
    logic [31:0] addr, wdata, rdata;
    logic        we, re, valid, ready;
    logic [3:0]  burst_len;
    logic [1:0]  burst_type;
    logic [2:0]  burst_size;

    middle u_middle (
        .clk, .rst_n,
        .addr, .wdata, .rdata,
        .we, .re, .valid, .ready,
        .burst_len, .burst_type, .burst_size
    );
endmodule

module middle (
    input  logic        clk, rst_n,
    // このモジュール自体は以下の信号を使わないが、
    // 下位モジュールに渡すためだけに宣言が必要
    input  logic [31:0] addr,
    input  logic [31:0] wdata,
    output logic [31:0] rdata,
    input  logic        we, re,
    output logic        valid,
    input  logic        ready,
    input  logic [3:0]  burst_len,
    input  logic [1:0]  burst_type,
    input  logic [2:0]  burst_size
);
    bottom u_bottom (
        .clk, .rst_n,
        .addr, .wdata, .rdata,
        .we, .re, .valid, .ready,
        .burst_len, .burst_type, .burst_size
    );
endmodule
```

インターフェースを使えば、これらの関連信号を一つにまとめ、単一のポートとして受け渡すことができます。

### 6.6.2 インターフェースの基本構文

インターフェースは `interface` キーワードで定義します。内部に信号（`logic` 型など）を宣言し、関連する信号群をひとまとめにします。

```systemverilog
// 簡単なバスインターフェースの定義
interface simple_bus_if;
    logic [31:0] addr;
    logic [31:0] wdata;
    logic [31:0] rdata;
    logic        we;
    logic        re;
    logic        valid;
    logic        ready;
endinterface
```

インターフェースをモジュールのポートとして使用する場合は、以下のように記述します。

```systemverilog
module master (
    input  logic clk,
    input  logic rst_n,
    simple_bus_if bus    // インターフェース型のポート
);
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            bus.addr  <= '0;
            bus.wdata <= '0;
            bus.we    <= 1'b0;
        end else begin
            // バス信号へのアクセスはドット表記
            bus.addr  <= 32'h1000_0000;
            bus.wdata <= 32'hDEAD_BEEF;
            bus.we    <= 1'b1;
        end
    end
endmodule

module slave (
    input  logic clk,
    input  logic rst_n,
    simple_bus_if bus    // 同じインターフェースを参照
);
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            bus.rdata <= '0;
            bus.valid <= 1'b0;
        end else if (bus.re) begin
            bus.rdata <= 32'hCAFE_BABE;
            bus.valid <= 1'b1;
        end
    end
endmodule
```

上位モジュールでの接続は以下のように非常に簡潔になります。

```systemverilog
module top;
    logic clk, rst_n;

    // インターフェースのインスタンス化
    simple_bus_if bus_inst();

    master u_master (
        .clk,
        .rst_n,
        .bus (bus_inst)
    );

    slave u_slave (
        .clk,
        .rst_n,
        .bus (bus_inst)
    );
endmodule
```

![インターフェースによる信号集約](/images/systemverilog-complete-guide/ch06_interface_concept.drawio.png)

### 6.6.3 modportによる入出力方向の定義

インターフェース単体では、各信号の入出力方向が定義されていません。`modport` を使用すると、インターフェースを使用する側の視点から信号の方向を指定できます。これにより、合成ツールが正しく方向を判断でき、誤った方向でのアクセスをコンパイル時に検出できます。

```systemverilog
interface simple_bus_if;
    logic [31:0] addr;
    logic [31:0] wdata;
    logic [31:0] rdata;
    logic        we;
    logic        re;
    logic        valid;
    logic        ready;

    // マスター側から見た方向定義
    modport master_mp (
        output addr, wdata, we, re,
        input  rdata, valid, ready
    );

    // スレーブ側から見た方向定義
    modport slave_mp (
        input  addr, wdata, we, re,
        output rdata, valid, ready
    );
endinterface
```

modportを使用したモジュールの記述は以下のようになります。

```systemverilog
module master (
    input  logic clk,
    input  logic rst_n,
    simple_bus_if.master_mp bus    // modportを指定
);
    // bus.addrへのoutput（書き込み）は許可
    // bus.rdataへのinput（読み出し）は許可
    // bus.rdataへのoutput（書き込み）はコンパイルエラー
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            bus.addr  <= '0;
            bus.we    <= 1'b0;
        end else begin
            bus.addr  <= 32'h0000_1000;
            bus.we    <= 1'b1;
        end
    end
endmodule

module slave (
    input  logic clk,
    input  logic rst_n,
    simple_bus_if.slave_mp bus     // modportを指定
);
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            bus.rdata <= '0;
            bus.valid <= 1'b0;
        end else if (bus.we) begin
            bus.rdata <= 32'hA5A5_A5A5;
            bus.valid <= 1'b1;
        end
    end
endmodule
```

### 6.6.4 インターフェース内でのタスク・関数の定義

インターフェースの強力な特長の一つは、信号だけでなくタスクや関数も内部に定義できる点です。バスプロトコルの操作手順をインターフェース自体にカプセル化することで、利用する側のコードが簡潔になり、プロトコルの一貫性も保証されます。

```systemverilog
interface memory_bus_if (
    input logic clk
);
    logic [31:0] addr;
    logic [31:0] wdata;
    logic [31:0] rdata;
    logic        we;
    logic        valid;
    logic        ready;

    modport master_mp (
        output addr, wdata, we,
        input  rdata, valid, ready,
        import task write_word(input logic [31:0] a, input logic [31:0] d),
        import task read_word(input logic [31:0] a, output logic [31:0] d)
    );

    modport slave_mp (
        input  addr, wdata, we,
        output rdata, valid, ready
    );

    // バス書き込みのプロトコル手順をカプセル化
    task automatic write_word(input logic [31:0] a, input logic [31:0] d);
        @(posedge clk);
        addr  <= a;
        wdata <= d;
        we    <= 1'b1;
        @(posedge clk);
        we    <= 1'b0;
        // readyが返るまで待機
        while (!ready) @(posedge clk);
    endtask

    // バス読み出しのプロトコル手順をカプセル化
    task automatic read_word(input logic [31:0] a, output logic [31:0] d);
        @(posedge clk);
        addr <= a;
        we   <= 1'b0;
        while (!valid) @(posedge clk);
        d = rdata;
    endtask
endinterface
```

テストベンチからの利用例を以下に示します。

```systemverilog
module tb_top;
    logic clk = 0;
    always #5 clk = ~clk;

    memory_bus_if mem_bus(.clk);

    // テストシーケンス
    initial begin
        logic [31:0] read_data;

        // インターフェース内のタスクを呼び出す
        mem_bus.write_word(32'h0000_1000, 32'hDEAD_BEEF);
        mem_bus.read_word(32'h0000_1000, read_data);

        $display("Read data: 0x%08h", read_data);
        $finish;
    end
endmodule
```

### 6.6.5 パラメータ化されたインターフェース

インターフェースもモジュールと同様にパラメータ化できます。アドレス幅やデータ幅をパラメータにすることで、さまざまなバス構成に対応する汎用的なインターフェースを定義できます。

```systemverilog
interface axi_lite_if #(
    parameter int ADDR_WIDTH = 32,
    parameter int DATA_WIDTH = 32
);
    // 書き込みアドレスチャネル
    logic [ADDR_WIDTH-1:0] awaddr;
    logic                  awvalid;
    logic                  awready;

    // 書き込みデータチャネル
    logic [DATA_WIDTH-1:0]   wdata;
    logic [DATA_WIDTH/8-1:0] wstrb;
    logic                    wvalid;
    logic                    wready;

    // 書き込み応答チャネル
    logic [1:0]  bresp;
    logic        bvalid;
    logic        bready;

    // 読み出しアドレスチャネル
    logic [ADDR_WIDTH-1:0] araddr;
    logic                  arvalid;
    logic                  arready;

    // 読み出しデータチャネル
    logic [DATA_WIDTH-1:0] rdata;
    logic [1:0]            rresp;
    logic                  rvalid;
    logic                  rready;

    // マスター側のmodport
    modport master (
        output awaddr, awvalid,
        input  awready,
        output wdata, wstrb, wvalid,
        input  wready,
        input  bresp, bvalid,
        output bready,
        output araddr, arvalid,
        input  arready,
        input  rdata, rresp, rvalid,
        output rready
    );

    // スレーブ側のmodport
    modport slave (
        input  awaddr, awvalid,
        output awready,
        input  wdata, wstrb, wvalid,
        output wready,
        output bresp, bvalid,
        input  bready,
        input  araddr, arvalid,
        output arready,
        output rdata, rresp, rvalid,
        input  rready
    );
endinterface
```

インスタンス化時にパラメータを指定します。

```systemverilog
module soc_top (
    input logic clk,
    input logic rst_n
);
    // 32ビットアドレス、64ビットデータのAXI-Liteバス
    axi_lite_if #(
        .ADDR_WIDTH (32),
        .DATA_WIDTH (64)
    ) axi_bus();

    cpu_core u_cpu (
        .clk,
        .rst_n,
        .m_axi (axi_bus)    // マスター側として接続
    );

    memory_ctrl u_mem (
        .clk,
        .rst_n,
        .s_axi (axi_bus)    // スレーブ側として接続
    );
endmodule
```

### 6.6.6 バス信号（AXI）への適用例

実際のSoC設計では、AXIバスのように多数の信号を持つプロトコルが使用されます。インターフェースを活用した実践的な設計例を示します。

```systemverilog
// AXI-Liteスレーブのレジスタファイル実装例
module axi_lite_regs #(
    parameter int NUM_REGS = 16
)(
    input  logic clk,
    input  logic rst_n,
    axi_lite_if.slave s_axi
);
    logic [s_axi.DATA_WIDTH-1:0] regs [NUM_REGS];

    // 書き込みロジック
    typedef enum logic [1:0] {
        WR_IDLE, WR_ADDR, WR_DATA, WR_RESP
    } wr_state_t;

    wr_state_t wr_state;
    logic [s_axi.ADDR_WIDTH-1:0] wr_addr_reg;

    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            wr_state     <= WR_IDLE;
            s_axi.awready <= 1'b0;
            s_axi.wready  <= 1'b0;
            s_axi.bvalid  <= 1'b0;
            s_axi.bresp   <= 2'b00;
            for (int i = 0; i < NUM_REGS; i++)
                regs[i] <= '0;
        end else begin
            case (wr_state)
                WR_IDLE: begin
                    s_axi.awready <= 1'b1;
                    if (s_axi.awvalid && s_axi.awready) begin
                        wr_addr_reg    <= s_axi.awaddr;
                        s_axi.awready  <= 1'b0;
                        s_axi.wready   <= 1'b1;
                        wr_state       <= WR_DATA;
                    end
                end
                WR_DATA: begin
                    if (s_axi.wvalid && s_axi.wready) begin
                        regs[wr_addr_reg[5:2]] <= s_axi.wdata;
                        s_axi.wready  <= 1'b0;
                        s_axi.bvalid  <= 1'b1;
                        s_axi.bresp   <= 2'b00;  // OKAY
                        wr_state      <= WR_RESP;
                    end
                end
                WR_RESP: begin
                    if (s_axi.bready) begin
                        s_axi.bvalid <= 1'b0;
                        wr_state     <= WR_IDLE;
                    end
                end
                default: wr_state <= WR_IDLE;
            endcase
        end
    end

    // 読み出しロジック（簡略化）
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            s_axi.arready <= 1'b0;
            s_axi.rvalid  <= 1'b0;
            s_axi.rdata   <= '0;
            s_axi.rresp   <= 2'b00;
        end else begin
            s_axi.arready <= 1'b1;
            if (s_axi.arvalid && s_axi.arready) begin
                s_axi.rdata  <= regs[s_axi.araddr[5:2]];
                s_axi.rvalid <= 1'b1;
                s_axi.rresp  <= 2'b00;  // OKAY
            end
            if (s_axi.rvalid && s_axi.rready)
                s_axi.rvalid <= 1'b0;
        end
    end
endmodule
```

このように、インターフェースを用いることで、AXI-Liteバスの約20本の信号を単一のポートとして扱えます。新しいスレーブを追加する場合も、インターフェースのインスタンスを追加して接続するだけでよく、信号の宣言漏れや接続ミスのリスクが大幅に軽減されます。

![AXIバスへのインターフェース適用](/images/systemverilog-complete-guide/ch06_axi_interface.drawio.png)

---

## 6.7 まとめ

本章では、SystemVerilogにおけるモジュールの階層構造とインターフェースについて解説しました。以下に要点を整理します。

**モジュールの階層構造**

- モジュールはハードウェア設計の基本単位であり、`module` / `endmodule` で定義します
- ポート宣言はANSI形式を推奨します
- ポート接続には、**名前付き接続**（`.name(signal)`）、**ドット接続**（`.name`）、**ワイルドカード接続**（`.*`）の3つの方法があります
- 名前付き接続が最も安全であり、ワイルドカード接続は利便性が高い反面、注意が必要です
- `parameter` を使用することで、ビット幅などを外部から設定可能な汎用モジュールを作成できます
- `generate` 文（`for`、`if`、`case`）により、条件付きまたは繰り返しのインスタンス化が可能です

**インターフェース**

- インターフェースは、関連する信号群を一つにまとめ、モジュール間の接続を簡潔にする仕組みです
- 信号のバケツリレー問題を解決し、中間モジュールでの不要なポート宣言を排除できます
- `modport` を使用して、使用する側の視点から信号の入出力方向を定義します
- インターフェース内にタスクや関数を定義し、バスプロトコルの操作手順をカプセル化できます
- パラメータ化により、アドレス幅やデータ幅の異なるバス構成に対応する汎用インターフェースを作成できます
- AXIなどの複雑なバスプロトコルの実装において特に大きな効果を発揮します

次章では、本章で触れたタスクと関数について、より詳細に解説します。
