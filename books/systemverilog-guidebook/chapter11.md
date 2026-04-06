---
title: "第11章：プロセス制御と同期"
---

# 第11章：プロセス制御と同期

## 11.1 この章で学ぶこと

本章では、SystemVerilogにおける**並列実行（fork-join）**と**同期プリミティブ（event, semaphore, mailbox）**について解説します。検証環境では、複数のプロセスが同時に動作し互いに連携する必要があります。ドライバがデータを送信しながらモニタが監視し、スコアボードが結果を比較する——こうした並列処理と同期の仕組みは、高品質なテストベンチの構築に不可欠です。

---

## 11.2 並列実行（fork-join）

### 11.2.1 fork-join の3つのバリエーション

SystemVerilogの `fork` ブロックは、内部の各文を独立したプロセス（スレッド）として並列に実行します。終了条件に応じて3つのバリエーションがあります。

#### fork-join：全プロセスの完了を待つ

```systemverilog
initial begin
    $display("[%0t] Start", $time);
    fork
        begin  // プロセス1
            #10;
            $display("[%0t] Process 1 done", $time);
        end
        begin  // プロセス2
            #20;
            $display("[%0t] Process 2 done", $time);
        end
        begin  // プロセス3
            #30;
            $display("[%0t] Process 3 done", $time);
        end
    join  // 全プロセスが完了するまで待つ
    $display("[%0t] All processes complete", $time);  // t=30で実行
end
```

#### fork-join_any：いずれかのプロセスの完了で続行

```systemverilog
initial begin
    fork
        begin  // プロセス1: 高速応答
            #5;
            $display("[%0t] Fast response", $time);
        end
        begin  // プロセス2: 遅い応答
            #100;
            $display("[%0t] Slow response", $time);
        end
    join_any  // 最初に完了したプロセスで続行
    $display("[%0t] At least one process done", $time);  // t=5で実行
    // 注意: プロセス2はまだバックグラウンドで実行中！
end
```

#### fork-join_none：即座に続行（バックグラウンド実行）

```systemverilog
initial begin
    $display("[%0t] Launching background processes", $time);
    fork
        begin  // バックグラウンドプロセス1
            forever begin
                @(posedge clk);
                monitor_bus();
            end
        end
        begin  // バックグラウンドプロセス2
            forever begin
                @(posedge clk);
                check_protocol();
            end
        end
    join_none  // 待たずに即座に続行
    $display("[%0t] Background processes launched", $time);  // t=0で実行

    // メインの処理を続行
    #1000;
    $display("[%0t] Main test complete", $time);
end
```

### 11.2.2 比較表

| バリエーション | 待機条件 | 主な用途 |
|--------------|---------|---------|
| `fork-join` | **全て**のプロセスが完了 | 並列操作の完了を待つ |
| `fork-join_any` | **いずれか**のプロセスが完了 | タイムアウト付き待機、レース検出 |
| `fork-join_none` | **待たない**（即座に続行） | バックグラウンド監視の起動 |

![fork-joinの3つのバリエーション](/images/systemverilog-complete-guide/ch11_fork_join_variants.drawio.png)

### 11.2.3 実践例：タイムアウト付きトランザクション

`fork-join_any` を使った典型的なパターンがタイムアウト検出です。

```systemverilog
task automatic wait_for_response(output bit timeout_flag);
    timeout_flag = 0;
    fork
        begin  // 応答待ち
            @(posedge response_valid);
            $display("[%0t] Response received!", $time);
        end
        begin  // タイムアウト
            #10000;
            timeout_flag = 1;
            $display("[%0t] TIMEOUT!", $time);
        end
    join_any
    disable fork;  // 残りのプロセスをキャンセル
endtask
```

---

## 11.3 プロセス管理

### 11.3.1 wait fork

`wait fork` は、現在のスコープで起動された全ての子プロセスが完了するまで待機します。

```systemverilog
task automatic run_all_tests();
    // テストをバックグラウンドで起動
    fork
        test_read();
    join_none
    fork
        test_write();
    join_none
    fork
        test_burst();
    join_none

    // 全テストの完了を待つ
    wait fork;
    $display("[%0t] All tests completed", $time);
endtask
```

### 11.3.2 disable fork

`disable fork` は、現在のスコープから起動された全ての子プロセスを強制的に終了させます。

```systemverilog
task automatic test_with_cleanup();
    fork
        begin  // メイン処理
            drive_stimulus();
        end
        begin  // 監視プロセス
            forever begin
                @(posedge clk);
                if (error_detected) begin
                    $error("Error detected during test!");
                    break;
                end
            end
        end
    join_any

    // fork内の残りのプロセスを全て停止
    disable fork;
    $display("Test finished, all processes cleaned up");
endtask
```

### 11.3.3 disable fork の注意点

`disable fork` は現在のスコープから起動された**全ての**子プロセスを停止するため、意図しないプロセスまで停止してしまう危険があります。これを防ぐために、`fork-join` で囲むパターンが推奨されます。

```systemverilog
task automatic safe_timeout_task();
    // fork-joinで囲むことで、disable forkの影響範囲を限定
    fork begin
        fork
            begin  // 実行したい処理
                do_work();
            end
            begin  // タイムアウト
                #1000;
                $display("Timeout!");
            end
        join_any
        disable fork;  // この fork-join 内のプロセスのみ停止
    end join
endtask
```

---

## 11.4 同期プリミティブ

### 11.4.1 event（イベント）

`event` は、プロセス間のシグナリングに使用するシンプルな同期メカニズムです。あるプロセスがイベントをトリガし、別のプロセスがそのイベントを待ちます。

```systemverilog
module event_example;
    event data_ready;    // イベントの宣言
    event ack_received;

    // プロデューサ: データを準備してイベントをトリガ
    initial begin
        #10;
        $display("[%0t] Producer: Data is ready", $time);
        -> data_ready;    // イベントをトリガ（発火）

        @(ack_received);  // ACKを待つ
        $display("[%0t] Producer: ACK received", $time);
    end

    // コンシューマ: イベントを待ってデータを処理
    initial begin
        @(data_ready);    // イベントの発火を待つ
        $display("[%0t] Consumer: Processing data", $time);
        #5;
        $display("[%0t] Consumer: Done, sending ACK", $time);
        -> ack_received;
    end
endmodule
```

#### triggered プロパティ

`@` によるイベント待ちはイベントが発火する「瞬間」を捉える必要がありますが、`triggered` プロパティを使うと同一タイムステップ内で発火済みかどうかを判定できます。

```systemverilog
event e;

// 方法1: @で待つ（発火の瞬間を逃す可能性あり）
@(e);

// 方法2: triggeredで安全に待つ
wait(e.triggered);  // 同一タイムステップで既に発火済みならすぐ進む
```

### 11.4.2 semaphore（セマフォ）

`semaphore` は、共有リソースへのアクセスを制御するための同期メカニズムです。鍵（key）の概念を使い、鍵を取得（`get`）できたプロセスだけがリソースにアクセスできます。

```systemverilog
module semaphore_example;
    semaphore bus_lock;

    initial begin
        bus_lock = new(1);  // 鍵1個のセマフォ（排他制御）

        fork
            bus_access("Agent A", 8'hAA);
            bus_access("Agent B", 8'hBB);
            bus_access("Agent C", 8'hCC);
        join
    end

    task automatic bus_access(string name, logic [7:0] data);
        $display("[%0t] %s: Waiting for bus access", $time, name);
        bus_lock.get(1);    // 鍵を取得（取得できるまでブロック）
        $display("[%0t] %s: Got bus access, sending 0x%h", $time, name, data);
        #20;                // バスを使用
        $display("[%0t] %s: Done, releasing bus", $time, name);
        bus_lock.put(1);    // 鍵を返却
    endtask
endmodule
```

#### セマフォの主なメソッド

| メソッド | 説明 |
|---------|------|
| `new(int keyCount)` | 鍵の初期数を指定して生成 |
| `get(int count=1)` | 鍵を取得（ブロッキング） |
| `put(int count=1)` | 鍵を返却 |
| `try_get(int count=1)` | 鍵の取得を試みる（ノンブロッキング、成功で1を返す） |

```systemverilog
// try_getを使ったノンブロッキングアクセス
task automatic try_bus_access(string name);
    if (bus_lock.try_get(1)) begin
        $display("[%0t] %s: Got bus access", $time, name);
        #10;
        bus_lock.put(1);
    end else begin
        $display("[%0t] %s: Bus busy, skipping", $time, name);
    end
endtask
```

### 11.4.3 mailbox（メールボックス）

`mailbox` は、プロセス間でデータを安全に受け渡すためのFIFO（先入れ先出し）キューです。プロデューサ・コンシューマパターンの実装に最適です。

```systemverilog
class Transaction;
    rand bit [31:0] addr;
    rand bit [31:0] data;
    rand bit        write;

    function void display(string prefix = "");
        $display("%sTxn: %s addr=0x%h, data=0x%h",
                 prefix, write ? "WR" : "RD", addr, data);
    endfunction
endclass

module mailbox_example;
    mailbox #(Transaction) txn_mb;  // 型パラメータ付きメールボックス

    initial begin
        txn_mb = new(8);  // 容量8のバウンド型メールボックス
        fork
            producer();
            consumer();
        join
    end

    // プロデューサ: トランザクションを生成してメールボックスに格納
    task automatic producer();
        Transaction txn;
        repeat (10) begin
            txn = new();
            assert(txn.randomize());
            txn.display("[Producer] ");
            txn_mb.put(txn);  // メールボックスに格納（満杯ならブロック）
            #10;
        end
    endtask

    // コンシューマ: メールボックスからトランザクションを取り出して処理
    task automatic consumer();
        Transaction txn;
        repeat (10) begin
            txn_mb.get(txn);  // メールボックスから取り出し（空ならブロック）
            txn.display("[Consumer] ");
            #15;
        end
    endtask
endmodule
```

#### メールボックスの主なメソッド

| メソッド | 説明 |
|---------|------|
| `new(int bound=0)` | 生成（boundが0なら無制限） |
| `put(item)` | データを格納（満杯ならブロック） |
| `get(ref item)` | データを取り出し（空ならブロック） |
| `try_put(item)` | 格納を試みる（ノンブロッキング） |
| `try_get(ref item)` | 取り出しを試みる（ノンブロッキング） |
| `peek(ref item)` | 先頭のデータを見る（取り出さない） |
| `try_peek(ref item)` | peekのノンブロッキング版 |
| `num()` | 格納されているデータ数を返す |

![mailboxのプロデューサ・コンシューマパターン](/images/systemverilog-complete-guide/ch11_mailbox_pattern.drawio.png)

---

## 11.5 同期プリミティブの使い分け

### 11.5.1 比較表

| 特性 | event | semaphore | mailbox |
|------|-------|-----------|---------|
| 目的 | 通知・シグナリング | リソース排他制御 | データの受け渡し |
| データ転送 | なし | なし | あり（FIFO） |
| ブロッキング | `@` / `wait` | `get` | `get` / `put` |
| 典型的な用途 | 状態変化の通知 | バスアクセスの排他 | Driver-Sequencer間通信 |

### 11.5.2 検証環境での組み合わせ例

実際の検証環境では、これらのプリミティブを組み合わせて使います。

```systemverilog
class VerificationEnv;
    mailbox #(Transaction) gen2drv_mb;   // Generator → Driver
    mailbox #(Transaction) mon2scb_mb;   // Monitor → Scoreboard
    semaphore              bus_sem;       // バスアクセス制御
    event                  test_done;     // テスト完了通知

    function new();
        gen2drv_mb = new();
        mon2scb_mb = new();
        bus_sem    = new(1);
    endfunction

    task run();
        fork
            generator();
            driver();
            monitor();
            scoreboard();
        join_none

        // テスト完了を待つ
        @(test_done);
        $display("[%0t] Test completed!", $time);
    endtask

    task automatic generator();
        Transaction txn;
        repeat (100) begin
            txn = new();
            assert(txn.randomize());
            gen2drv_mb.put(txn);
            #1;
        end
        -> test_done;  // 全トランザクション生成完了
    endtask

    task automatic driver();
        Transaction txn;
        forever begin
            gen2drv_mb.get(txn);
            bus_sem.get(1);       // バスの排他アクセスを取得
            drive_to_dut(txn);
            bus_sem.put(1);
        end
    endtask

    task automatic monitor();
        Transaction observed;
        forever begin
            @(posedge clk iff valid);
            observed = new();
            observed.addr  = dut_addr;
            observed.data  = dut_data;
            observed.write = dut_write;
            mon2scb_mb.put(observed);
        end
    endtask

    task automatic scoreboard();
        Transaction expected, actual;
        forever begin
            mon2scb_mb.get(actual);
            // 期待値と比較...
        end
    endtask
endclass
```

---

## 11.6 細粒度プロセス制御（fine-grain process control）

### 11.6.1 process クラス

SystemVerilogの `process` クラスを使うと、個別のプロセスをより細かく制御できます。

```systemverilog
initial begin
    process p1, p2;

    fork
        begin
            p1 = process::self();  // 現在のプロセスハンドルを取得
            #100;
        end
        begin
            p2 = process::self();
            #200;
        end
    join_none

    #50;
    $display("P1 status: %s", p1.status().name());  // WAITING
    $display("P2 status: %s", p2.status().name());  // WAITING

    p1.kill();  // プロセス1を強制終了
    $display("P1 status: %s", p1.status().name());  // KILLED

    p2.await();  // プロセス2の完了を待つ
    $display("P2 status: %s", p2.status().name());  // FINISHED
end
```

#### process のメソッド

| メソッド | 説明 |
|---------|------|
| `self()` | 現在のプロセスのハンドルを返す（static） |
| `status()` | プロセスの状態を返す（FINISHED, RUNNING, WAITING, SUSPENDED, KILLED） |
| `kill()` | プロセスを強制終了 |
| `await()` | プロセスの完了を待つ |
| `suspend()` | プロセスを一時停止 |
| `resume()` | 一時停止されたプロセスを再開 |

---

## 11.7 まとめ

本章では、SystemVerilogのプロセス制御と同期について学びました。

1. **`fork-join`** は全プロセスの完了を待ちます。**`fork-join_any`** は最初のプロセス完了で続行します。**`fork-join_none`** はバックグラウンド実行です。
2. **`wait fork`** は全子プロセスの完了を待ち、**`disable fork`** は全子プロセスを停止します。`disable fork` の影響範囲に注意が必要です。
3. **`event`** はシンプルな通知メカニズムです。`triggered` プロパティで同一タイムステップの発火を検出できます。
4. **`semaphore`** は共有リソースの排他制御に使用します。`get`/`put` でブロッキング、`try_get` でノンブロッキングです。
5. **`mailbox`** はプロセス間のデータ受け渡し用FIFOです。型パラメータで型安全性を確保できます。
6. 検証環境では、これらのプリミティブを**組み合わせて**使用し、Generator-Driver-Monitor-Scoreboardのパイプラインを構築します。

次章では、設計の正しさを形式的に記述するSystemVerilog Assertions（SVA）について学びます。
