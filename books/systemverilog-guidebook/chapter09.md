---
title: "第9章：オブジェクト指向プログラミング（OOP）"
---

# 第9章：オブジェクト指向プログラミング（OOP）

## 9.1 この章で学ぶこと

本章では、SystemVerilogにおける**オブジェクト指向プログラミング（OOP）**の仕組みを解説します。クラスの定義、プロパティとメソッド、コンストラクタ（`new`）、継承（`extends`）、多態性（`virtual`）、抽象クラス、静的メンバなど、検証環境の構築に不可欠なOOP機能を体系的に学びます。OOPの概念はUVMをはじめとする現代の検証メソドロジの基盤であり、SystemVerilogエンジニアにとって必須の知識です。

---

## 9.2 クラスの基本

### 9.2.1 クラスとオブジェクト

**クラス（class）**はデータ（プロパティ）と操作（メソッド）をひとまとめにした設計図です。クラスから生成された実体を**オブジェクト（object）**またはインスタンスと呼びます。

ハードウェア記述における `module` がポートと内部回路の設計図であるのと同様に、`class` はデータ構造とその振る舞いの設計図です。ただし、クラスは検証（シミュレーション）専用であり、合成対象にはなりません。

```systemverilog
class Packet;
    // プロパティ（データメンバ）
    bit [7:0]  src_addr;
    bit [7:0]  dst_addr;
    bit [31:0] payload;
    int        packet_id;

    // メソッド（メンバ関数）
    function void display();
        $display("Packet[%0d]: src=0x%h, dst=0x%h, payload=0x%h",
                 packet_id, src_addr, dst_addr, payload);
    endfunction

    function bit is_broadcast();
        return (dst_addr == 8'hFF);
    endfunction
endclass
```

### 9.2.2 オブジェクトの生成と使用

クラスのオブジェクトを使用するには、まず**ハンドル**（参照変数）を宣言し、`new` でオブジェクトを生成します。ハンドルの宣言だけではオブジェクトは生成されず、`null` 状態です。

```systemverilog
module test;
    initial begin
        Packet pkt;          // ハンドルの宣言（この時点ではnull）

        pkt = new();         // オブジェクトの生成（メモリ確保）
        pkt.src_addr  = 8'h01;
        pkt.dst_addr  = 8'hFF;
        pkt.payload   = 32'hDEAD_BEEF;
        pkt.packet_id = 1;

        pkt.display();       // メソッドの呼び出し

        if (pkt.is_broadcast())
            $display("This is a broadcast packet");
    end
endmodule
```

### 9.2.3 コンストラクタ（new）

`new` 関数はクラスの**コンストラクタ**です。オブジェクト生成時に自動的に呼び出され、プロパティの初期化を行います。明示的に定義しない場合、引数なしのデフォルトコンストラクタが自動的に提供されます。

```systemverilog
class Packet;
    bit [7:0]  src_addr;
    bit [7:0]  dst_addr;
    bit [31:0] payload;
    int        packet_id;

    // カスタムコンストラクタ
    function new(bit [7:0] src = 8'h00, bit [7:0] dst = 8'h00);
        this.src_addr  = src;
        this.dst_addr  = dst;
        this.payload   = 32'h0;
        this.packet_id = get_next_id();
    endfunction

    // ID自動採番のためのstatic変数とメソッド
    static int id_counter = 0;

    static function int get_next_id();
        id_counter++;
        return id_counter;
    endfunction

    function void display();
        $display("Packet[%0d]: src=0x%h -> dst=0x%h", packet_id, src_addr, dst_addr);
    endfunction
endclass

// 使用例
module test;
    initial begin
        Packet p1, p2, p3;
        p1 = new(8'h01, 8'h02);   // src=0x01, dst=0x02
        p2 = new(8'h03, 8'hFF);   // src=0x03, dst=0xFF
        p3 = new();                // デフォルト引数を使用

        p1.display();  // Packet[1]: src=0x01 -> dst=0x02
        p2.display();  // Packet[2]: src=0x03 -> dst=0xff
        p3.display();  // Packet[3]: src=0x00 -> dst=0x00
    end
endmodule
```

### 9.2.4 ハンドルの代入とコピー

ハンドルの代入はオブジェクトのコピーではなく、**参照のコピー**です。2つのハンドルが同じオブジェクトを指すことになります。

```systemverilog
initial begin
    Packet p1, p2;
    p1 = new(8'h01, 8'h02);
    p2 = p1;                    // p2はp1と同じオブジェクトを指す

    p2.src_addr = 8'hFF;        // p1.src_addrも変わる！
    $display("p1.src = 0x%h", p1.src_addr);  // 0xFF

    // 独立したコピーを作るには新しいオブジェクトを生成
    Packet p3;
    p3 = new p1;                // シャローコピー（浅いコピー）
    p3.src_addr = 8'h99;
    $display("p1.src = 0x%h", p1.src_addr);  // 0xFF（p3の変更はp1に影響しない）
end
```

![ハンドルとオブジェクトの関係](/images/systemverilog-complete-guide/ch09_handle_object.drawio.png)

---

## 9.3 継承（Inheritance）

### 9.3.1 extends キーワード

**継承**は、既存のクラス（基底クラス/親クラス）の機能を引き継いで新しいクラス（派生クラス/子クラス）を作る仕組みです。`extends` キーワードを使用します。

```systemverilog
// 基底クラス
class Transaction;
    bit [31:0] addr;
    bit [31:0] data;
    int        id;

    function new(bit [31:0] a = 0, bit [31:0] d = 0);
        this.addr = a;
        this.data = d;
    endfunction

    virtual function void display();
        $display("Transaction[%0d]: addr=0x%h, data=0x%h", id, addr, data);
    endfunction
endclass

// 派生クラス: 書き込みトランザクション
class WriteTransaction extends Transaction;
    bit [3:0] byte_enable;

    function new(bit [31:0] a = 0, bit [31:0] d = 0, bit [3:0] be = 4'hF);
        super.new(a, d);          // 親クラスのコンストラクタ呼び出し
        this.byte_enable = be;
    endfunction

    virtual function void display();
        $display("WriteTransaction[%0d]: addr=0x%h, data=0x%h, BE=0x%h",
                 id, addr, data, byte_enable);
    endfunction
endclass

// 派生クラス: 読み出しトランザクション
class ReadTransaction extends Transaction;
    int expected_latency;

    function new(bit [31:0] a = 0, int latency = 1);
        super.new(a, 0);
        this.expected_latency = latency;
    endfunction

    virtual function void display();
        $display("ReadTransaction[%0d]: addr=0x%h, expected_latency=%0d",
                 id, addr, expected_latency);
    endfunction
endclass
```

### 9.3.2 super キーワード

`super` は親クラスのメンバにアクセスするためのキーワードです。特にコンストラクタ内で親クラスの初期化を行う場合に重要です。

```systemverilog
class ErrorTransaction extends WriteTransaction;
    bit inject_error;

    function new(bit [31:0] a, bit [31:0] d, bit err = 0);
        super.new(a, d, 4'hF);   // WriteTransactionのコンストラクタを呼ぶ
        this.inject_error = err;
    endfunction

    virtual function void display();
        super.display();          // 親クラスのdisplayを呼んだ後に追加情報を表示
        if (inject_error)
            $display("  *** ERROR INJECTION ENABLED ***");
    endfunction
endclass
```

### 9.3.3 this キーワード

`this` は現在のオブジェクト自身への参照です。プロパティ名と引数名が同じ場合の曖昧さ解消などに使います。

```systemverilog
class Register;
    string name;
    bit [31:0] value;

    function new(string name, bit [31:0] value);
        this.name  = name;     // this.name = プロパティ, name = 引数
        this.value = value;
    endfunction
endclass
```

---

## 9.4 多態性（Polymorphism）

### 9.4.1 virtual メソッド

**多態性**は、同じインターフェース（メソッド名）で異なる動作を実現する仕組みです。SystemVerilogでは、`virtual` キーワードを付けたメソッドが多態性をサポートします。

`virtual` を付けたメソッドは、実行時にオブジェクトの**実際の型**に基づいて呼び出されます（動的ディスパッチ）。`virtual` なしの場合は、ハンドルの**宣言型**に基づいて呼び出されます（静的ディスパッチ）。

```systemverilog
class Animal;
    string name;

    function new(string name);
        this.name = name;
    endfunction

    // virtualメソッド: 子クラスでオーバーライド可能
    virtual function string speak();
        return "...";
    endfunction

    function void introduce();
        $display("%s says: %s", name, speak());
    endfunction
endclass

class Dog extends Animal;
    function new(string name);
        super.new(name);
    endfunction

    virtual function string speak();
        return "Woof!";
    endfunction
endclass

class Cat extends Animal;
    function new(string name);
        super.new(name);
    endfunction

    virtual function string speak();
        return "Meow!";
    endfunction
endclass

// 多態性のデモ
module test;
    initial begin
        Animal animals[3];  // 基底クラスのハンドル配列

        animals[0] = new("Generic");
        animals[1] = Dog::new("Rex");     // DogオブジェクトをAnimalハンドルに代入
        animals[2] = Cat::new("Whiskers");

        foreach (animals[i])
            animals[i].introduce();
        // 出力:
        // Generic says: ...
        // Rex says: Woof!
        // Whiskers says: Meow!
    end
endmodule
```

### 9.4.2 virtual なしの場合の動作

`virtual` を付けないと、ハンドルの宣言型に基づいてメソッドが解決されます。

```systemverilog
class Base;
    function string get_type();  // virtualなし
        return "Base";
    endfunction
endclass

class Derived extends Base;
    function string get_type();  // オーバーライドのつもりだが...
        return "Derived";
    endfunction
endclass

module test;
    initial begin
        Base b;
        Derived d;
        d = new();
        b = d;  // DerivedオブジェクトをBaseハンドルに代入

        $display("d.get_type() = %s", d.get_type());  // "Derived"
        $display("b.get_type() = %s", b.get_type());  // "Base" ← virtualなしのため
    end
endmodule
```

**重要**: オーバーライドを意図するメソッドには必ず `virtual` を付けましょう。UVM環境では全メソッドが `virtual` であることが前提となっています。

![多態性の仕組み](/images/systemverilog-complete-guide/ch09_polymorphism.drawio.png)

---

## 9.5 抽象クラス（Virtual Class）

### 9.5.1 virtual class の定義

**抽象クラス**は `virtual class` として宣言し、直接インスタンス化できないクラスです。インターフェース（API）を定義し、具体的な実装はサブクラスに委ねます。

```systemverilog
// 抽象クラス: Driverの基本インターフェースを定義
virtual class BaseDriver;
    string name;

    function new(string name);
        this.name = name;
    endfunction

    // 純粋仮想関数: サブクラスで必ず実装が必要
    pure virtual task drive(input bit [31:0] data);
    pure virtual task reset();
    pure virtual function void report();
endclass
```

### 9.5.2 純粋仮想関数（Pure Virtual Function）

`pure virtual` として宣言されたメソッドは、本体を持ちません。すべてのサブクラスで**必ず実装**しなければなりません。これにより、サブクラスが必要なインターフェースを確実に実装することを保証します。

```systemverilog
// 具体的な実装クラス: AXI Driver
class AxiDriver extends BaseDriver;
    virtual axi_if vif;  // 仮想インターフェース

    function new(string name, virtual axi_if vif);
        super.new(name);
        this.vif = vif;
    endfunction

    // 純粋仮想関数の実装
    virtual task drive(input bit [31:0] data);
        @(posedge vif.clk);
        vif.wdata  <= data;
        vif.wvalid <= 1'b1;
        @(posedge vif.clk iff vif.wready);
        vif.wvalid <= 1'b0;
    endtask

    virtual task reset();
        vif.wdata  <= '0;
        vif.wvalid <= 1'b0;
        repeat (5) @(posedge vif.clk);
    endtask

    virtual function void report();
        $display("[%s] AXI Driver report complete", name);
    endfunction
endclass

// 具体的な実装クラス: APB Driver
class ApbDriver extends BaseDriver;
    virtual apb_if vif;

    function new(string name, virtual apb_if vif);
        super.new(name);
        this.vif = vif;
    endfunction

    virtual task drive(input bit [31:0] data);
        @(posedge vif.pclk);
        vif.psel   <= 1'b1;
        vif.pwdata <= data;
        @(posedge vif.pclk);
        vif.penable <= 1'b1;
        @(posedge vif.pclk iff vif.pready);
        vif.psel    <= 1'b0;
        vif.penable <= 1'b0;
    endtask

    virtual task reset();
        vif.psel    <= 1'b0;
        vif.penable <= 1'b0;
        vif.pwdata  <= '0;
        repeat (3) @(posedge vif.pclk);
    endtask

    virtual function void report();
        $display("[%s] APB Driver report complete", name);
    endfunction
endclass
```

### 9.5.3 抽象クラスの活用シーン

抽象クラスは以下の場面で有効です：

- **検証コンポーネントの共通インターフェース定義**（Driver, Monitor等）
- **プロトコル非依存のテストベンチフレームワーク構築**
- **将来の拡張を見据えた設計**（新プロトコルの追加が容易）

```systemverilog
// 抽象クラスを使って、プロトコル非依存のテストを記述
module test;
    initial begin
        BaseDriver driver;

        // 設定に応じて適切なDriverを選択
        if (USE_AXI)
            driver = new AxiDriver("axi_drv", axi_vif);
        else
            driver = new ApbDriver("apb_drv", apb_vif);

        // 同じインターフェースでテスト実行
        driver.reset();
        driver.drive(32'hA5A5_A5A5);
        driver.report();
    end
endmodule
```

---

## 9.6 静的メンバ（Static Members）

### 9.6.1 static プロパティ

`static` プロパティは、クラスの全オブジェクトで**共有**される変数です。オブジェクトごとに個別の値を持つ通常のプロパティとは異なり、クラスに対して1つだけ存在します。

```systemverilog
class Packet;
    // インスタンスプロパティ（各オブジェクト固有）
    bit [31:0] data;
    int        id;

    // 静的プロパティ（全オブジェクトで共有）
    static int packet_count = 0;
    static int total_bytes  = 0;

    function new(bit [31:0] d);
        this.data = d;
        packet_count++;               // 生成されたパケット数をカウント
        this.id = packet_count;
        total_bytes += 4;             // 4バイト加算
    endfunction

    // 静的メソッド（インスタンスなしで呼べる）
    static function void print_stats();
        $display("Total packets: %0d, Total bytes: %0d", packet_count, total_bytes);
    endfunction
endclass

module test;
    initial begin
        Packet p1 = new(32'h1111);
        Packet p2 = new(32'h2222);
        Packet p3 = new(32'h3333);

        Packet::print_stats();  // クラス名::で呼び出し
        // 出力: Total packets: 3, Total bytes: 12
    end
endmodule
```

### 9.6.2 static メソッド

`static` メソッドは、特定のオブジェクトに紐付かない関数です。クラス名を使って直接呼び出すことができ、インスタンスを生成する必要がありません。ただし、`static` メソッド内からは非 `static` メンバにアクセスできません。

```systemverilog
class MathUtils;
    static function int abs(int value);
        return (value < 0) ? -value : value;
    endfunction

    static function int max(int a, int b);
        return (a > b) ? a : b;
    endfunction

    static function int min(int a, int b);
        return (a < b) ? a : b;
    endfunction

    static function int clamp(int value, int lo, int hi);
        if (value < lo) return lo;
        if (value > hi) return hi;
        return value;
    endfunction
endclass

// 使用例: オブジェクトを生成せずに呼べる
module test;
    initial begin
        int result;
        result = MathUtils::abs(-42);       // 42
        result = MathUtils::max(10, 20);    // 20
        result = MathUtils::clamp(150, 0, 100);  // 100
    end
endmodule
```

### 9.6.3 シングルトンパターン

静的メンバを活用した代表的なデザインパターンがシングルトンです。システム全体で1つだけ存在すべきオブジェクト（設定マネージャなど）に使用します。

```systemverilog
class ConfigManager;
    static local ConfigManager instance;  // 唯一のインスタンス
    int timeout;
    int verbosity;

    // コンストラクタをlocalに（外部からのnewを禁止）
    local function new();
        timeout   = 1000;
        verbosity = 1;
    endfunction

    // インスタンス取得メソッド
    static function ConfigManager get();
        if (instance == null)
            instance = new();
        return instance;
    endfunction
endclass

// 使用例
module test;
    initial begin
        ConfigManager cfg = ConfigManager::get();
        cfg.timeout = 5000;

        // 別の場所で取得しても同じインスタンス
        ConfigManager cfg2 = ConfigManager::get();
        $display("timeout = %0d", cfg2.timeout);  // 5000
    end
endmodule
```

---

## 9.7 型キャスト（$cast）

継承関係にあるクラス間で型変換を行う場合、`$cast` を使用します。

```systemverilog
class Transaction;
    bit [31:0] addr;
    virtual function void display();
        $display("Transaction: addr=0x%h", addr);
    endfunction
endclass

class WriteTransaction extends Transaction;
    bit [31:0] data;
    virtual function void display();
        $display("Write: addr=0x%h, data=0x%h", addr, data);
    endfunction
endclass

module test;
    initial begin
        Transaction  base_h;
        WriteTransaction write_h, write_h2;

        write_h = new();
        write_h.addr = 32'h100;
        write_h.data = 32'hABCD;

        // アップキャスト（暗黙的 - 常に安全）
        base_h = write_h;

        // ダウンキャスト（明示的 - $castが必要）
        if ($cast(write_h2, base_h))
            write_h2.display();  // 成功
        else
            $display("Cast failed!");
    end
endmodule
```

---

## 9.8 まとめ

本章では、SystemVerilogのオブジェクト指向プログラミング機能について学びました。

1. **クラス**はデータ（プロパティ）と操作（メソッド）をカプセル化する設計図。`new` でオブジェクトを生成する。
2. **継承（extends）**により既存クラスを拡張できる。`super` で親クラスにアクセスし、`this` で自身を参照する。
3. **多態性（virtual）**により、基底クラスのハンドルから派生クラスのメソッドを呼び出せる。検証環境の柔軟性に不可欠。
4. **抽象クラス（virtual class）**と**純粋仮想関数（pure virtual）**で、サブクラスに実装を強制するインターフェースを定義できる。
5. **静的メンバ（static）**はクラス全体で共有されるプロパティ・メソッド。ユーティリティ関数やカウンタに有用。
6. **$cast** で安全な型変換を行う。ダウンキャストの成否を実行時にチェックできる。

次章では、OOPを基盤としたSystemVerilog独自の強力な機能——制約付きランダム検証について学びます。
