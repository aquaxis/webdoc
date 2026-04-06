---
title: "第5章：組み合わせ回路と順序回路"
---

# 第5章：組み合わせ回路と順序回路

## 5.1 この章で学ぶこと

デジタル回路は大きく**組み合わせ回路（combinational circuit）**と**順序回路（sequential circuit）**の2種類に分類されます。組み合わせ回路は入力の現在の値だけで出力が決まる回路であり、順序回路はクロックに同期して状態を保持する回路です。SystemVerilogでは、これら2種類の回路を明確に区別して記述するための仕組みが用意されています。

本章では、以下の内容を扱います。

- **alwaysブロックの進化**：Verilog時代の`always`から、SystemVerilogの`always_comb`、`always_ff`、`always_latch`への移行
- **代入文の使い分け**：ブロッキング代入（`=`）とノンブロッキング代入（`<=`）の正しい使い方
- **制御フロー構文**：`unique if`、`priority if`、`unique case`、`priority case`による設計意図の明示

これらを正しく理解し使い分けることで、設計者の意図を明確に伝え、シミュレーションと合成結果の一致した信頼性の高いRTLコードを書けるようになります。

![組み合わせ回路と順序回路の概要](/images/systemverilog-complete-guide/ch05_overview.drawio.png)

---

## 5.2 alwaysブロックの進化

### 5.2.1 Verilog時代のalwaysブロックの問題点

Verilog-2001では、すべての手続き型ロジックを汎用的な`always`ブロックで記述していました。このアプローチにはいくつかの深刻な問題がありました。

```systemverilog
// Verilog時代の組み合わせ回路記述
always @(a or b or sel) begin
    if (sel)
        y = a;
    else
        y = b;
end
```

**問題1：感度リストの記述漏れ**

組み合わせ回路では、ブロック内で参照するすべての信号を感度リストに列挙する必要があります。しかし、信号を1つでも書き忘れると、シミュレーション結果と合成結果が一致しないという危険な不具合が生じます。Verilog-2001で導入された`always @(*)`により多少改善されましたが、根本的な解決ではありませんでした。

```systemverilog
// 感度リストの記述漏れ — cが含まれていない
always @(a or b) begin
    y = a & b & c;  // cが変化してもブロックが実行されない
end
```

**問題2：設計意図が不明瞭**

`always`ブロックだけでは、そのブロックが組み合わせ回路なのか、順序回路なのか、あるいは意図的なラッチなのかが構文上わかりません。コードレビューやメンテナンスの際に、設計者の意図を読み取るのが困難でした。

**問題3：意図しないラッチの生成**

組み合わせ回路のつもりで書いた`always`ブロックで、`if`文の`else`節を書き忘れたり、`case`文の分岐を網羅しなかったりすると、合成ツールが意図しないラッチを推論します。これはバグの温床でした。

```systemverilog
// 意図しないラッチが生成される例
always @(*) begin
    if (en)
        q = d;
    // else節がないため、en=0のときqは値を保持 → ラッチが推論される
end
```

### 5.2.2 always_comb：組み合わせ回路専用

`always_comb`は、組み合わせ回路を記述するための専用ブロックです。SystemVerilogの中でも最も頻繁に使われるブロックのひとつであり、以下の特性を持ちます。

- **自動感度リスト**：ブロック内で読み取られるすべての信号が自動的に感度リストに含まれます。手動で感度リストを記述する必要がありません。
- **ラッチ推論の検出**：すべての実行パスで出力に値が代入されていない場合、合成ツールやシミュレータが警告を発します。
- **シミュレーション時間0での実行**：シミュレーション開始時に自動的にブロックが1回実行され、初期値の整合性が確保されます。
- **関数呼び出しの感度追跡**：ブロック内から呼び出される関数の内部で参照される信号も、自動的に感度リストに含まれます。

```systemverilog
// always_combによる組み合わせ回路の記述
module mux4to1 (
    input  logic [7:0] a, b, c, d,
    input  logic [1:0] sel,
    output logic [7:0] y
);

    always_comb begin
        case (sel)
            2'b00:   y = a;
            2'b01:   y = b;
            2'b10:   y = c;
            2'b11:   y = d;
            default: y = 'x;  // 到達しないが安全のため記述
        endcase
    end

endmodule
```

`always_comb`を使用する際のベストプラクティスを以下に示します。

- ブロック内では**ブロッキング代入（`=`）**のみを使用します
- すべての実行パスですべての出力信号に値を代入します
- `default`節を必ず記述します
- ブロック内で代入する信号は、他のブロックから代入しません

![always_combの動作原理](/images/systemverilog-complete-guide/ch05_always_comb.drawio.png)

### 5.2.3 always_ff：順序回路（フリップフロップ）専用

`always_ff`は、クロックエッジに同期して動作するフリップフロップ（順序回路）を記述するための専用ブロックです。感度リストにはクロックエッジとオプションで非同期リセットを記述します。

```systemverilog
// 非同期リセット付きフリップフロップ
module dff_async_rst (
    input  logic       clk,
    input  logic       rst_n,
    input  logic [7:0] d,
    output logic [7:0] q
);

    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n)
            q <= 8'h00;
        else
            q <= d;
    end

endmodule
```

```systemverilog
// 同期リセット付きフリップフロップ
module dff_sync_rst (
    input  logic       clk,
    input  logic       rst_n,
    input  logic [7:0] d,
    output logic [7:0] q
);

    always_ff @(posedge clk) begin
        if (!rst_n)
            q <= 8'h00;
        else
            q <= d;
    end

endmodule
```

`always_ff`を使用する際のベストプラクティスを以下に示します。

- ブロック内では**ノンブロッキング代入（`<=`）**のみを使用します
- 感度リストには`posedge clk`または`negedge clk`を記述します
- 非同期リセットを使う場合は、感度リストに`negedge rst_n`（または`posedge rst`）を追加します
- ブロック内で代入する信号は、他のブロックから代入しません

### 5.2.4 always_latch：ラッチ専用

`always_latch`は、レベルセンシティブなラッチを意図的に設計する場合に使用するブロックです。通常のFPGA設計ではラッチの使用は避けるべきですが、特定のASIC設計やメモリ回路では必要となることがあります。

```systemverilog
// 意図的なラッチの記述
module latch_example (
    input  logic       en,
    input  logic [7:0] d,
    output logic [7:0] q
);

    always_latch begin
        if (en)
            q <= d;
        // enが0のときqは値を保持（ラッチの正しい動作）
    end

endmodule
```

`always_latch`は「このブロックはラッチを意図している」という設計者の意思表明です。もし`always_comb`で記述していれば警告が出るような不完全な代入を、`always_latch`では正しいラッチ動作として許容します。

### 5.2.5 使い分けのまとめ

| ブロック | 用途 | 代入文 | 感度リスト |
|----------|------|--------|------------|
| `always_comb` | 組み合わせ回路 | ブロッキング（`=`） | 自動 |
| `always_ff` | フリップフロップ | ノンブロッキング（`<=`） | クロックエッジ（手動） |
| `always_latch` | ラッチ | ノンブロッキング（`<=`） | 自動 |

これらの専用ブロックを使い分けることで、設計者の意図がコードから明確に読み取れるようになり、ツールによる静的チェックも強化されます。汎用の`always`ブロックは、新しいRTLコードでは使用すべきではありません。

![alwaysブロックの使い分け](/images/systemverilog-complete-guide/ch05_always_blocks.drawio.png)

---

## 5.3 代入文

### 5.3.1 ブロッキング代入（`=`）

ブロッキング代入は、手続き型ブロック内で文が上から順に逐次実行される代入方式です。前の代入が完了してから次の代入が実行されます。

```systemverilog
// ブロッキング代入の例
always_comb begin
    temp = a & b;      // まずtempが計算される
    y    = temp | c;   // 次にyがtempの新しい値を使って計算される
end
```

ブロッキング代入は**組み合わせ回路（`always_comb`）**で使用します。組み合わせ回路では中間変数を使って段階的に値を計算することが多いため、逐次実行のセマンティクスが自然に適合します。

### 5.3.2 ノンブロッキング代入（`<=`）

ノンブロッキング代入は、右辺の評価と左辺への代入が分離される代入方式です。ブロック内のすべての右辺がまず評価され、その後にすべての左辺への代入がまとめて行われます。

```systemverilog
// ノンブロッキング代入の例
always_ff @(posedge clk) begin
    q1 <= d;     // dの現在の値を記録
    q2 <= q1;    // q1の「更新前の」値を記録（新しい値ではない）
    q3 <= q2;    // q2の「更新前の」値を記録
end
```

上記の例は3段のシフトレジスタを実現しています。ノンブロッキング代入により、`q1`、`q2`、`q3`への代入は同時に行われるため、代入順序に依存しません。これが順序回路での正しい動作です。

### 5.3.3 混在使用の危険性

ブロッキング代入とノンブロッキング代入を同じ信号に対して異なるブロックで混在させることは、シミュレーションと合成の不一致を引き起こす最も一般的な原因のひとつです。

```systemverilog
// 危険な例：ブロッキング代入を順序回路で使用
// この記述はシフトレジスタにならない
always_ff @(posedge clk) begin
    q1 = d;     // q1にdが即座に代入される
    q2 = q1;    // q1の「新しい値」（= d）が使われる
    q3 = q2;    // q2の「新しい値」（= d）が使われる
end
// 結果：q1 = q2 = q3 = d となり、シフトレジスタではなく単なるレジスタになる
```

この問題を回避するための原則は明快です。

- **`always_comb`内ではブロッキング代入（`=`）を使用する**
- **`always_ff`内ではノンブロッキング代入（`<=`）を使用する**
- **同一信号に対して異なる種類の代入を混在させない**

### 5.3.4 具体的なコード例：代入方式による挙動の違い

以下に、代入方式の違いが回路の挙動にどう影響するかを具体的に示します。

```systemverilog
// 正しいシフトレジスタ（ノンブロッキング代入）
module shift_reg_correct (
    input  logic clk,
    input  logic d,
    output logic q1, q2, q3
);

    always_ff @(posedge clk) begin
        q1 <= d;
        q2 <= q1;
        q3 <= q2;
    end

endmodule
// クロックサイクルごとに d → q1 → q2 → q3 とデータがシフトする
```

```systemverilog
// 誤ったシフトレジスタ（ブロッキング代入 — アンチパターン）
module shift_reg_wrong (
    input  logic clk,
    input  logic d,
    output logic q1, q2, q3
);

    always_ff @(posedge clk) begin
        q1 = d;    // q1 = d
        q2 = q1;   // q2 = d（q1の新しい値）
        q3 = q2;   // q3 = d（q2の新しい値）
    end

endmodule
// 結果：すべての出力が同時にdの値になる。シフト動作しない。
```

![ブロッキング代入とノンブロッキング代入の違い](/images/systemverilog-complete-guide/ch05_blocking_vs_nonblocking.drawio.png)

以下に、組み合わせ回路での中間変数を用いた典型的なパターンを示します。

```systemverilog
// 組み合わせ回路での中間変数（ブロッキング代入が適切）
module alu_simple (
    input  logic [7:0]  a, b,
    input  logic [1:0]  op,
    output logic [7:0]  result,
    output logic        zero_flag
);

    logic [7:0] alu_out;

    always_comb begin
        case (op)
            2'b00:   alu_out = a + b;
            2'b01:   alu_out = a - b;
            2'b10:   alu_out = a & b;
            2'b11:   alu_out = a | b;
            default: alu_out = '0;
        endcase

        result    = alu_out;
        zero_flag = (alu_out == '0);
    end

endmodule
```

---

## 5.4 制御フロー

SystemVerilogでは、標準的な`if`-`else`文や`case`文に加えて、設計者の意図を明示するための修飾子として`unique`と`priority`が導入されました。これらは合成ツールに対する最適化ヒントであると同時に、シミュレーション時の動的チェック機能を持ちます。

### 5.4.1 unique if

`unique if`は、条件分岐が**排他的（mutual exclusive）**であることを宣言します。つまり、同時に複数の条件が真になることはないと設計者が保証します。

```systemverilog
// unique ifの使用例
always_comb begin
    unique if (sel == 2'b00)
        y = a;
    else if (sel == 2'b01)
        y = b;
    else if (sel == 2'b10)
        y = c;
    else if (sel == 2'b11)
        y = d;
end
```

`unique if`の効果は以下のとおりです。

- **合成ツールへの影響**：条件が排他的であるため、優先エンコーダではなく**並列マルチプレクサ**として合成できます。これにより回路の遅延が削減されます。
- **シミュレーション時の検出**：実行時にどの条件にも一致しない場合、または複数の条件が同時に真になる場合に、警告が発生します。

![並列マルチプレクサと優先エンコーダ](/images/systemverilog-complete-guide/ch05_parallel_vs_priority.drawio.png)

### 5.4.2 priority if

`priority if`は、条件分岐に**優先順位がある**ことを宣言します。上から順に条件が評価され、最初に真になった条件の処理が実行されます。

```systemverilog
// priority ifの使用例：割り込み優先度エンコーダ
always_comb begin
    priority if (irq[3])
        irq_id = 2'd3;  // 最高優先度
    else if (irq[2])
        irq_id = 2'd2;
    else if (irq[1])
        irq_id = 2'd1;
    else if (irq[0])
        irq_id = 2'd0;  // 最低優先度
end
```

`priority if`の効果は以下のとおりです。

- **合成ツールへの影響**：**優先エンコーダ**として合成されます。条件の評価順序が回路構造に反映されます。
- **シミュレーション時の検出**：どの条件にも一致しない場合に警告が発生します。ただし、複数の条件が同時に真になることは許容されます（優先順位で最初の条件が選ばれます）。

### 5.4.3 unique case / priority case

`case`文にも同様に`unique`と`priority`の修飾子を適用できます。

```systemverilog
// unique caseの使用例
always_comb begin
    unique case (state)
        IDLE:    next_state = RUN;
        RUN:     next_state = DONE;
        DONE:    next_state = IDLE;
        default: next_state = IDLE;
    endcase
end
```

```systemverilog
// priority caseの使用例
always_comb begin
    priority case (1'b1)
        req[3]: grant = 4'b1000;  // 最高優先度
        req[2]: grant = 4'b0100;
        req[1]: grant = 4'b0010;
        req[0]: grant = 4'b0001;  // 最低優先度
        default: grant = 4'b0000;
    endcase
end
```

`unique case`は各ケース値が排他的であることを保証します。シミュレーション時に、一致するケースがない場合に警告が出ます。`priority case`は上から順に評価され、最初に一致したケースが選択されます。

### 5.4.4 unique0とpriority0

SystemVerilog-2009では、`unique0`と`priority0`が追加されました。これらは`unique`/`priority`と基本的に同じ動作をしますが、**どの条件にも一致しない場合に警告を発生させない**という違いがあります。

```systemverilog
// unique0 if — どの条件にも一致しなくても警告なし
always_comb begin
    y = '0;  // デフォルト値
    unique0 if (sel == 2'b00)
        y = a;
    else if (sel == 2'b01)
        y = b;
    else if (sel == 2'b10)
        y = c;
    // sel == 2'b11 のとき、yはデフォルト値のまま（警告なし）
end
```

| 修飾子 | 条件の排他性 | 優先順位 | 不一致時の警告 |
|--------|-------------|----------|----------------|
| `unique` | 排他的であることを要求 | なし（並列） | 警告あり |
| `unique0` | 排他的であることを要求 | なし（並列） | 警告なし |
| `priority` | 排他的でなくてよい | あり（順序） | 警告あり |
| `priority0` | 排他的でなくてよい | あり（順序） | 警告なし |

### 5.4.5 合成ツールへの影響

`unique`と`priority`の選択は、合成結果の回路構造に直接影響します。

**unique（並列マルチプレクサ）**：すべての条件が同時に評価され、該当する1つの出力が選択されます。遅延は条件の数に関係なく一定であり、高速な回路が得られます。ただし、条件が実際に排他的でなければ、動作が未定義になります。

**priority（優先エンコーダ）**：条件が上から順に評価されるカスケード構造となります。条件が多いほど遅延が増加しますが、条件の重複を許容できるため柔軟性があります。

```systemverilog
// 実践例：アドレスデコーダ（unique caseが適切）
module address_decoder (
    input  logic [15:0] addr,
    output logic        cs_rom,
    output logic        cs_ram,
    output logic        cs_io
);

    always_comb begin
        cs_rom = 1'b0;
        cs_ram = 1'b0;
        cs_io  = 1'b0;

        unique case (addr[15:12])
            4'h0, 4'h1, 4'h2, 4'h3: cs_rom = 1'b1;  // 0x0000-0x3FFF: ROM
            4'h4, 4'h5, 4'h6, 4'h7,
            4'h8, 4'h9, 4'hA, 4'hB: cs_ram = 1'b1;  // 0x4000-0xBFFF: RAM
            4'hC, 4'hD, 4'hE, 4'hF: cs_io  = 1'b1;  // 0xC000-0xFFFF: I/O
            default: ;  // すべて非選択のまま
        endcase
    end

endmodule
```

```systemverilog
// 実践例：割り込みコントローラ（priority ifが適切）
module interrupt_controller (
    input  logic [7:0] irq,
    output logic [2:0] irq_id,
    output logic       irq_valid
);

    always_comb begin
        irq_valid = 1'b1;

        priority if (irq[7])      irq_id = 3'd7;
        else if (irq[6])          irq_id = 3'd6;
        else if (irq[5])          irq_id = 3'd5;
        else if (irq[4])          irq_id = 3'd4;
        else if (irq[3])          irq_id = 3'd3;
        else if (irq[2])          irq_id = 3'd2;
        else if (irq[1])          irq_id = 3'd1;
        else if (irq[0])          irq_id = 3'd0;
        else begin
            irq_id    = 3'd0;
            irq_valid = 1'b0;
        end
    end

endmodule
```

---

## 5.5 総合的なコード例

ここまで学んだ内容を統合した実践的なコード例を示します。以下は、簡単なプロセッサのデータパスの一部を模したモジュールです。

```systemverilog
module datapath_example (
    input  logic        clk,
    input  logic        rst_n,
    input  logic [7:0]  data_in,
    input  logic [1:0]  alu_op,
    input  logic        load_en,
    input  logic        store_en,
    output logic [7:0]  data_out,
    output logic        zero
);

    logic [7:0] reg_a, reg_b;
    logic [7:0] alu_result;

    // 組み合わせ回路：ALU（always_comb + ブロッキング代入）
    always_comb begin
        unique case (alu_op)
            2'b00:   alu_result = reg_a + reg_b;
            2'b01:   alu_result = reg_a - reg_b;
            2'b10:   alu_result = reg_a & reg_b;
            2'b11:   alu_result = reg_a | reg_b;
            default: alu_result = '0;
        endcase
        zero = (alu_result == '0);
    end

    // 順序回路：レジスタファイル（always_ff + ノンブロッキング代入）
    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            reg_a <= 8'h00;
            reg_b <= 8'h00;
        end else begin
            if (load_en)
                reg_a <= data_in;
            if (store_en)
                reg_b <= data_in;
        end
    end

    // 出力の接続
    assign data_out = alu_result;

endmodule
```

このモジュールでは以下のポイントが実践されています。

- ALUは`always_comb`と`unique case`で記述し、組み合わせ回路であることを明示しています
- レジスタは`always_ff`とノンブロッキング代入で記述し、順序回路であることを明示しています
- `unique case`により、合成ツールがALUを並列マルチプレクサとして最適化できます
- すべてのケースでデフォルト値が設定されており、意図しないラッチが生成されません

---

## 5.6 まとめ

本章では、SystemVerilogにおける組み合わせ回路と順序回路の記述方法について学びました。重要なポイントを以下に整理します。

**alwaysブロックの使い分け**

- `always_comb`は組み合わせ回路専用です。自動感度リスト、ラッチ推論の防止、時間0での初期実行が特長です。
- `always_ff`は順序回路（フリップフロップ）専用です。クロックエッジと非同期リセットを感度リストに記述します。
- `always_latch`はラッチ専用です。意図的なラッチ設計であることを明示します。
- 汎用の`always`ブロックは新しいRTLコードでは使用しません。

**代入文のルール**

- `always_comb`ではブロッキング代入（`=`）を使用します。
- `always_ff`ではノンブロッキング代入（`<=`）を使用します。
- 両者の混在は、シミュレーションと合成結果の不一致を招く危険があるため、厳禁です。

**制御フロー修飾子**

- `unique if` / `unique case`は、条件が排他的であることを宣言し、並列マルチプレクサとしての合成を可能にします。
- `priority if` / `priority case`は、条件に優先順位があることを宣言し、優先エンコーダとして合成されます。
- `unique0` / `priority0`は、どの条件にも一致しない場合の警告を抑制するバリアントです。
- これらの修飾子により、設計意図がコードに明記され、ツールによるチェックが強化されます。

これらのルールを一貫して守ることで、読みやすく保守しやすいRTLコードが書けるようになり、シミュレーションと合成の一致が保証されたデジタル設計を実現できます。
