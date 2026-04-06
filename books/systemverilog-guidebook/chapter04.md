---
title: "第4章：配列と集合"
---

# 第4章：配列と集合

## 4.1 この章で学ぶこと

SystemVerilogは、ハードウェア設計と検証の両方で必要となる多彩な配列型を提供しています。従来のVerilogでは固定サイズの配列しか扱えませんでしたが、SystemVerilogでは動的配列、連想配列、キューといった柔軟なデータ構造が利用可能になりました。

本章では、以下の内容を扱います。

- **配列の種類**：静的配列、動的配列、連想配列、キューの宣言と操作
- **パック配列とアンパック配列**：メモリ配置の違いと使い分け
- **配列メソッド**：ソート、検索、集約など、配列に対する組み込み操作

これらのデータ構造を正しく使い分けることで、テストベンチの記述効率が飛躍的に向上し、複雑なデータ管理も簡潔に表現できるようになります。

![配列の種類と特徴の概要](/images/systemverilog-complete-guide/ch04_array_overview.drawio.png)

---

## 4.2 配列の種類

### 4.2.1 静的配列（固定サイズ配列）

静的配列は、コンパイル時にサイズが確定する最も基本的な配列です。Verilogから引き継がれた構文であり、RTL設計で広く使用されます。

```systemverilog
// 1次元配列の宣言
logic [7:0] mem [0:255];         // 256要素の8ビット配列
int         scores [5];          // 5要素のint配列（インデックス0〜4）
bit [31:0]  registers [0:31];    // 32個の32ビットレジスタ

// 配列の初期化
int data [4] = '{10, 20, 30, 40};            // リテラル初期化
int zeros [8] = '{default: 0};               // 全要素を0で初期化
logic [7:0] pattern [4] = '{8'hAA, 8'h55, 8'hAA, 8'h55};

// 配列要素へのアクセス
initial begin
    mem[0] = 8'hFF;
    mem[1] = 8'h00;
    scores[2] = 85;

    for (int i = 0; i < 4; i++) begin
        $display("data[%0d] = %0d", i, data[i]);
    end
end
```

#### 多次元配列

静的配列は多次元に拡張できます。2次元配列はメモリやレジスタファイルのモデリングでよく使用されます。

```systemverilog
// 2次元配列
int matrix [3][4];               // 3行4列の行列
logic [7:0] frame_buf [480][640]; // 480x640のフレームバッファ

// 3次元配列
logic [7:0] cube [4][4][4];      // 4x4x4の3次元配列

// 多次元配列の初期化とアクセス
initial begin
    // 2次元配列の初期化
    matrix = '{
        '{1, 2, 3, 4},
        '{5, 6, 7, 8},
        '{9, 10, 11, 12}
    };

    // 要素アクセス
    $display("matrix[1][2] = %0d", matrix[1][2]);  // 7

    // ネストしたforループでの走査
    for (int row = 0; row < 3; row++) begin
        for (int col = 0; col < 4; col++) begin
            $display("[%0d][%0d] = %0d", row, col, matrix[row][col]);
        end
    end
end
```

---

### 4.2.2 動的配列

動的配列は、サイズを実行時に決定できる配列です。宣言時にはサイズを指定せず、`new[]`コンストラクタでメモリを確保します。テストベンチにおいて、テストデータの件数が事前に不明な場合に特に有用です。

```systemverilog
// 動的配列の宣言（サイズ未指定）
int data [];
logic [7:0] packet_data [];
bit [31:0] test_vectors [];

// new[]によるサイズ確保
initial begin
    // サイズを指定してメモリ確保
    data = new[10];             // 10要素を確保
    $display("サイズ: %0d", data.size());  // 10

    // 要素に値を代入
    for (int i = 0; i < data.size(); i++) begin
        data[i] = i * 100;
    end

    // サイズ変更（既存データを保持）
    data = new[20](data);       // 20要素に拡張、元の10要素はコピーされる
    $display("拡張後サイズ: %0d", data.size());  // 20
    $display("data[5] = %0d", data[5]);          // 500（元データ保持）

    // サイズ変更（既存データを破棄）
    data = new[5];              // 5要素で新規確保、元データは破棄
    $display("再確保後サイズ: %0d", data.size()); // 5

    // 配列の削除
    data.delete();
    $display("削除後サイズ: %0d", data.size());  // 0
end
```

動的配列は、ファイルからテストデータを読み込む場面で特に威力を発揮します。

```systemverilog
// ファイルからデータを読み込んで動的配列に格納
logic [7:0] file_data [];
int fd, count;

initial begin
    // まずデータ数をカウント
    fd = $fopen("test_data.txt", "r");
    count = 0;
    while (!$feof(fd)) begin
        int tmp;
        if ($fscanf(fd, "%d", tmp) == 1)
            count++;
    end
    $fclose(fd);

    // 必要なサイズだけ確保
    file_data = new[count];

    // データを読み込み
    fd = $fopen("test_data.txt", "r");
    for (int i = 0; i < count; i++) begin
        int val;
        void'($fscanf(fd, "%d", val));
        file_data[i] = val[7:0];
    end
    $fclose(fd);

    $display("読み込んだデータ数: %0d", file_data.size());
end
```

---

### 4.2.3 連想配列

連想配列は、スパース（疎）なデータを効率的に格納するためのデータ構造です。通常の配列とは異なり、インデックスが連続している必要がありません。巨大なアドレス空間のうち一部のみにデータが存在するメモリモデルなどで威力を発揮します。

```systemverilog
// 連想配列の宣言
int    score_map [string];          // 文字列をキーとする連想配列
logic [31:0] memory [int];          // intをキーとする連想配列（スパースメモリ）
bit [7:0] sparse_mem [logic [31:0]]; // 32ビットアドレスをキーとする配列

// 基本的な使い方
initial begin
    // 要素の追加（キーを指定して代入）
    score_map["Alice"]   = 95;
    score_map["Bob"]     = 82;
    score_map["Charlie"] = 78;

    // 要素のアクセス
    $display("Aliceのスコア: %0d", score_map["Alice"]);  // 95

    // 要素数の確認
    $display("登録数: %0d", score_map.num());  // 3

    // キーの存在確認
    if (score_map.exists("Bob"))
        $display("Bobは登録済み");

    // 要素の削除
    score_map.delete("Charlie");
    $display("削除後の登録数: %0d", score_map.num());  // 2

    // 全要素の走査
    string name;
    if (score_map.first(name)) begin
        do begin
            $display("%s: %0d", name, score_map[name]);
        end while (score_map.next(name));
    end
end
```

#### スパースメモリモデル

連想配列の代表的な活用例として、スパースメモリモデルがあります。4GBのアドレス空間を持つメモリを静的配列で実現すると膨大なメモリが必要になりますが、連想配列を使えばアクセスされたアドレスの分だけメモリを消費します。

```systemverilog
// 連想配列によるスパースメモリモデル
class sparse_memory;
    logic [7:0] mem [logic [31:0]];  // 32ビットアドレス、8ビットデータ

    // 書き込み
    function void write(logic [31:0] addr, logic [7:0] data);
        mem[addr] = data;
    endfunction

    // 読み出し（未書き込みアドレスはデフォルト値を返す）
    function logic [7:0] read(logic [31:0] addr);
        if (mem.exists(addr))
            return mem[addr];
        else
            return 8'hxx;  // 未初期化
    endfunction

    // メモリ使用量の確認
    function int get_usage();
        return mem.num();
    endfunction
endclass
```

#### ワイルドカードインデックス

連想配列のインデックス型に`[*]`を使用すると、任意の整数型をキーとして使用できます。

```systemverilog
// ワイルドカードインデックス
int wildcard_map [*];

initial begin
    wildcard_map[0]    = 100;
    wildcard_map[1000] = 200;
    wildcard_map[-5]   = 300;

    $display("要素数: %0d", wildcard_map.num());  // 3
end
```

![静的配列・動的配列・連想配列のメモリ配置の比較](/images/systemverilog-complete-guide/ch04_array_memory_layout.drawio.png)

---

### 4.2.4 キュー

キューは、可変長のFIFO（First-In, First-Out）的なデータ構造であり、`[$]`で宣言します。動的配列とは異なり、`new[]`を呼び出す必要がなく、要素の追加と削除が柔軟に行えます。テストベンチにおけるトランザクションの管理やスコアボードの実装で頻繁に使用されます。

```systemverilog
// キューの宣言
int q [$];                      // 空のキュー
int q_init [$] = {1, 2, 3};    // 初期値付きキュー
string name_q [$];              // 文字列のキュー

// キューの基本操作
initial begin
    int q [$];

    // 末尾に要素を追加
    q.push_back(10);
    q.push_back(20);
    q.push_back(30);
    $display("キュー: %p", q);  // '{10, 20, 30}

    // 先頭に要素を追加
    q.push_front(5);
    $display("キュー: %p", q);  // '{5, 10, 20, 30}

    // 末尾から要素を取り出し
    int last = q.pop_back();
    $display("取り出し: %0d", last);   // 30
    $display("キュー: %p", q);         // '{5, 10, 20}

    // 先頭から要素を取り出し
    int first = q.pop_front();
    $display("取り出し: %0d", first);  // 5
    $display("キュー: %p", q);         // '{10, 20}

    // サイズの確認
    $display("サイズ: %0d", q.size()); // 2

    // 特定位置への挿入
    q.insert(1, 15);                   // インデックス1に15を挿入
    $display("キュー: %p", q);         // '{10, 15, 20}

    // 特定位置の要素を削除
    q.delete(0);                       // インデックス0を削除
    $display("キュー: %p", q);         // '{15, 20}

    // 全要素の削除
    q.delete();
    $display("サイズ: %0d", q.size()); // 0
end
```

#### キューを用いたスコアボードの例

キューはUVM等の検証環境でスコアボードを構築する際に頻繁に利用されます。以下に簡単な例を示します。

```systemverilog
// キューを使ったシンプルなスコアボード
class scoreboard;
    typedef struct {
        int id;
        logic [31:0] data;
        logic [31:0] addr;
    } transaction_t;

    transaction_t expected_q [$];  // 期待値キュー

    // 期待値を追加
    function void add_expected(int id, logic [31:0] data, logic [31:0] addr);
        transaction_t t;
        t.id   = id;
        t.data = data;
        t.addr = addr;
        expected_q.push_back(t);
    endfunction

    // 実測値と比較
    function void check(int id, logic [31:0] data, logic [31:0] addr);
        if (expected_q.size() == 0) begin
            $error("予期しないトランザクション: id=%0d", id);
            return;
        end

        transaction_t exp = expected_q.pop_front();
        if (exp.data !== data || exp.addr !== addr) begin
            $error("不一致: 期待 data=%h addr=%h, 実測 data=%h addr=%h",
                   exp.data, exp.addr, data, addr);
        end else begin
            $display("一致: id=%0d OK", id);
        end
    endfunction

    // 未処理のトランザクション数
    function int pending();
        return expected_q.size();
    endfunction
endclass
```

---

## 4.3 パック配列とアンパック配列

SystemVerilogの配列には、**パック配列**（packed array）と**アンパック配列**（unpacked array）という2つの概念があります。これらはメモリ上の配置が根本的に異なり、適切に使い分けることが重要です。

### 4.3.1 パック配列

パック配列は、全ビットが連続したビットベクタとして格納されます。宣言では型名の直後（変数名の前）に次元を記述します。

```systemverilog
// パック配列の宣言
logic [3:0][7:0] packed_data;   // 4つの8ビット要素 = 32ビットのビットベクタ
bit [1:0][3:0][7:0] bus_data;   // 2x4x8 = 64ビットのビットベクタ

initial begin
    // パック配列はビットベクタとして連続
    packed_data = 32'hDEAD_BEEF;

    // 個別要素へのアクセス
    $display("packed_data[3] = %h", packed_data[3]);  // DE（最上位バイト）
    $display("packed_data[2] = %h", packed_data[2]);  // AD
    $display("packed_data[1] = %h", packed_data[1]);  // BE
    $display("packed_data[0] = %h", packed_data[0]);  // EF（最下位バイト）

    // ビットベクタ全体としても操作可能
    logic [31:0] flat = packed_data;  // そのまま代入可能
    $display("全体: %h", flat);       // DEADBEEF
end
```

### 4.3.2 アンパック配列

アンパック配列は、各要素が独立したメモリ領域に格納されます。宣言では変数名の後ろに次元を記述します。

```systemverilog
// アンパック配列の宣言
logic [7:0] unpacked_data [4];  // 4つの独立した8ビット要素
int values [8];                 // 8つの独立したint要素

initial begin
    unpacked_data[0] = 8'hEF;
    unpacked_data[1] = 8'hBE;
    unpacked_data[2] = 8'hAD;
    unpacked_data[3] = 8'hDE;

    // アンパック配列は要素ごとに独立しているため、
    // ビットベクタ全体として直接扱うことはできない
    // logic [31:0] flat = unpacked_data;  // コンパイルエラー

    // foreachによる走査
    foreach (unpacked_data[i]) begin
        $display("unpacked_data[%0d] = %h", i, unpacked_data[i]);
    end
end
```

### 4.3.3 パック配列とアンパック配列の比較

| 特性 | パック配列 | アンパック配列 |
|------|-----------|---------------|
| 次元の位置 | 型名の後（変数名の前） | 変数名の後 |
| メモリ配置 | 連続したビットベクタ | 要素ごとに独立 |
| ビット全体操作 | 可能 | 不可 |
| 要素の型制約 | 整数型、enum等に限定 | 任意の型 |
| 主な用途 | RTL設計（バス、レジスタ） | テストベンチ、メモリモデル |

### 4.3.4 混在使用時の注意点

パック次元とアンパック次元は同一の変数に混在させることができます。この場合、パック次元が変数名の左側に、アンパック次元が右側に記述されます。

```systemverilog
// パックとアンパックの混在
logic [3:0][7:0] mixed_array [8];
// ^^^^^^^^^^^^               ^^^
// パック次元（32ビットベクタ） アンパック次元（8要素）
// つまり、32ビットのベクタが8個並んだ配列

initial begin
    mixed_array[0] = 32'h1234_5678;        // アンパック要素0に32ビット値を代入
    mixed_array[0][3] = 8'hAA;             // アンパック要素0のパック要素3を変更

    $display("mixed_array[0] = %h", mixed_array[0]);     // AA345678
    $display("mixed_array[0][2] = %h", mixed_array[0][2]); // 34
end
```

パック配列とアンパック配列の間で代入を行う際は、ストリーミング演算子（`{>>{}}`, `{<<{}}`）を使用する必要がある場合があります。

```systemverilog
// ストリーミング演算子によるパック⇔アンパック変換
logic [7:0] unpacked [4];
logic [31:0] packed_val;

initial begin
    unpacked = '{8'hDE, 8'hAD, 8'hBE, 8'hEF};

    // アンパック配列からパック値への変換
    packed_val = {>>{unpacked}};
    $display("packed: %h", packed_val);  // DEADBEEF

    // パック値からアンパック配列への変換
    packed_val = 32'hCAFE_BABE;
    {>>{unpacked}} = packed_val;
    foreach (unpacked[i])
        $display("unpacked[%0d] = %h", i, unpacked[i]);
end
```

![パック配列とアンパック配列のメモリ配置](/images/systemverilog-complete-guide/ch04_packed_vs_unpacked.drawio.png)

---

## 4.4 配列メソッド

SystemVerilogの配列には、ソート、検索、集約といった強力な組み込みメソッドが用意されています。これらのメソッドを活用することで、複雑なデータ操作を簡潔に記述できます。

### 4.4.1 ソートと並べ替え

#### sort() と rsort()

`sort()`は昇順に、`rsort()`は降順に配列をソートします。これらのメソッドは配列を直接変更します（in-place操作）。

```systemverilog
initial begin
    int arr [$] = {30, 10, 50, 20, 40};

    // 昇順ソート
    arr.sort();
    $display("昇順: %p", arr);  // '{10, 20, 30, 40, 50}

    // 降順ソート
    arr.rsort();
    $display("降順: %p", arr);  // '{50, 40, 30, 20, 10}
end
```

構造体の配列をソートする場合は、`with`句でソートキーを指定します。

```systemverilog
typedef struct {
    string name;
    int    score;
    int    age;
} student_t;

initial begin
    student_t students [$];
    students.push_back('{"Alice",   90, 20});
    students.push_back('{"Bob",     75, 22});
    students.push_back('{"Charlie", 88, 19});
    students.push_back('{"Diana",   95, 21});

    // スコアで昇順ソート
    students.sort() with (item.score);
    foreach (students[i])
        $display("%s: %0d点", students[i].name, students[i].score);
    // Bob: 75点, Charlie: 88点, Alice: 90点, Diana: 95点
end
```

#### reverse() と shuffle()

`reverse()`は配列の要素を逆順にし、`shuffle()`はランダムに並べ替えます。

```systemverilog
initial begin
    int arr [$] = {1, 2, 3, 4, 5};

    // 逆順
    arr.reverse();
    $display("逆順: %p", arr);  // '{5, 4, 3, 2, 1}

    // シャッフル（ランダム並べ替え）
    arr.shuffle();
    $display("シャッフル: %p", arr);  // ランダム順（例: '{3, 1, 5, 2, 4}）
end
```

---

### 4.4.2 検索メソッド

配列の検索メソッドは、条件に一致する要素やインデックスをキューとして返します。

#### find()

`with`句で指定した条件に一致する全要素を返します。

```systemverilog
initial begin
    int arr [$] = {15, 3, 28, 7, 42, 10, 35};

    // 20以上の要素を検索
    int result [$] = arr.find() with (item >= 20);
    $display("20以上: %p", result);  // '{28, 42, 35}

    // 偶数の要素を検索
    int evens [$] = arr.find() with (item % 2 == 0);
    $display("偶数: %p", evens);  // '{28, 42, 10}
end
```

#### find_first() と find_last()

条件に一致する最初の要素または最後の要素を返します。戻り値はキュー（要素数0または1）です。

```systemverilog
initial begin
    int arr [$] = {5, 12, 8, 25, 3, 18};

    // 条件に一致する最初の要素
    int first [$] = arr.find_first() with (item > 10);
    if (first.size() > 0)
        $display("最初の>10: %0d", first[0]);  // 12

    // 条件に一致する最後の要素
    int last [$] = arr.find_last() with (item > 10);
    if (last.size() > 0)
        $display("最後の>10: %0d", last[0]);   // 18
end
```

#### find_index()、find_first_index()、find_last_index()

条件に一致する要素のインデックスを返します。

```systemverilog
initial begin
    int arr [$] = {5, 12, 8, 25, 3, 18};

    // 10以上の要素のインデックスをすべて取得
    int indices [$] = arr.find_index() with (item >= 10);
    $display("インデックス: %p", indices);  // '{1, 3, 5}

    // 最初のインデックス
    int first_idx [$] = arr.find_first_index() with (item >= 10);
    if (first_idx.size() > 0)
        $display("最初のインデックス: %0d", first_idx[0]);  // 1

    // 最後のインデックス
    int last_idx [$] = arr.find_last_index() with (item >= 10);
    if (last_idx.size() > 0)
        $display("最後のインデックス: %0d", last_idx[0]);   // 5
end
```

---

### 4.4.3 集約メソッド

集約メソッドは、配列の全要素に対して演算を行い、単一の値を返します。

#### sum() と product()

```systemverilog
initial begin
    int arr [$] = {1, 2, 3, 4, 5};

    // 合計
    int total = arr.sum();
    $display("合計: %0d", total);  // 15

    // 積
    int prod = arr.product();
    $display("積: %0d", prod);     // 120

    // with句を使った条件付き合計
    int even_sum = arr.sum() with (item % 2 == 0 ? item : 0);
    $display("偶数の合計: %0d", even_sum);  // 6 (2 + 4)
end
```

#### and()、or()、xor()

ビット単位の集約演算を行います。

```systemverilog
initial begin
    logic [7:0] arr [$] = {8'hFF, 8'h0F, 8'hF0};

    // ビット単位AND（全要素のAND）
    logic [7:0] and_result = arr.and();
    $display("AND: %h", and_result);  // 00 (FF & 0F & F0)

    // ビット単位OR（全要素のOR）
    logic [7:0] or_result = arr.or();
    $display("OR:  %h", or_result);   // FF (FF | 0F | F0)

    // ビット単位XOR（全要素のXOR）
    logic [7:0] xor_result = arr.xor();
    $display("XOR: %h", xor_result);  // 00 (FF ^ 0F ^ F0)
end
```

---

### 4.4.4 with句による条件指定

`with`句は多くの配列メソッドで使用でき、柔軟な条件指定を可能にします。`item`は配列の各要素を参照する暗黙の変数です。

```systemverilog
typedef struct {
    string name;
    int    score;
    string grade;
} record_t;

initial begin
    record_t records [$];
    records.push_back('{"Alice",   92, "A"});
    records.push_back('{"Bob",     67, "C"});
    records.push_back('{"Charlie", 85, "B"});
    records.push_back('{"Diana",   78, "B"});
    records.push_back('{"Eve",     95, "A"});

    // スコアが80以上の学生を検索
    record_t honors [$] = records.find() with (item.score >= 80);
    $display("80点以上: %0d人", honors.size());  // 3

    // スコアの合計をwith句で計算
    int total_score = records.sum() with (item.score);
    $display("合計スコア: %0d", total_score);  // 417

    // 最小スコアの要素を検索
    record_t min_q [$] = records.min() with (item.score);
    if (min_q.size() > 0)
        $display("最低スコア: %s (%0d点)", min_q[0].name, min_q[0].score);
    // 最低スコア: Bob (67点)

    // 最大スコアの要素を検索
    record_t max_q [$] = records.max() with (item.score);
    if (max_q.size() > 0)
        $display("最高スコア: %s (%0d点)", max_q[0].name, max_q[0].score);
    // 最高スコア: Eve (95点)

    // ユニークなグレードの一覧
    string grades [$];
    foreach (records[i]) grades.push_back(records[i].grade);
    grades = grades.unique();
    $display("グレード種別: %p", grades);  // '{"A", "C", "B"}
end
```

---

### 4.4.5 配列メソッド一覧

以下に、主要な配列メソッドを一覧にまとめます。

| カテゴリ | メソッド | 説明 | 対象配列 |
|---------|---------|------|---------|
| ソート | `sort()` | 昇順ソート | 固定/動的/キュー |
| ソート | `rsort()` | 降順ソート | 固定/動的/キュー |
| 並べ替え | `reverse()` | 要素を逆順にする | 固定/動的/キュー |
| 並べ替え | `shuffle()` | 要素をランダムに並べ替える | 固定/動的/キュー |
| 検索 | `find()` | 条件に一致する全要素を返す | 固定/動的/キュー |
| 検索 | `find_first()` | 条件に一致する最初の要素を返す | 固定/動的/キュー |
| 検索 | `find_last()` | 条件に一致する最後の要素を返す | 固定/動的/キュー |
| 検索 | `find_index()` | 条件に一致する全インデックスを返す | 固定/動的/キュー |
| 検索 | `find_first_index()` | 条件に一致する最初のインデックスを返す | 固定/動的/キュー |
| 検索 | `find_last_index()` | 条件に一致する最後のインデックスを返す | 固定/動的/キュー |
| 検索 | `min()` | 最小要素を返す | 固定/動的/キュー |
| 検索 | `max()` | 最大要素を返す | 固定/動的/キュー |
| 検索 | `unique()` | 重複を除いた要素を返す | 固定/動的/キュー |
| 集約 | `sum()` | 全要素の合計 | 固定/動的/キュー |
| 集約 | `product()` | 全要素の積 | 固定/動的/キュー |
| 集約 | `and()` | 全要素のビットAND | 固定/動的/キュー |
| 集約 | `or()` | 全要素のビットOR | 固定/動的/キュー |
| 集約 | `xor()` | 全要素のビットXOR | 固定/動的/キュー |

---

## 4.5 まとめ

本章では、SystemVerilogの配列と集合について体系的に解説しました。以下に重要なポイントを整理します。

**配列の種類**
- **静的配列**はコンパイル時にサイズが確定し、RTL設計で広く使用されます。多次元配列にも対応しています
- **動的配列**は`new[]`で実行時にサイズを決定でき、`delete()`で解放できます。テストデータの管理に適しています
- **連想配列**はスパースなデータ格納に最適であり、`[key_type]`でキーの型を指定します。メモリモデルの構築で威力を発揮します。`[*]`のワイルドカードインデックスも利用可能です
- **キュー**は`[$]`で宣言し、`push_back`/`push_front`/`pop_back`/`pop_front`で柔軟にFIFO的な操作が行えます。スコアボード等の検証コンポーネントで頻用されます

**パック配列とアンパック配列**
- パック配列は連続したビットベクタとして格納され、ビット全体の操作が可能です。次元は変数名の左側に記述します
- アンパック配列は要素ごとに独立して格納され、任意の型を要素にできます。次元は変数名の右側に記述します
- 両者の混在使用は可能ですが、相互変換にはストリーミング演算子が必要な場合があります

**配列メソッド**
- `sort()`/`rsort()`で昇順・降順ソート、`reverse()`で逆順、`shuffle()`でランダム並べ替えが可能です
- `find()`系メソッドで条件に一致する要素やインデックスを検索できます
- `sum()`/`product()`/`and()`/`or()`/`xor()`で集約演算が行えます
- `with`句を使用することで、構造体のフィールドや計算式に基づいた条件指定が可能です

次章（第5章）では、組み合わせ回路と順序回路について解説します。
