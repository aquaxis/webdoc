---
title: "第2章：開発環境とワークフロー"
---

# 第2章：開発環境とワークフロー

## 2.1 この章で学ぶこと

SystemVerilogで設計や検証を行うには、まずソースコードを記述し、それをシミュレータで実行して動作を確認するという一連のワークフローを理解する必要がある。本章では、以下の内容を扱う。

- **シミュレータの種類と選択基準**：商用シミュレータとオープンソースシミュレータそれぞれの特徴と使い分け
- **コンパイルからシミュレーション実行までの流れ**：コンパイル、エラボレーション、シミュレーションの3つのフェーズの役割
- **IEEE 1800規格書の読み方**：エンジニアとして一次情報にあたる重要性と、規格書を実際に活用する方法

これらの知識は、SystemVerilogの文法を学ぶ前に身につけておくべき基盤である。どれほど優れたコードを書いても、開発環境を正しく構築し、ツールの挙動を理解していなければ、効率的な開発は不可能だからである。

---

## 2.2 シミュレータの種類

SystemVerilogのシミュレータは、大きく「商用シミュレータ」と「オープンソースシミュレータ」の2種類に分類される。プロジェクトの規模、予算、必要とする機能に応じて適切なツールを選択することが重要である。

![図2.1: シミュレータの分類と特徴](/images/systemverilog-guidebook/ch02_simulator_overview.drawio.png)

### 2.2.1 商用シミュレータ

商用シミュレータは、SystemVerilog IEEE 1800規格の広範なサポート、高度なデバッグ機能、そしてプロフェッショナルなテクニカルサポートを提供する。半導体業界の実務では、ほぼ例外なく商用シミュレータが使用されている。

#### VCS（Synopsys）

VCS（Verilog Compiler Simulator）はSynopsys社が提供するシミュレータであり、業界で最も広く使われているツールの一つである。

**特徴：**
- コンパイル型のシミュレータであり、ソースコードをネイティブコードにコンパイルして実行するため、大規模設計でも高速にシミュレーションできる
- SystemVerilog IEEE 1800規格への対応が早く、最新仕様のサポートが充実している
- Verdi（波形ビューア・デバッガ）との統合が強力で、RTLからゲートレベルまでの階層的なデバッグが可能である
- UVM（Universal Verification Methodology）のリファレンス実装を最初にサポートしたシミュレータの一つである
- カバレッジの統合管理機能が優れている

**基本的なコマンド例：**

**リスト2.1: シンプルな加算器モジュール（simple_adder.sv）**

```systemverilog
// サンプルデザイン: simple_adder.sv
module simple_adder (
    input  logic [7:0] a, b,
    output logic [8:0] sum
);
    assign sum = a + b;
endmodule
```

```bash
# コンパイルとエラボレーション（単一ステップ）
vcs -sverilog -full64 simple_adder.sv tb_simple_adder.sv -o simv

# シミュレーション実行
./simv

# 波形ダンプ付きでコンパイル
vcs -sverilog -full64 -debug_access+all simple_adder.sv tb_simple_adder.sv -o simv

# シミュレーション実行（波形をFSDBフォーマットで保存）
./simv +fsdbfile+dump.fsdb

# Verdiで波形を確認
verdi -ssf dump.fsdb &
```

#### Questa（Siemens EDA、旧Mentor Graphics）

Questa（旧称ModelSim）はSiemens EDA社が提供するシミュレータである。長い歴史を持ち、特にFPGA開発分野での採用実績が豊富である。

**特徴：**
- インタプリタ型とコンパイル型の両方のモードを持ち、用途に応じて使い分けられる
- GUIベースの統合デバッグ環境が洗練されており、波形表示、ソースコードとの連動、信号のドラッグ&ドロップなど直感的な操作が可能である
- FPGA向けのシミュレーションライブラリが充実しており、Xilinx（AMD）やIntel FPGAの開発フローとの統合がスムーズである
- 形式検証ツール（Questa Formal）やCDC（Clock Domain Crossing）検証ツールとの連携が強い
- アサーションベース検証（ABV）のサポートが手厚い

**基本的なコマンド例：**

```bash
# ライブラリの作成
vlib work

# コンパイル
vlog -sv simple_adder.sv tb_simple_adder.sv

# シミュレーション実行（バッチモード）
vsim -batch -do "run -all; quit" work.tb_simple_adder

# シミュレーション実行（GUIモード）
vsim work.tb_simple_adder

# コンパイルとシミュレーションをまとめて実行
vlog -sv simple_adder.sv tb_simple_adder.sv && vsim -batch -do "run -all; quit" work.tb_simple_adder
```

#### Xcelium（Cadence）

XceliumはCadence社が提供するシミュレータであり、前身のIUS（Incisive Unified Simulator）から大幅に性能が向上している。

**特徴：**
- マルチコアシミュレーションに対応しており、大規模SoC設計のシミュレーション高速化に優れている
- SystemVerilog、VHDL、SystemCの混在設計（ミックスドランゲージ）のサポートが強力である
- Cadence社の検証IP（VIP）が豊富で、バスプロトコル（AXI、PCIe、USBなど）の検証を効率化できる
- SimVision（波形ビューア）による高度なデバッグ機能を提供する
- Specmanとの連携による独自の検証メソドロジ（e言語）もサポートしている

**基本的なコマンド例：**

```bash
# コンパイル、エラボレーション、シミュレーションを一括実行
xrun simple_adder.sv tb_simple_adder.sv

# コンパイルのみ
xrun -compile simple_adder.sv tb_simple_adder.sv

# エラボレーションのみ
xrun -elaborate simple_adder.sv tb_simple_adder.sv

# 波形ダンプ付きで実行
xrun -access +rwc simple_adder.sv tb_simple_adder.sv

# マルチコアシミュレーション
xrun -mccodegen simple_adder.sv tb_simple_adder.sv
```

![図2.2: 商用シミュレータ比較（VCS / Questa / Xcelium）](/images/systemverilog-guidebook/ch02_commercial_simulators.drawio.png)

### 2.2.2 オープンソースシミュレータ

オープンソースシミュレータは無償で利用できるため、学習用途や小規模プロジェクト、CI/CD環境での自動テストなどに活用されている。ただし、SystemVerilogの全機能をサポートしているわけではないため、その制限事項を理解した上で使用する必要がある。

#### Verilator

VerilatorはC++/SystemCベースの高速シミュレータであり、オープンソースのSystemVerilogシミュレータとして最も広く使われている。

**特徴：**
- ソースコードをC++に変換してコンパイルするため、他のシミュレータと比較して桁違いに高速である
- 2値シミュレーション（0と1のみ）であるため、`x`（不定値）や`z`（ハイインピーダンス）は扱えない
- 合成可能なRTLコードのシミュレーションに特化しており、テストベンチの記述にはC++を使用する
- Lintチェック機能が優れており、コーディングスタイルの統一にも活用できる
- 大規模なオープンソースプロジェクト（RISC-V実装など）で広く採用されている
- バージョン5以降でSystemVerilogの一部検証構文（クラス、ランダム制約など）のサポートが進んでいるが、まだ限定的である

**制限事項：**
- 4値論理（`x`, `z`）をサポートしない
- `fork-join`などの並行プロセス制御のサポートが限定的である
- タイミング制御（`#delay`、`@(posedge clk)`）のサポートが限定的である
- UVMは部分的にしかサポートされていない

**基本的なコマンド例：**

```bash
# Lintチェックのみ
verilator --lint-only -Wall simple_adder.sv

# C++テストベンチを使ったシミュレーション
verilator --cc --exe --build -Wall simple_adder.sv tb_simple_adder.cpp

# 実行
./obj_dir/Vsimple_adder

# 波形ダンプ付きでビルド
verilator --cc --exe --build --trace simple_adder.sv tb_simple_adder.cpp

# 実行して波形を確認
./obj_dir/Vsimple_adder
gtkwave dump.vcd &

# SystemVerilogテストベンチを使用する場合（Verilator 5以降）
verilator --binary -Wall simple_adder.sv tb_simple_adder.sv
```

#### Icarus Verilog

Icarus Verilog（iverilog）は軽量なオープンソースシミュレータであり、Verilog HDLの学習用として長い歴史を持つ。

**特徴：**
- 非常に軽量で、インストールが容易である
- 4値シミュレーションに対応しており、`x`や`z`の挙動を確認できる
- Verilog-2005規格のサポートが充実している
- SystemVerilogの基本的な構文（`logic`型、`always_comb`など）をある程度サポートしている
- VPIインターフェースを通じたC言語連携が可能である

**制限事項：**
- SystemVerilogの高度な機能（クラス、制約付きランダム、アサーションなど）はほとんどサポートされていない
- 大規模設計でのシミュレーション速度は商用シミュレータに大きく劣る
- UVMは非サポートである
- メンテナンスの頻度が低下しており、新しい規格への対応は遅い

**基本的なコマンド例：**

```bash
# コンパイル
iverilog -g2012 -o sim.vvp simple_adder.sv tb_simple_adder.sv

# シミュレーション実行
vvp sim.vvp

# 波形ダンプ付きで実行
vvp sim.vvp -lxt2
gtkwave dump.lxt &

# 複数ファイルをファイルリストで指定
iverilog -g2012 -o sim.vvp -f filelist.txt
```

![図2.3: オープンソースシミュレータ比較（Verilator / Icarus Verilog）](/images/systemverilog-guidebook/ch02_opensource_simulators.drawio.png)

### 2.2.3 シミュレータ比較表

各シミュレータの特徴を表2.1にまとめる。

**表2.1: シミュレータ比較表**
| 項目 | VCS | Questa | Xcelium | Verilator | Icarus Verilog |
|------|-----|--------|---------|-----------|----------------|
| 提供元 | Synopsys | Siemens EDA | Cadence | OSS | OSS |
| ライセンス | 商用 | 商用 | 商用 | LGPL | GPL |
| SV対応範囲 | 完全 | 完全 | 完全 | 部分的（設計中心） | 限定的 |
| 4値シミュレーション | 対応 | 対応 | 対応 | 非対応（2値のみ） | 対応 |
| UVMサポート | 完全 | 完全 | 完全 | 部分的 | 非対応 |
| シミュレーション速度 | 高速 | 中〜高速 | 高速 | 非常に高速 | 低〜中速 |
| デバッグ機能 | Verdi | GUI統合 | SimVision | 限定的 | 限定的 |
| 主な用途 | ASIC設計・検証 | FPGA / ASIC | SoC設計・検証 | RTL検証・CI | 学習・小規模設計 |
| コスト | 高額 | 中〜高額 | 高額 | 無料 | 無料 |

プロジェクトの要件に応じてシミュレータを選択することが重要である。企業での本格的なASIC/SoC開発には商用シミュレータが必須であり、個人学習やオープンソースプロジェクトではVerilatorやIcarus Verilogを活用するのが現実的な選択となる。

---

## 2.3 コンパイルとエラボレーション

SystemVerilogのソースコードがシミュレーション結果を出力するまでには、複数のフェーズを経る。この流れを正確に理解することは、エラーメッセージの解釈やデバッグにおいて極めて重要である。

![図2.4: コンパイルからシミュレーション実行までのワークフロー](/images/systemverilog-guidebook/ch02_workflow.drawio.png)

### 2.3.1 全体の流れ

ソースコードからシミュレーション結果が得られるまでの流れは、以下の3つのフェーズに分かれる。

1. **コンパイルフェーズ**（Compilation）
2. **エラボレーションフェーズ**（Elaboration）
3. **シミュレーション実行フェーズ**（Simulation）

シミュレータによっては、これらのフェーズを明示的に分離して実行するものもあれば、一括で処理するものもある。しかし、内部的にはどのシミュレータもこの3つのフェーズを順に実行している。

### 2.3.2 コンパイルフェーズ

コンパイルフェーズでは、ソースコードに対して以下の処理が行われる。

**字句解析（Lexical Analysis）：**
ソースコードをトークン（識別子、キーワード、演算子、リテラルなど）に分解する。これがコンパイルの最初のステップであり、ソースコードを意味のある最小単位に分割する。

**構文解析（Parsing）：**
字句解析で得られたトークン列が、SystemVerilogの文法規則に従っているかを検査する。構文エラーがあればこの段階で検出される。例えば、セミコロンの欠落、括弧の不一致、未定義の演算子の使用などがここで報告される。

**型チェック（Type Checking）：**
変数の型が正しく使用されているかを検査する。例えば、ビット幅の不一致や、不正な型変換の検出がここで行われる。

**リスト2.2: コンパイルフェーズで検出されるエラーの例**

```systemverilog
module compile_example (
    input  logic [7:0] data_in,
    output logic [3:0] data_out
);
    // コンパイルフェーズで検出されるエラーの例:
    // assign data_out = data_in + ;  // 構文エラー: 式が不完全
    // assign data_out = undefined_signal; // 未定義信号の参照

    // ビット幅の不一致は警告として報告されることが多い
    assign data_out = data_in[3:0];  // 正しい記述
endmodule
```

### 2.3.3 エラボレーションフェーズ

エラボレーションフェーズは、コンパイルされた設計を「実体化」するフェーズである。ハードウェア設計特有の処理が多く含まれるため、ソフトウェア開発にはない概念が登場する。

**パラメータの展開（Parameter Resolution）：**
`parameter`や`localparam`で定義された値が確定し、それに基づいて回路の構成が決定される。

**階層構造の構築（Hierarchy Construction）：**
モジュールのインスタンスが展開され、設計全体の階層ツリーが構築される。トップモジュールから再帰的にすべてのサブモジュールがインスタンス化される。

**ジェネレート文の展開（Generate Block Expansion）：**
`generate-for`や`generate-if`によって条件的に生成される回路が、パラメータ値に基づいて確定する。

**リスト2.3: パラメータ化されたモジュールとエラボレーション**

```systemverilog
module parameterized_adder #(
    parameter int WIDTH = 8   // エラボレーション時にWIDTHの値が確定する
) (
    input  logic [WIDTH-1:0] a, b,
    output logic [WIDTH:0]   sum
);
    assign sum = a + b;
endmodule

module top;
    logic [7:0]  a8, b8;
    logic [8:0]  sum8;
    logic [15:0] a16, b16;
    logic [16:0] sum16;

    // エラボレーション時に、WIDTHがそれぞれ8と16に展開される
    parameterized_adder #(.WIDTH(8))  u_add8  (.a(a8),  .b(b8),  .sum(sum8));
    parameterized_adder #(.WIDTH(16)) u_add16 (.a(a16), .b(b16), .sum(sum16));
endmodule
```

上記の例では、`parameterized_adder`というモジュールが2つインスタンス化されている。エラボレーション時に、`u_add8`では`WIDTH=8`、`u_add16`では`WIDTH=16`としてそれぞれ異なる回路が生成される。

### 2.3.4 シミュレーション実行フェーズ

エラボレーションが完了すると、シミュレーションが実行される。このフェーズでは以下の処理が行われる。

- **初期化**：変数の初期値設定、`initial`ブロックの実行開始
- **イベント駆動シミュレーション**：信号の変化（イベント）に基づいてシミュレーションが進行する
- **時間の進行**：`#delay`やクロックエッジなどのタイミング制御に従ってシミュレーション時間が進む
- **結果出力**：`$display`や`$monitor`による出力、波形ファイルへのダンプ

**リスト2.4: シミュレーション実行フェーズのテストベンチ例**

```systemverilog
module tb_simple_adder;
    logic [7:0] a, b;
    logic [8:0] sum;

    simple_adder u_dut (.a(a), .b(b), .sum(sum));

    initial begin
        // シミュレーション実行フェーズで逐次実行される
        a = 8'd10; b = 8'd20;
        #10;  // 10単位時間待機
        $display("a=%0d, b=%0d, sum=%0d", a, b, sum);  // 出力: a=10, b=20, sum=30

        a = 8'd255; b = 8'd1;
        #10;
        $display("a=%0d, b=%0d, sum=%0d", a, b, sum);  // 出力: a=255, b=1, sum=256

        $finish;
    end
endmodule
```

### 2.3.5 各シミュレータでの実行フロー比較

各シミュレータでの3フェーズの実行方法を以下にまとめる。

```bash
# === VCS (Synopsys) ===
# コンパイル + エラボレーション（一括）
vcs -sverilog -full64 design.sv testbench.sv -o simv
# シミュレーション実行
./simv

# === Questa (Siemens EDA) ===
# ライブラリ作成
vlib work
# コンパイル
vlog -sv design.sv testbench.sv
# エラボレーション + シミュレーション実行
vsim -batch -do "run -all; quit" work.tb_top

# === Xcelium (Cadence) ===
# 全フェーズ一括実行
xrun design.sv testbench.sv
# または各フェーズを分離
xrun -compile design.sv testbench.sv
xrun -elaborate design.sv testbench.sv
xrun -R  # シミュレーション実行のみ

# === Verilator ===
# コンパイル（C++への変換 + ビルド）
verilator --binary -Wall design.sv testbench.sv
# シミュレーション実行
./obj_dir/Vdesign

# === Icarus Verilog ===
# コンパイル
iverilog -g2012 -o sim.vvp design.sv testbench.sv
# シミュレーション実行
vvp sim.vvp
```

---

## 2.4 IEEE 1800規格書の見方

### 2.4.1 規格書とは何か

IEEE 1800はSystemVerilogの言語仕様を定めた国際規格である。正式名称は「IEEE Standard for SystemVerilog -- Unified Hardware Design, Specification, and Verification Language」であり、設計、仕様記述、検証のすべてをカバーする統一的な言語仕様書である。

規格書は、シミュレータの実装者がツールを開発するための「設計図」であると同時に、ユーザーが言語の正確な挙動を理解するための「最終的な拠り所」でもある。教科書やチュートリアルは分かりやすさを重視して内容を簡略化していることが多いが、規格書には曖昧さのない厳密な定義が記されている。

最新の規格はIEEE 1800-2023であり、IEEE SAから購入またはIEEE GET Programを通じて無料で閲覧できる場合がある。

### 2.4.2 規格書のセクション構成

IEEE 1800規格書は約1300ページにおよぶ大部な文書であるが、論理的に整理されたセクション構成になっている。主要なセクションの概要は以下の通りである。

- **Section 1-4**：概要、参考文献、用語定義、規約
- **Section 5**：字句規約（トークン、キーワード、リテラル）
- **Section 6-7**：データ型、集合型（配列）
- **Section 8**：クラス
- **Section 9**：プロセス（`always`、`initial`、`fork-join`など）
- **Section 10**：代入文
- **Section 11-12**：演算子、手続き的プログラミング文
- **Section 13**：タスクと関数
- **Section 14-16**：アサーション、カバレッジ
- **Section 23**：モジュールとポート
- **Section 25**：インターフェース
- **Section 35**：DPI（Direct Programming Interface）

規格書を最初から読み通す必要はない。実務では、特定の構文や機能について疑問が生じたときに、該当するセクションを参照するという使い方が一般的である。

![図2.5: IEEE 1800規格書のセクション構成](/images/systemverilog-guidebook/ch02_ieee1800_structure.drawio.png)

### 2.4.3 BNF記法の読み方

規格書では、言語の構文をBNF（Backus-Naur Form）記法で厳密に定義している。BNFはプログラミング言語の文法を形式的に記述するための記法であり、規格書を読む上で避けて通れない。

以下にBNF記法の基本ルールを示す。

- `::=`は「〜として定義される」を意味する
- `|`は「または」を意味する（選択肢）
- `[ ]`は「省略可能」を意味する
- `{ }`は「0回以上の繰り返し」を意味する
- **太字**またはそのままの表記はキーワード（終端記号）
- *斜体*は別の場所で定義される構文要素（非終端記号）

例えば、`module`宣言のBNF定義は以下のような形式で記述されている（簡略化した例）。

```
module_declaration ::=
    module_ansi_header [ timeunits_declaration ] { module_item } endmodule [ : module_identifier ]

module_ansi_header ::=
    { attribute_instance } module_keyword [ lifetime ] module_identifier
    [ parameter_port_list ] [ list_of_port_declarations ] ;

module_keyword ::= module | macromodule
```

この定義を読むと、以下のことが分かる。

1. `module`宣言は、`module_ansi_header`で始まり、0個以上の`module_item`を含み、`endmodule`で終わる
2. `endmodule`の後にはオプションで`: モジュール名`を付けることができる
3. `module_keyword`は`module`または`macromodule`のいずれかである

これを実際のSystemVerilogコードに対応させると以下のようになる。

**リスト2.5: BNF定義と実際のコードの対応例**

```systemverilog
module simple_counter #(          // module_ansi_header の開始
    parameter int WIDTH = 8       // parameter_port_list
) (                               // list_of_port_declarations の開始
    input  logic             clk,
    input  logic             rst_n,
    output logic [WIDTH-1:0] count
);                                // module_ansi_header の終了

    // module_item（モジュール内の記述）
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n)
            count <= '0;
        else
            count <= count + 1'b1;
    end

endmodule : simple_counter        // endmodule [ : module_identifier ]
```

**追加のBNF記法**

規格書では、上記の基本ルールに加えて以下の記法も使用される。

- `( )`は「グルーピング」を意味する（複数の要素をまとめる）
- `[ ] { }`の組み合わせで「0回以上の繰り返し（省略可能）」を表現
- `||=`は「非終端記号の定義を継続」を意味する

**if文のBNF例**

if文のBNF定義を見て、構文を理解する練習をしよう。

```
conditional_statement ::=
    if ( expression ) statement_or_null [ else statement_or_null ]
  | if ( expression ) statement_or_null else if ( expression ) statement_or_null
    { else if ( expression ) statement_or_null } [ else statement_or_null ]
```

この定義から読み取れることは以下である。

1. `if (条件) 文` は必須、`else` は省略可能
2. `else if` は0回以上繰り返し可能
3. 最後の `else` は省略可能

これを実際のコードに対応させると以下のようになる。

```systemverilog
if (a > b)              // if ( expression )
    result = 1;         // statement_or_null
else if (a == b)        // else if ( expression )
    result = 0;         // statement_or_null
else                    // [ else statement_or_null ]
    result = -1;
```

**BNFを読む際のポイント**

1. **非終端記号を追跡する**: 斜体で書かれた記号は別の場所で定義されているため、該当する定義を探す必要がある
2. **再帰的定義に注意**: 一部の構文（式、リストなど）は再帰的に定義されており、同じパターンが繰り返し現れる
3. **省略可能な要素を見逃さない**: `[ ]`で囲まれた要素は省略可能であり、最も単純な形式から複雑な形式まで存在する
4. **規格書の付録を活用**: IEEE 1800規格書の付録Aには、全BNF定義が集約されている

### 2.4.4 一次情報にあたることの重要性

エンジニアとして、一次情報（規格書）にあたる習慣を身につけることは極めて重要である。その理由を以下に挙げる。

**正確性の担保：** ブログ記事やチュートリアルには誤りが含まれていることがある。規格書は言語仕様の唯一の正式な定義であり、最も信頼性が高い情報源である。

**ツール間の挙動差の理解：** シミュレータによって挙動が異なる場面に遭遇することがある。そのような場合、規格書を参照することで、どのシミュレータの挙動が正しいのか（あるいは規格上未定義なのか）を判断できる。

**エッジケースへの対応：** 一般的なチュートリアルでは触れられないような特殊なケースに遭遇した場合、規格書が唯一の指針となる。

**技術的議論の基盤：** チーム内での技術的な議論において、規格書の該当箇所を引用できることは、議論の質を大きく向上させる。

規格書を読むことに最初は抵抗を感じるかもしれないが、日常的に参照する習慣をつけることで、次第に必要な情報を素早く見つけられるようになる。まずは自分が使っている構文に対応するセクションから読み始めるとよいであろう。

---

## 2.5 まとめ

本章では、SystemVerilogの開発環境とワークフローについて学んだ。要点を以下に整理する。

**シミュレータの選択：**
- 商用シミュレータ（VCS、Questa、Xcelium）はSystemVerilog規格を完全にサポートしており、UVM対応、高度なデバッグ機能、テクニカルサポートを提供する。半導体企業での実務にはこれらが不可欠である
- オープンソースシミュレータ（Verilator、Icarus Verilog）は無償で利用でき、学習や小規模開発に適している。ただし、SystemVerilogの対応範囲に制限がある
- Verilatorは2値シミュレーションに特化しており、RTL検証やCI環境での高速テストに優れている
- Icarus Verilogは4値シミュレーションに対応しているが、SystemVerilogの高度な機能はサポートしていない

**コンパイルからシミュレーションまでの3フェーズ：**
- コンパイルフェーズでは字句解析、構文解析、型チェックが順に行われる
- エラボレーションフェーズではパラメータの展開と階層構造の構築が行われる。これはハードウェア設計特有のフェーズである
- シミュレーション実行フェーズではイベント駆動シミュレーションにより回路の動作が再現される

**IEEE 1800規格書：**
- 規格書はSystemVerilogの言語仕様を定めた唯一の公式文書である
- BNF記法を理解することで、言語構文の厳密な定義を読み解くことができる
- エンジニアとして一次情報にあたる習慣を持つことは、正確な理解とツール間の挙動差の解決に不可欠である

次章では、SystemVerilogの基本データ型とリテラルについて詳しく学ぶ。4値論理と2値論理の違い、符号付き・符号なしの型、ユーザー定義型など、設計と検証の両方に関わる基礎的な概念を扱う。
