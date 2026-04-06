---
title: "第10章：制約付きランダム検証（CRV）"
---

# 第10章：制約付きランダム検証（CRV）

## 10.1 この章で学ぶこと

本章では、SystemVerilogの最も強力な検証機能の一つである**制約付きランダム検証（Constrained Random Verification, CRV）**について解説します。手動でテストパターンを記述する代わりに、制約条件を満たすランダムな刺激を自動生成することで、人間が思いつかないようなコーナーケースを効率的に検出できます。`rand`/`randc`修飾子、制約ブロック（`constraint`）、インライン制約（`randomize() with`）の使い方を体系的に学びます。

---

## 10.2 ランダム変数の宣言

### 10.2.1 rand 修飾子

`rand` を付けたプロパティは、`randomize()` メソッドが呼ばれるたびにランダムな値が割り当てられます。制約がなければ、値の範囲内で均一な分布を持ちます。

```systemverilog
class Packet;
    rand bit [7:0]  src_addr;
    rand bit [7:0]  dst_addr;
    rand bit [31:0] data;
    rand bit [2:0]  pkt_type;
         int        pkt_id;       // randなし → ランダム化されない

    function new(int id);
        this.pkt_id = id;
    endfunction

    function void display();
        $display("Packet[%0d]: src=0x%h, dst=0x%h, type=%0d, data=0x%h",
                 pkt_id, src_addr, dst_addr, pkt_type, data);
    endfunction
endclass

module test;
    initial begin
        Packet pkt = new(1);

        repeat (5) begin
            if (pkt.randomize())
                pkt.display();
            else
                $error("Randomization failed!");
        end
    end
endmodule
```

### 10.2.2 randc 修飾子

`randc`（random cyclic）は**巡回ランダム**です。すべての値を一巡するまで同じ値は繰り返されません。カードのシャッフルに例えると分かりやすいでしょう。

```systemverilog
class DeckOfCards;
    randc bit [5:0] card;  // 0〜63の値（最大64種類）

    constraint valid_card {
        card inside {[0:51]};  // 52枚のカードのみ
    }
endclass

module test;
    initial begin
        DeckOfCards deck = new();

        $display("--- Dealing cards ---");
        repeat (10) begin
            if (deck.randomize())
                $display("Card: %0d", deck.card);
        end
        // 10回のランダム化で同じ値は出ない（52枚以内）
    end
endmodule
```

### 10.2.3 rand と randc の比較

| 特性 | `rand` | `randc` |
|------|--------|---------|
| 値の分布 | 均一分布（重複あり） | 巡回（全値を一巡するまで重複なし） |
| 適用場面 | 一般的なランダム変数 | ラウンドロビン、全パターン網羅 |
| パフォーマンス | 高速 | 値域が大きいと遅くなる場合がある |
| 値域の推奨サイズ | 制限なし | 小さい値域に推奨 |

---

## 10.3 制約ブロック（constraint）

### 10.3.1 基本的な制約

`constraint` ブロックは、ランダム変数に対する条件を宣言的に記述します。制約ソルバが条件を満たす値を自動的に見つけ出します。

```systemverilog
class BusTransaction;
    rand bit [31:0] addr;
    rand bit [31:0] data;
    rand bit [3:0]  burst_len;
    rand bit        write;

    // アドレスの制約
    constraint addr_range {
        addr >= 32'h0000_0000;
        addr <  32'h1000_0000;    // 256MBの範囲内
        addr[1:0] == 2'b00;       // 4バイトアラインメント
    }

    // バースト長の制約
    constraint burst_constraint {
        burst_len inside {1, 2, 4, 8};  // 1, 2, 4, 8のいずれか
    }

endclass
```

### 10.3.2 inside 演算子

`inside` は、値が指定されたリストまたは範囲内にあることを制約します。

```systemverilog
class Config;
    rand int          timeout;
    rand bit [7:0]    channel;
    rand bit [15:0]   size;

    // リストからの選択
    constraint timeout_values {
        timeout inside {100, 200, 500, 1000, 5000};
    }

    // 範囲指定
    constraint channel_range {
        channel inside {[0:15]};   // 0〜15の範囲
    }

    // 除外（否定）
    constraint size_valid {
        size inside {[64:4096]};
        !(size inside {[1000:1100]});  // 1000〜1100を除外
    }
endclass
```

### 10.3.3 dist（分布制約）

`dist` は、値ごとに重み（確率）を指定できます。`:=` は各要素に同じ重みを、`:/` は範囲全体で重みを分配します。

```systemverilog
class ErrorInjector;
    rand bit [1:0] error_type;
    rand int       delay;

    // := 各値に同じ重み
    constraint error_dist {
        error_type dist {
            2'b00 := 70,   // エラーなし: 重み70
            2'b01 := 20,   // 軽微なエラー: 重み20
            2'b10 := 8,    // 重大なエラー: 重み8
            2'b11 := 2     // 致命的エラー: 重み2
        };
    }

    // :/ 範囲に対して重みを分配
    constraint delay_dist {
        delay dist {
            [1:10]   :/ 50,     // 1〜10に重み50を分配（各5）
            [11:100] :/ 30,     // 11〜100に重み30を分配（各約0.33）
            [101:1000] :/ 20    // 101〜1000に重み20を分配
        };
    }
endclass
```

`:=` と `:/` の違いを図で示します：

![dist演算子の:=と:/の違い](/images/systemverilog-complete-guide/ch10_dist_operator.drawio.png)

### 10.3.4 solve...before

`solve...before` は制約ソルバに対して、変数の解決順序を指示します。ある変数の値が他の変数の制約に影響する場合に使用します。

```systemverilog
class ConditionalPacket;
    rand bit        is_long;
    rand bit [15:0] length;

    constraint length_c {
        if (is_long)
            length inside {[256:1500]};
        else
            length inside {[64:255]};

        // is_longを先に決定してからlengthを解決
        solve is_long before length;
    }
endclass
```

`solve...before` がない場合、ソルバは `is_long` と `length` を同時に解こうとし、確率分布が偏る可能性があります。`solve is_long before length` を指定すると、まず `is_long` が50%/50%で決まり、その後 `length` が対応する範囲でランダム化されます。

### 10.3.5 条件付き制約（Implication）

`->` 演算子（含意）や `if-else` を使って条件付きの制約を記述できます。

```systemverilog
class ProtocolPacket;
    rand bit [3:0]  cmd;
    rand bit [31:0] addr;
    rand bit [31:0] data;
    rand bit [7:0]  length;

    // 含意演算子（->）
    constraint read_cmd {
        (cmd == 4'h1) -> (data == 32'h0);  // READコマンドならdataは0
    }

    // if-else による条件付き制約
    constraint cmd_specific {
        if (cmd == 4'h2) {   // WRITEコマンド
            length inside {[1:64]};
            addr[1:0] == 2'b00;
        } else if (cmd == 4'h3) {  // BURSTコマンド
            length inside {4, 8, 16, 32};
            addr[3:0] == 4'b0000;  // 16バイトアラインメント
        } else {
            length == 1;
        }
    }
endclass
```

---

## 10.4 制約の制御

### 10.4.1 制約の有効/無効

`constraint_mode()` メソッドで制約の有効/無効を動的に切り替えられます。

```systemverilog
class FlexibleTransaction;
    rand bit [31:0] addr;
    rand bit [31:0] data;

    constraint normal_range {
        addr inside {[32'h0000:32'hFFFF]};
    }

    constraint high_range {
        addr inside {[32'hFFFF_0000:32'hFFFF_FFFF]};
    }
endclass

module test;
    initial begin
        FlexibleTransaction txn = new();

        // 通常範囲のテスト
        txn.high_range.constraint_mode(0);  // high_rangeを無効化
        txn.randomize();

        // 高アドレス範囲のテスト
        txn.normal_range.constraint_mode(0);
        txn.high_range.constraint_mode(1);
        txn.randomize();

        // すべての制約を無効化
        txn.constraint_mode(0);  // オブジェクトの全制約を無効化
        txn.randomize();         // 完全にランダム
    end
endmodule
```

### 10.4.2 ランダム変数の有効/無効

`rand_mode()` メソッドで個別のランダム変数のランダム化を制御できます。

```systemverilog
module test;
    initial begin
        FlexibleTransaction txn = new();

        txn.addr = 32'h0000_1000;       // addrを固定値に設定
        txn.addr.rand_mode(0);           // addrのランダム化を無効化
        txn.randomize();                  // dataのみランダム化される
    end
endmodule
```

---

## 10.5 インライン制約（randomize() with）

`randomize() with { ... }` を使うと、クラス定義を変更せずに追加の制約を指定できます。テストシナリオごとに異なる制約を適用する場合に非常に便利です。

```systemverilog
class Packet;
    rand bit [7:0]  src;
    rand bit [7:0]  dst;
    rand bit [31:0] data;
    rand bit [3:0]  priority_level;

    constraint base_c {
        src != dst;  // 送信元と宛先は異なること
    }

    function void display();
        $display("Packet: src=0x%h, dst=0x%h, data=0x%h, priority=%0d",
                 src, dst, data, priority_level);
    endfunction
endclass

module test;
    initial begin
        Packet pkt = new();

        // テスト1: ブロードキャストパケット
        assert(pkt.randomize() with {
            dst == 8'hFF;
            priority_level == 4'hF;
        }) else $fatal("Randomization failed");

        // テスト2: 低優先度の小さなパケット
        assert(pkt.randomize() with {
            priority_level inside {[0:3]};
            data < 32'h0000_FFFF;
        }) else $fatal("Randomization failed");

        // テスト3: 特定アドレス間の通信
        assert(pkt.randomize() with {
            src == 8'h01;
            dst inside {8'h10, 8'h20, 8'h30};
        }) else $fatal("Randomization failed");

        pkt.display();
    end
endmodule
```

---

## 10.6 pre_randomize と post_randomize

`randomize()` の前後に自動的に呼ばれるコールバックメソッドを定義できます。

```systemverilog
class SmartPacket;
    rand bit [7:0]  src;
    rand bit [7:0]  dst;
    rand bit [31:0] data;
         bit [7:0]  checksum;   // randではない（計算で決まる）
    static int      gen_count = 0;

    constraint valid_c {
        src != dst;
    }

    // ランダム化前に呼ばれる
    function void pre_randomize();
        $display("[pre_randomize] Preparing packet generation #%0d", gen_count + 1);
    endfunction

    // ランダム化後に呼ばれる
    function void post_randomize();
        checksum = src ^ dst ^ data[7:0] ^ data[15:8] ^ data[23:16] ^ data[31:24];
        gen_count++;
        $display("[post_randomize] Checksum = 0x%h", checksum);
    endfunction

    function void display();
        $display("Packet: src=0x%h, dst=0x%h, data=0x%h, checksum=0x%h",
                 src, dst, data, checksum);
    endfunction
endclass
```

---

## 10.7 ランダム化の実践例

### 10.7.1 メモリアクセスパターンの生成

```systemverilog
class MemoryAccess;
    rand bit [31:0] addr;
    rand bit [31:0] data;
    rand bit [1:0]  access_type;  // 0:byte, 1:halfword, 2:word
    rand bit        write;

    // アドレスアラインメント制約
    constraint alignment {
        (access_type == 2'b01) -> (addr[0] == 1'b0);      // ハーフワードは2バイト境界
        (access_type == 2'b10) -> (addr[1:0] == 2'b00);   // ワードは4バイト境界
    }

    // メモリマップ制約
    constraint mem_map {
        addr inside {
            [32'h0000_0000:32'h0000_FFFF],   // ROM領域
            [32'h2000_0000:32'h2000_FFFF],   // RAM領域
            [32'h4000_0000:32'h4000_00FF]    // ペリフェラル領域
        };
    }

    // ROM領域は読み出しのみ
    constraint rom_read_only {
        (addr inside {[32'h0000_0000:32'h0000_FFFF]}) -> (write == 1'b0);
    }
endclass
```

### 10.7.2 プロトコルシーケンスの生成

```systemverilog
class UartConfig;
    rand int       baud_rate;
    rand bit [1:0] parity;      // 0:none, 1:odd, 2:even
    rand bit       stop_bits;   // 0:1bit, 1:2bits
    rand bit [1:0] data_bits;   // 0:5bits, 1:6bits, 2:7bits, 3:8bits

    constraint valid_baud {
        baud_rate inside {9600, 19200, 38400, 57600, 115200, 230400, 460800};
    }

    constraint common_config {
        data_bits dist {
            2'b11 := 80,    // 8ビットが最も一般的
            2'b10 := 15,    // 7ビット
            2'b01 := 3,     // 6ビット
            2'b00 := 2      // 5ビット
        };
    }

    constraint parity_rule {
        // 5ビットモードではパリティなしのみ
        (data_bits == 2'b00) -> (parity == 2'b00);
    }

    function void display();
        string parity_str;
        case (parity)
            2'b00: parity_str = "None";
            2'b01: parity_str = "Odd";
            2'b10: parity_str = "Even";
            default: parity_str = "Unknown";
        endcase
        $display("UART Config: %0d baud, %0d data bits, %s parity, %0d stop bits",
                 baud_rate, data_bits + 5, parity_str, stop_bits + 1);
    endfunction
endclass
```

---

## 10.8 まとめ

本章では、SystemVerilogの制約付きランダム検証（CRV）について学びました。

1. **`rand`** は均一分布のランダム変数、**`randc`** は巡回（重複なし）のランダム変数を宣言します。
2. **`constraint` ブロック**で制約条件を宣言的に記述します。`inside`、`dist`、`solve...before`、条件付き制約（`->`/`if-else`）を使い分けます。
3. **`dist` 演算子**で値の出現確率を制御できます。`:=` は各値に同じ重み、`:/` は範囲に重みを分配します。
4. **`solve...before`** で変数の解決順序を制御し、確率分布の偏りを防ぎます。
5. **インライン制約（`randomize() with`）**でテストシナリオごとに柔軟な制約を追加できます。
6. **`constraint_mode()`** と **`rand_mode()`** で制約やランダム変数の有効/無効を動的に制御できます。
7. **`pre_randomize()` / `post_randomize()`** でランダム化前後のカスタム処理が可能です。

次章では、ランダム検証を支えるもう一つの重要な機能——プロセス制御と同期について学びます。
