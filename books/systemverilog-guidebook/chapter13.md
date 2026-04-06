---
title: "第13章：機能カバレッジ"
---

# 第13章：機能カバレッジ

## 13.1 この章で学ぶこと

本章では、SystemVerilogの**機能カバレッジ（Functional Coverage）**について解説します。ランダム検証では膨大な数のテストが自動生成されますが、「テストが十分に行われたか」「どのシナリオがまだ検証されていないか」を把握するのは容易ではありません。機能カバレッジは、検証の**網羅性を数値化**し、検証のゴールを明確にする仕組みです。`covergroup`、`coverpoint`、`cross` の使い方と、カバレッジ目標の設定方法を学びます。

---

## 13.2 機能カバレッジとは

### 13.2.1 カバレッジの種類

検証で使われるカバレッジは主に2種類あります。

| カバレッジ種類 | 説明 | 自動/手動 |
|-------------|------|----------|
| **コードカバレッジ** | RTLコードの実行率（行、分岐、条件、トグル） | ツールが自動収集 |
| **機能カバレッジ** | 仕様上の検証ポイントの網羅率 | エンジニアが定義 |

コードカバレッジが100%でも、仕様上テストすべきシナリオが抜けている可能性があります。機能カバレッジはエンジニアが「何を検証すべきか」を明示的に定義し、その達成率を追跡します。

### 13.2.2 機能カバレッジの流れ

1. **カバレッジモデルの定義**: `covergroup` で検証対象のシナリオを記述
2. **シミュレーションの実行**: テスト実行中にカバレッジ情報が自動収集される
3. **カバレッジの分析**: 達成率を確認し、未到達のシナリオを特定
4. **テストの追加**: カバレッジ穴（coverage hole）を埋めるテストを追加
5. **目標達成**: すべてのカバレッジが目標値に到達したら検証完了

![機能カバレッジの検証フロー](/images/systemverilog-complete-guide/ch13_coverage_flow.drawio.png)

---

## 13.3 covergroup の基本

### 13.3.1 covergroup の定義とインスタンス化

`covergroup` は、カバレッジ収集の対象と条件を定義するブロックです。

```systemverilog
class BusMonitor;
    bit [31:0] addr;
    bit [31:0] data;
    bit        write;
    bit [2:0]  burst_type;

    // カバレッジグループの定義
    covergroup bus_cov @(posedge clk);
        // アドレス範囲のカバレッジ
        addr_cp: coverpoint addr {
            bins low_addr   = {[32'h0000_0000:32'h0000_FFFF]};
            bins mid_addr   = {[32'h0001_0000:32'h000F_FFFF]};
            bins high_addr  = {[32'h0010_0000:32'h00FF_FFFF]};
            bins io_region  = {[32'hFF00_0000:32'hFFFF_FFFF]};
        }

        // リード/ライトのカバレッジ
        rw_cp: coverpoint write {
            bins read  = {0};
            bins write = {1};
        }

        // バーストタイプのカバレッジ
        burst_cp: coverpoint burst_type {
            bins single = {3'b000};
            bins incr   = {3'b001};
            bins wrap   = {3'b010};
            bins incr4  = {3'b011};
        }
    endgroup

    // コンストラクタでカバレッジグループをインスタンス化
    function new();
        bus_cov = new();
    endfunction
endclass
```

### 13.3.2 サンプリングのタイミング

カバレッジデータは、指定されたタイミングで「サンプリング」されます。サンプリングの方法は以下のとおりです。

```systemverilog
// 方法1: イベントベース（定義時に指定）
covergroup cov_event @(posedge clk);
    // clkの立ち上がりでサンプリング
endgroup

// 方法2: 明示的な sample() 呼び出し
covergroup cov_manual;
    // ...
endgroup

initial begin
    cov_manual cov = new();
    repeat (100) begin
        @(posedge clk);
        if (valid)
            cov.sample();  // 明示的にサンプリング
    end
end

// 方法3: with function sample（引数付き）
covergroup cov_with_args with function sample(bit [7:0] data, bit wr);
    data_cp: coverpoint data;
    wr_cp: coverpoint wr;
endgroup

initial begin
    cov_with_args cov = new();
    // ...
    cov.sample(captured_data, captured_wr);
end
```

---

## 13.4 coverpoint の詳細

### 13.4.1 自動ビン（Auto Bins）

`coverpoint` に明示的なビン定義がない場合、ツールが自動的にビンを生成します。

```systemverilog
covergroup auto_cov;
    // 自動ビン: bit [2:0] なら 0〜7 の8つのビンが自動生成
    cmd_cp: coverpoint cmd_type;

    // 自動ビンの数を制限
    addr_cp: coverpoint addr {
        option.auto_bin_max = 16;  // 最大16ビンに分割
    }
endgroup
```

### 13.4.2 明示的なビン定義

実際のプロジェクトでは、意味のあるビンを明示的に定義することが推奨されます。

```systemverilog
covergroup detailed_cov @(posedge clk);
    // 個別の値
    opcode_cp: coverpoint opcode {
        bins add   = {4'b0000};
        bins sub   = {4'b0001};
        bins and_  = {4'b0010};
        bins or_   = {4'b0011};
        bins xor_  = {4'b0100};
        bins nop   = {4'b1111};
        bins other = default;    // 上記以外すべて
    }

    // 範囲のビン
    size_cp: coverpoint packet_size {
        bins small   = {[1:64]};
        bins medium  = {[65:256]};
        bins large   = {[257:1024]};
        bins jumbo   = {[1025:9000]};
    }

    // 遷移ビン（値の変化パターン）
    state_cp: coverpoint state {
        bins idle_to_active = (IDLE => ACTIVE);
        bins active_to_done = (ACTIVE => DONE);
        bins done_to_idle   = (DONE => IDLE);
        bins full_cycle     = (IDLE => ACTIVE => DONE => IDLE);
    }

    // 除外ビン（カバレッジ計算から除外）
    reserved_cp: coverpoint cmd {
        bins valid_cmds[] = {[0:10]};
        ignore_bins reserved = {[11:14]};  // 無視
        illegal_bins bad_cmd = {15};       // 発生したらエラー
    }
endgroup
```

### 13.4.3 ビンの配列

`[]` を使うと、各値に個別のビンが自動的に作成されます。

```systemverilog
covergroup array_bins_cov;
    // 各値ごとに個別のビンを生成
    port_cp: coverpoint port_id {
        bins ports[] = {[0:7]};  // port_0, port_1, ..., port_7 の8ビン
    }

    // 範囲を均等分割
    addr_cp: coverpoint addr {
        bins ranges[4] = {[0:255]};  // 4つのビンに均等分割
        // ranges[0] = {[0:63]}, ranges[1] = {[64:127]}, ...
    }
endgroup
```

---

## 13.5 クロスカバレッジ（cross）

### 13.5.1 cross の基本

`cross` は複数の `coverpoint` の組み合わせ（直積）をカバレッジとして定義します。個々の `coverpoint` だけでは検出できない、条件の組み合わせの網羅性を確認できます。

```systemverilog
covergroup cross_cov @(posedge clk);
    cmd_cp: coverpoint cmd {
        bins read  = {0};
        bins write = {1};
    }

    size_cp: coverpoint size {
        bins small  = {[1:64]};
        bins medium = {[65:256]};
        bins large  = {[257:1024]};
    }

    priority_cp: coverpoint priority {
        bins low  = {[0:3]};
        bins high = {[4:7]};
    }

    // cmd × size のクロスカバレッジ
    cmd_size_cross: cross cmd_cp, size_cp;
    // 生成されるビン:
    //   <read, small>, <read, medium>, <read, large>,
    //   <write, small>, <write, medium>, <write, large>

    // cmd × size × priority の3次元クロス
    full_cross: cross cmd_cp, size_cp, priority_cp;
    // 2 × 3 × 2 = 12 ビンが生成される
endgroup
```

### 13.5.2 クロスカバレッジのフィルタリング

すべての組み合わせが意味を持つわけではありません。`binsof` と `ignore_bins` でフィルタリングできます。

```systemverilog
covergroup filtered_cross_cov @(posedge clk);
    cmd_cp: coverpoint cmd {
        bins read  = {0};
        bins write = {1};
        bins burst = {2};
    }

    addr_cp: coverpoint addr_region {
        bins rom    = {0};
        bins ram    = {1};
        bins io     = {2};
    }

    // クロスカバレッジ（一部の組み合わせを除外）
    cmd_addr_cross: cross cmd_cp, addr_cp {
        // ROM領域への書き込みとバーストは意味がないので除外
        ignore_bins no_rom_write = binsof(cmd_cp.write) && binsof(addr_cp.rom);
        ignore_bins no_rom_burst = binsof(cmd_cp.burst) && binsof(addr_cp.rom);
    }
endgroup
```

---

## 13.6 カバレッジオプション

### 13.6.1 インスタンスオプション

```systemverilog
covergroup configurable_cov @(posedge clk);
    option.per_instance = 1;       // インスタンスごとにカバレッジを独立管理
    option.name = "bus_coverage";  // カバレッジ名
    option.comment = "Bus protocol coverage model";

    addr_cp: coverpoint addr {
        option.at_least = 10;      // 各ビンが最低10回ヒットで達成
        bins ranges[8] = {[0:32'hFFFF_FFFF]};
    }

    cmd_cp: coverpoint cmd {
        option.at_least = 5;       // 各ビンが最低5回ヒットで達成
        option.weight = 2;         // このcoverpointの重みを2倍に
        bins cmds[] = {[0:7]};
    }
endgroup
```

### 13.6.2 主なオプション一覧

| オプション | レベル | デフォルト | 説明 |
|-----------|--------|-----------|------|
| `option.per_instance` | covergroup | 0 | インスタンスごとにカバレッジを分離 |
| `option.at_least` | coverpoint/cross | 1 | ビンが「ヒット」と見なされる最小回数 |
| `option.weight` | coverpoint/cross | 1 | カバレッジ計算における重み |
| `option.goal` | covergroup | 100 | カバレッジ目標（%） |
| `option.auto_bin_max` | coverpoint | 64 | 自動生成ビンの最大数 |
| `option.comment` | 全レベル | "" | コメント |

### 13.6.3 type_option

`type_option` は、同一型のすべてのインスタンスに共通するオプションを設定します。

```systemverilog
covergroup shared_cov;
    type_option.merge_instances = 1;  // 全インスタンスのカバレッジをマージ
    type_option.weight = 1;
    type_option.goal = 95;            // 目標95%

    data_cp: coverpoint data {
        bins values[] = {[0:15]};
    }
endgroup
```

---

## 13.7 カバレッジの取得と分析

### 13.7.1 組み込みメソッド

```systemverilog
module coverage_analysis;
    covergroup cov @(posedge clk);
        addr_cp: coverpoint addr {
            bins ranges[4] = {[0:255]};
        }
        cmd_cp: coverpoint cmd {
            bins cmds[] = {[0:3]};
        }
    endgroup

    cov cg = new();

    // シミュレーション終了時にカバレッジを表示
    final begin
        real total_cov, addr_cov, cmd_cov;

        // カバレッジグループ全体の達成率
        total_cov = cg.get_coverage();
        $display("Total Coverage: %.1f%%", total_cov);

        // 個別coverpoint の達成率
        addr_cov = cg.addr_cp.get_coverage();
        cmd_cov  = cg.cmd_cp.get_coverage();
        $display("  Address Coverage: %.1f%%", addr_cov);
        $display("  Command Coverage: %.1f%%", cmd_cov);

        // 型全体のカバレッジ（全インスタンスのマージ）
        $display("Type Coverage: %.1f%%", cov::get_coverage());
    end
endmodule
```

### 13.7.2 カバレッジレポートの活用

多くのシミュレータは、カバレッジデータベースをGUIで確認する機能を提供しています。

```bash
# VCSの場合
simv -cm line+cond+fsm+tgl+branch+assert
urg -dir simv.vdb -report coverage_report

# Questaの場合
vsim -coverage
coverage report -detail -file coverage_report.txt
```

---

## 13.8 カバレッジ駆動検証の実践

### 13.8.1 完全な検証環境での機能カバレッジ

```systemverilog
class SpiTransaction;
    rand bit [7:0]  tx_data;
    rand bit [1:0]  mode;      // SPI mode (0-3)
    rand bit [2:0]  clk_div;   // クロック分周
    rand bit        cs_polarity;
endclass

class SpiCoverage;
    SpiTransaction txn;

    covergroup spi_cov with function sample(SpiTransaction t);
        // SPI モードのカバレッジ
        mode_cp: coverpoint t.mode {
            bins mode0 = {0};
            bins mode1 = {1};
            bins mode2 = {2};
            bins mode3 = {3};
        }

        // 送信データのカバレッジ
        data_cp: coverpoint t.tx_data {
            bins zero      = {8'h00};
            bins all_ones  = {8'hFF};
            bins low_vals  = {[8'h01:8'h7F]};
            bins high_vals = {[8'h80:8'hFE]};
        }

        // クロック分周のカバレッジ
        clkdiv_cp: coverpoint t.clk_div {
            bins dividers[] = {[0:7]};
        }

        // モード × データパターンのクロス
        mode_data_cross: cross mode_cp, data_cp;

        // モード × クロック分周のクロス
        mode_clk_cross: cross mode_cp, clkdiv_cp;
    endgroup

    function new();
        spi_cov = new();
    endfunction

    function void sample(SpiTransaction t);
        spi_cov.sample(t);
    endfunction

    function void report();
        $display("=== SPI Coverage Report ===");
        $display("  Overall: %.1f%%", spi_cov.get_coverage());
        $display("  Mode:    %.1f%%", spi_cov.mode_cp.get_coverage());
        $display("  Data:    %.1f%%", spi_cov.data_cp.get_coverage());
        $display("  ClkDiv:  %.1f%%", spi_cov.clkdiv_cp.get_coverage());
    endfunction
endclass
```

### 13.8.2 カバレッジ駆動のテスト制御

カバレッジが目標に達するまでテストを自動的に継続する手法です。

```systemverilog
class CoverageDrivenTest;
    SpiTransaction txn;
    SpiCoverage    cov;
    mailbox #(SpiTransaction) gen2drv;
    real coverage_goal = 95.0;

    function new();
        txn = new();
        cov = new();
        gen2drv = new();
    endfunction

    task run();
        int iteration = 0;
        real current_cov;

        // カバレッジが目標に達するまでテストを継続
        do begin
            txn = new();
            assert(txn.randomize());
            gen2drv.put(txn);
            cov.sample(txn);
            iteration++;

            if (iteration % 100 == 0) begin
                current_cov = cov.spi_cov.get_coverage();
                $display("[Iter %0d] Coverage: %.1f%%", iteration, current_cov);
            end

            current_cov = cov.spi_cov.get_coverage();
        end while (current_cov < coverage_goal && iteration < 100000);

        $display("=== Test Complete ===");
        $display("Iterations: %0d", iteration);
        cov.report();
    endtask
endclass
```

---

## 13.9 まとめ

本章では、SystemVerilogの機能カバレッジについて学びました。

1. **機能カバレッジ**はエンジニアが定義する検証の網羅性指標です。コードカバレッジとは補完的な関係にあります。
2. **`covergroup`** でカバレッジモデルを定義し、`sample()` またはイベントでサンプリングします。
3. **`coverpoint`** で個別の変数のカバレッジを定義します。ビン（`bins`）で値を分類し、遷移ビンで状態遷移も検証できます。
4. **`cross`** で複数 `coverpoint` の組み合わせカバレッジを定義します。`ignore_bins` で不要な組み合わせを除外します。
5. **`option.at_least`** で各ビンの最小ヒット数、**`option.goal`** でカバレッジ目標を設定できます。
6. **カバレッジ駆動検証**により、カバレッジが目標に達するまで自動的にテストを継続する手法が実現できます。

次章からは第5部「高度なシステム連携編」に入り、C/C++との連携を可能にするDPI-Cについて学びます。
