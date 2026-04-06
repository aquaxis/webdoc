---
title: "第1章：ハードウェア記述言語の進化"
---

# 第1章：ハードウェア記述言語の進化

## 1.1 この章で学ぶこと

本章では、ハードウェア記述言語（HDL: Hardware Description Language）がどのような歴史的背景のもとで発展してきたかを概観します。特に、Verilog HDLの誕生から始まり、IEEE標準化を経て、現代のSystemVerilogに至るまでの進化の流れを追います。

具体的には、以下の内容を扱います。

- Verilog-95、Verilog-2001、SystemVerilog IEEE 1800 の各バージョンにおける主要な改善点
- SystemVerilogが誕生した背景と、その必要性
- VHDL、SystemC、Chisel、Bluespecといった他のハードウェア関連言語との比較
- SystemVerilogを学ぶメリットとデメリット

これらを理解することで、なぜ今SystemVerilogを学ぶ価値があるのかが明確になるでしょう。

---

## 1.2 HDLの歴史

### 1.2.1 HDLとは何か

HDL（Hardware Description Language）とは、デジタル回路の構造や動作をテキストベースで記述するためのプログラミング言語です。一般的なソフトウェアプログラミング言語（C言語やPythonなど）がCPU上で実行される命令列を記述するのに対し、HDLはFPGA（Field-Programmable Gate Array）やASIC（Application-Specific Integrated Circuit）といったハードウェアの論理回路そのものを記述します。

HDLで書かれたコードは、**論理合成**（Synthesis）というプロセスを経て、実際のゲートレベルの回路に変換されます。つまり、HDLは「プログラムを書く」というよりも「回路を設計する」ための言語です。

![HDLの役割：コードから回路への変換フロー](/images/systemverilog-complete-guide/ch01_hdl_flow.drawio.png)

### 1.2.2 Verilog-95（IEEE 1364-1995）

Verilogは1984年にGateway Design Automation社のPhil Moorby氏とPrabhu Goel氏によって開発されました。当初はシミュレーション用の言語として設計されましたが、その後Cadence Design Systems社が買収し、1995年にIEEE 1364-1995として初めて国際標準化されました。これが一般に「Verilog-95」と呼ばれるバージョンです。

Verilog-95の主な特徴は以下の通りです。

- **モジュールベースの設計**: `module`キーワードによる階層的な設計が可能
- **4値論理**: `0`（Low）、`1`（High）、`x`（不定）、`z`（ハイインピーダンス）の4状態を扱える
- **ゲートレベル記述とRTL記述の両方をサポート**: 基本ゲート（AND、OR、NOTなど）の直接記述から、`always`ブロックによる動作レベルの記述まで対応
- **テストベンチの記述**: シミュレーション用の刺激（テストベクタ）を記述可能

ただし、Verilog-95にはいくつかの制限がありました。例えば、ポート宣言の冗長さ、ジェネレート文の欠如、符号付き演算のサポート不足などが挙げられます。

以下はVerilog-95スタイルのモジュール記述の例です。

```systemverilog
// Verilog-95 スタイル：ポート宣言が冗長
module adder(a, b, sum, cout);
    input  [3:0] a;     // 4ビット入力 a
    input  [3:0] b;     // 4ビット入力 b
    output [3:0] sum;   // 4ビット加算結果
    output       cout;  // キャリー出力（桁上げ）

    assign {cout, sum} = a + b;
endmodule
```

このように、Verilog-95ではモジュールのポートリストにまず信号名だけを列挙し、その後に個別に方向（`input`/`output`）とビット幅を宣言する必要がありました。

### 1.2.3 Verilog-2001（IEEE 1364-2001）

2001年に発行されたIEEE 1364-2001、通称「Verilog-2001」では、Verilog-95の多くの不満点が解消されました。主要な改善点は次の通りです。

- **ANSI Cスタイルのポート宣言**: ポートの方向・型・ビット幅をモジュールヘッダ内で一括宣言できるようになった
- **符号付きデータ型（`signed`）の導入**: 符号付き演算が言語レベルでサポートされた
- **`generate`文の導入**: パラメータに基づく条件付きハードウェア生成が可能になった
- **`always @*`（自動感度リスト）**: 組み合わせ回路の記述時に、感度リストの記載漏れを防止
- **多次元配列**: メモリや複雑なデータ構造の記述が容易になった

先ほどのVerilog-95のコードをVerilog-2001スタイルで書き直すと、以下のようになります。

```systemverilog
// Verilog-2001 スタイル：ANSI Cスタイルのポート宣言で簡潔に
module adder (
    input  wire [3:0] a,     // 4ビット入力 a
    input  wire [3:0] b,     // 4ビット入力 b
    output wire [3:0] sum,   // 4ビット加算結果
    output wire       cout   // キャリー出力
);
    assign {cout, sum} = a + b;
endmodule
```

ポートの宣言がモジュールヘッダにまとまり、可読性が大きく向上しました。Verilog-2001は、現在でも設計の基礎として広く使われています。

### 1.2.4 SystemVerilog（IEEE 1800）

2005年、Accellera（半導体業界の標準化団体）が策定した仕様がIEEE 1800-2005として標準化され、**SystemVerilog**が誕生しました。その後、2009年にIEEE 1364（Verilog）とIEEE 1800（SystemVerilog）が統合され、IEEE 1800-2009として一本化されました。さらに2012年、2017年、2023年と改訂が続けられています。

SystemVerilogはVerilogの上位互換であり、Verilogのすべての機能を含みつつ、以下のような大幅な拡張が加えられました。

- **新しいデータ型**: `logic`、`bit`、`byte`、`int`、`shortint`、`longint`、`string`、列挙型（`enum`）、構造体（`struct`）、共用体（`union`）
- **インターフェース（`interface`）**: モジュール間の信号接続を抽象化し、再利用性を向上
- **オブジェクト指向プログラミング（OOP）**: `class`、継承（`extends`）、多態性（ポリモーフィズム）による高度な検証環境の構築
- **制約付きランダム生成（Constrained Random）**: テストケースの自動生成
- **アサーション（SVA: SystemVerilog Assertions）**: 設計仕様の形式的な検証
- **カバレッジ（Coverage）**: 機能カバレッジによるテスト網羅率の測定
- **プロセス制御**: `fork`/`join`による並行処理の制御

![Verilog-95からSystemVerilogへの進化の流れ](/images/systemverilog-complete-guide/ch01_hdl_evolution.drawio.png)

---

## 1.3 SystemVerilogの誕生理由

### 1.3.1 検証の複雑化

半導体チップのトランジスタ数は、ムーアの法則に従い約2年ごとに倍増してきました。チップの複雑さが増すにつれて、設計そのものよりも**検証**（Verification）がプロジェクト全体の工数の大部分を占めるようになりました。現代のSoC（System on Chip）開発では、プロジェクト工数の60〜70%が検証作業に費やされるとも言われています。

Verilog-2001の言語機能だけでは、このような大規模な検証を効率的に行うことが困難でした。特に、ランダムテストの生成、カバレッジの収集、再利用可能なテスト環境の構築といった高度な検証手法を実現するには、より強力な言語機能が必要でした。

### 1.3.2 HVL（Hardware Verification Language）の統合

SystemVerilog以前は、設計にはVerilogを、検証にはe言語（Specman）やVera（Synopsys社）といったHVL（Hardware Verification Language：ハードウェア検証言語）を別々に使用するのが一般的でした。これには以下のような問題がありました。

- **2つの言語を習得する負担**: 設計者と検証者がそれぞれ異なる言語を学ぶ必要があった
- **ツールのライセンスコスト**: 異なるEDAツールを複数購入する必要があった
- **設計と検証間の不整合**: 異なる言語間でのデータ型の違いやインターフェースの不一致が発生しやすかった

SystemVerilogは、設計言語と検証言語を**単一の言語**に統合することで、これらの問題を解決しました。1つの言語で設計（RTL記述）も検証（テストベンチ、アサーション、カバレッジ）もすべて行えるようになったのです。

### 1.3.3 抽象度の向上

チップ設計の規模が拡大するにつれ、ゲートレベルやRTL（Register Transfer Level）だけでの記述では設計の生産性に限界がありました。SystemVerilogでは、インターフェース、パッケージ（`package`）、クラスといった高い抽象度の機能が導入され、より効率的な設計・検証が可能になりました。

---

## 1.4 他の言語との比較

ハードウェア設計・検証に使われる言語はSystemVerilogだけではありません。以下の表で、代表的な言語との比較を示します。

| 特徴 | SystemVerilog | VHDL | SystemC | Chisel | Bluespec |
|------|-------------|------|---------|--------|----------|
| **標準規格** | IEEE 1800 | IEEE 1076 | IEEE 1666 | オープンソース | オープンソース |
| **主な用途** | 設計 + 検証 | 設計 + 検証 | システムレベル設計・検証 | RTL設計（Scala DSL） | 高レベル合成 |
| **ベース言語** | Verilog拡張 | Ada風の独自言語 | C++ライブラリ | Scala | Haskell風の独自言語 |
| **OOPサポート** | あり | 限定的（VHDL-2008） | あり（C++） | あり（Scala） | なし |
| **ランダム検証** | 組み込み | 外部ライブラリ必要 | 外部ライブラリ必要 | 限定的 | 限定的 |
| **アサーション** | SVA（組み込み） | PSL（外部） | なし | なし | あり |
| **業界での普及度** | 非常に高い | 高い（特に欧州・軍事） | 中程度 | 成長中 | ニッチ |
| **学習コスト** | 中〜高 | 高い | 高い（C++知識必要） | 中（Scala知識必要） | 高い |
| **EDAツール対応** | 主要3社すべて対応 | 主要3社すべて対応 | 限定的 | Verilog変換で対応 | Verilog変換で対応 |

ここで言う「主要3社」とは、Synopsys、Cadence、Siemens EDA（旧Mentor Graphics）の3大EDAベンダーのことを指します。

### 各言語の簡単な説明

- **VHDL**: 米国国防総省が主導して開発した言語で、Adaプログラミング言語の影響を強く受けています。厳密な型チェックが特徴で、軍事・航空宇宙分野や欧州で広く使われています。
- **SystemC**: C++のクラスライブラリとして実装されており、ハードウェアとソフトウェアを統合的にモデリングできます。トランザクションレベルモデリング（TLM）に特に強みがあります。
- **Chisel**: カリフォルニア大学バークレー校で開発された、Scala言語をベースとするハードウェア構成言語です。RISC-Vプロセッサの設計に使われたことで注目を集めました。最終的にVerilogコードを生成します。
- **Bluespec**: ルールベースの設計手法を採用した高レベルなハードウェア記述言語です。自動的な並行制御が特徴ですが、採用事例は限定的です。

---

## 1.5 VerilogとSystemVerilogの違い：コード例

実際にコード例を見ることで、VerilogからSystemVerilogへの進化を体感しましょう。

### 1.5.1 列挙型によるステートマシンの改善

従来のVerilogでは、ステートマシン（有限状態機械）の状態を`parameter`や`` `define``で定義していました。SystemVerilogでは`enum`（列挙型）を使うことで、より安全で可読性の高いコードを書くことができます。

```systemverilog
// --- Verilog-2001 スタイル ---
// ステートの定義に parameter を使用
module fsm_verilog (
    input  wire       clk,
    input  wire       rst_n,    // アクティブLowリセット
    input  wire       start,
    output reg  [1:0] state
);
    // 状態の定数を parameter で定義
    parameter IDLE = 2'b00;
    parameter RUN  = 2'b01;
    parameter DONE = 2'b10;

    always @(posedge clk or negedge rst_n) begin
        if (!rst_n)
            state <= IDLE;
        else begin
            case (state)
                IDLE: if (start) state <= RUN;
                RUN:             state <= DONE;
                DONE:            state <= IDLE;
                default:         state <= IDLE;
            endcase
        end
    end
endmodule
```

```systemverilog
// --- SystemVerilog スタイル ---
// enum で型安全なステートマシンを実現
module fsm_sv (
    input  logic clk,
    input  logic rst_n,
    input  logic start,
    output logic [1:0] state
);
    // 列挙型で状態を定義（型安全・可読性向上）
    typedef enum logic [1:0] {
        IDLE = 2'b00,
        RUN  = 2'b01,
        DONE = 2'b10
    } state_t;

    state_t current_state;

    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n)
            current_state <= IDLE;
        else begin
            case (current_state)
                IDLE: if (start) current_state <= RUN;
                RUN:             current_state <= DONE;
                DONE:            current_state <= IDLE;
                default:         current_state <= IDLE;
            endcase
        end
    end

    assign state = current_state;
endmodule
```

SystemVerilog版では以下の改善が見られます。

- **`typedef enum`**: 状態を列挙型として定義することで、無効な値の代入をコンパイル時に検出できる
- **`logic`型**: `wire`と`reg`の区別が不要になり、統一的に`logic`で宣言できる
- **`always_ff`**: フリップフロップ（順序回路）専用の`always`ブロック。合成ツールが設計意図を正確に理解でき、誤った記述を検出しやすくなる

### 1.5.2 always ブロックの使い分け

SystemVerilogでは、回路の種類に応じて`always`ブロックが3つに分類されました。

```systemverilog
// 組み合わせ回路（Combinational Logic）
// 入力の変化に応じて即座に出力が決まる回路
always_comb begin
    result = a & b;  // AND演算
end

// 順序回路（Sequential Logic）
// クロックの立ち上がりエッジで値が更新されるフリップフロップ
always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n)
        q <= '0;     // リセット時は0にクリア
    else
        q <= d;      // クロックエッジでデータを取り込む
end

// ラッチ（Latch）
// レベルセンシティブな記憶素子（意図的に使う場合のみ）
always_latch begin
    if (enable)
        q <= d;
end
```

Verilog-2001では、これらすべてが`always`ブロック1種類で記述されていたため、設計者の意図（組み合わせ回路なのか順序回路なのか）が曖昧になることがありました。SystemVerilogの`always_comb`、`always_ff`、`always_latch`を使うことで、意図しないラッチの生成などの設計ミスを早期に発見できます。

![always_comb, always_ff, always_latch の使い分け](/images/systemverilog-complete-guide/ch01_always_blocks.drawio.png)

---

## 1.6 SystemVerilogのメリットとデメリット

### メリット

1. **設計と検証の一体化**
   1つの言語でRTL設計もテストベンチも記述できるため、設計者と検証者の間のコミュニケーションコストが低減します。また、UVM（Universal Verification Methodology）という業界標準の検証手法がSystemVerilog上に構築されており、再利用可能な検証環境を効率的に構築できます。

2. **強力なランダム生成**
   `rand`や`randc`キーワードを使った制約付きランダム生成（Constrained Random Generation）により、膨大な数のテストケースを自動的に生成できます。人手では到底網羅しきれないコーナーケースを効率よくテストすることが可能です。

3. **アサーション機能（SVA）**
   SystemVerilog Assertions（SVA）を使うことで、「クロックの立ち上がりから3サイクル以内にACK信号がアサートされる」といった時間的な性質を宣言的に記述し、シミュレーション中やフォーマル検証で自動的にチェックできます。

4. **OOP（オブジェクト指向プログラミング）対応**
   クラス、継承、ポリモーフィズムといったOOP機能により、検証コンポーネントの再利用性と拡張性が大幅に向上します。特にUVMはOOPを全面的に活用しています。

5. **豊富なデータ型**
   `enum`、`struct`、`union`、動的配列、連想配列、キューなど、ソフトウェア言語に近い豊富なデータ型が利用でき、複雑なデータ構造を自然に表現できます。

### デメリット

1. **膨大な仕様**
   IEEE 1800規格書は1300ページを超える分量があり、言語仕様が非常に広範です。すべての機能を把握するには長期間の学習が必要であり、初学者にとってはどこから手をつけるべきか判断しにくい場合があります。

2. **ツール間でのサポート状況の差**
   SystemVerilogの全機能をすべてのEDAツールが同じレベルでサポートしているわけではありません。特に、合成ツールがサポートするSystemVerilogのサブセットと、シミュレータがサポートする範囲には差があります。あるツールで動作するコードが別のツールではエラーになることもあり得ます。

3. **学習コスト**
   設計側の機能だけであれば比較的短期間で習得できますが、検証側の機能（OOP、ランダム生成、アサーション、カバレッジ、UVM）まで含めると、習得すべき内容は非常に多くなります。ソフトウェアエンジニアリングの知識（オブジェクト指向設計パターンなど）も求められるため、ハードウェア設計のみの経験者には追加の学習が必要です。

4. **シミュレーション性能**
   高度な検証機能（ランダム制約ソルバー、カバレッジ収集など）を多用すると、シミュレーションの実行速度が低下する場合があります。大規模設計ではシミュレーション時間の管理が課題となることがあります。

---

## 1.7 まとめ

本章では、ハードウェア記述言語（HDL）の歴史と進化について学びました。主要なポイントを振り返ります。

- **Verilog-95**（IEEE 1364-1995）はHDLの基礎を確立したが、ポート宣言の冗長さや符号付き演算の不足などの制限があった
- **Verilog-2001**（IEEE 1364-2001）ではANSI Cスタイルのポート宣言、`generate`文、`always @*`などが導入され、設計の利便性が大きく向上した
- **SystemVerilog**（IEEE 1800）はVerilogの上位互換として、設計言語と検証言語を統合した。`logic`型、列挙型、構造体、クラス、アサーション、カバレッジなどの強力な機能が追加された
- SystemVerilog誕生の背景には、**検証の複雑化**、**HVLの統合**、**抽象度の向上**の3つの要因があった
- VHDL、SystemC、Chisel、Bluespecなどの代替言語と比較して、SystemVerilogは設計と検証の両面をカバーする唯一の業界標準言語であり、EDAツールのサポートも最も充実している
- メリットとしては設計・検証の一体化、ランダム生成、アサーション、OOP対応が挙げられ、デメリットとしては仕様の膨大さ、ツール間の差異、学習コストがある

次章では、SystemVerilogの開発環境のセットアップと、最初のシミュレーションを実行する方法について解説します。
