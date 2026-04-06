---
title: "第15章：VPI"
---

# 第15章：VPI

## 15.1 この章で学ぶこと

本章では、SystemVerilogの**VPI（Verilog Procedural Interface）**と、設計・検証における**デバッグ技法**について包括的に解説します。VPIは、C言語からシミュレーション内部のオブジェクトにアクセスし、値の読み書きやコールバックの登録を可能にする強力なインターフェースです。カスタムシステムタスクの作成、信号値のモニタリング、ネットリストの走査など、VPIを活用した高度なシミュレーション制御の方法を学びます。

---

## 15.2 VPIの概要

### 15.2.1 VPIとは（Verilog Procedural Interface）

**VPI（Verilog Procedural Interface）**は、IEEE 1364-2001で標準化されたC言語プログラミングインターフェースです。VPIを使うことで、C/C++プログラムからシミュレーション内部のあらゆるオブジェクト（モジュール、ネット、レジスタ、ポート、タスク、関数など）にアクセスし、操作することができます。

VPIの主な用途は以下の通りです。

- **カスタムシステムタスク/関数の作成**: `$mytask` のような独自のシステムタスクを定義できます
- **シミュレーションの監視・制御**: 信号値の読み書き、コールバックによるイベント通知を実現します
- **外部ツールとの連携**: デバッガ、波形ビューワ、スクリプト言語との統合が可能です
- **ネットリストの走査**: 設計階層を動的にトラバースし、構造情報を取得できます
- **カバレッジ収集・解析**: カスタムカバレッジメトリクスを定義・収集できます

VPIはC言語のヘッダファイル `vpi_user.h` として提供され、シミュレータとリンクする共有ライブラリ（`.so` / `.dll`）として実装します。

```c
#include "vpi_user.h"

// VPIの基本的な使用例：シミュレーション開始時にメッセージを表示
static int start_of_sim_cb(p_cb_data cb_data) {
    vpi_printf("=== シミュレーション開始 ===\n");
    return 0;
}
```

### 15.2.2 PLI 1.0からVPIへの歴史

VPIは長い歴史の中で進化してきました。その経緯を理解することは、既存コードの保守やツール選択に役立ちます。

| 世代 | 名称 | 規格 | 特徴 |
|------|------|------|------|
| 第1世代 | PLI 1.0（TF/ACC） | IEEE 1364-1995 | `tf_` 関数と `acc_` 関数の2つのAPIセット |
| 第2世代 | VPI（PLI 2.0） | IEEE 1364-2001 | 統一されたオブジェクトモデル、一貫したAPI |
| 現行 | VPI（SystemVerilog拡張） | IEEE 1800-2017 | クラス、アサーション等の新オブジェクトに対応 |

PLI 1.0は以下の2つのサブセットから構成されていました。

- **TF（Task/Function）ルーチン**: `tf_putp()`, `tf_getp()` など、システムタスクの引数にアクセスするための関数群
- **ACC（Access）ルーチン**: `acc_handle_by_name()`, `acc_fetch_value()` など、設計オブジェクトにアクセスするための関数群

PLI 1.0の問題点は、TFとACCが別々に設計されたため一貫性がなく、機能の重複や制限がありました。VPIはこれらを統一した単一のAPIとして再設計され、以下の改善を実現しました。

- **統一されたオブジェクトモデル**: すべてのオブジェクトを `vpiHandle` で統一的に扱えます
- **一貫したAPI命名規則**: すべての関数が `vpi_` プレフィックスを持ちます
- **イテレータベースのアクセス**: オブジェクト群を効率的に走査できます
- **コールバックメカニズム**: イベント駆動型のプログラミングが可能です

### 15.2.3 VPIのオブジェクトモデル

VPIはオブジェクト指向のモデルを採用しており、シミュレーション内部のすべてのエンティティを**オブジェクト**として表現します。オブジェクト間の関係は**1対1（one-to-one）**と**1対多（one-to-many）**の2種類で表されます。

![VPIオブジェクトモデル](/images/systemverilog-complete-guide/ch15_vpi_object_model.drawio.png)

主要なオブジェクトタイプは以下の通りです。

| オブジェクトタイプ | 定数名 | 説明 |
|------------------|--------|------|
| モジュール | `vpiModule` | モジュールインスタンス |
| ネット | `vpiNet` | wire, tri などのネット |
| レジスタ | `vpiReg` | reg, logic などのレジスタ |
| ポート | `vpiPort` | モジュールのポート |
| パラメータ | `vpiParameter` | パラメータ定義 |
| タスク | `vpiTask` | タスク定義 |
| 関数 | `vpiFunction` | 関数定義 |
| 内部スコープ | `vpiInternalScope` | begin-end, fork-join ブロック |
| コールバック | `vpiCallback` | 登録されたコールバック |
| システムタスク呼び出し | `vpiSysTfCall` | システムタスク/関数の呼び出し |

オブジェクトへのアクセスは `vpiHandle`（不透明なポインタ型）を通じて行います。

```c
#include "vpi_user.h"

// オブジェクトモデルの走査例
static void traverse_hierarchy(vpiHandle scope, int depth) {
    vpiHandle iter, child;
    char indent[256] = "";

    for (int i = 0; i < depth; i++) strcat(indent, "  ");

    // スコープ名を表示
    vpi_printf("%s%s (%s)\n",
        indent,
        vpi_get_str(vpiName, scope),
        vpi_get_str(vpiType, scope));

    // 子モジュールを走査
    iter = vpi_iterate(vpiModule, scope);
    if (iter != NULL) {
        while ((child = vpi_scan(iter)) != NULL) {
            traverse_hierarchy(child, depth + 1);
        }
    }
}
```

### 15.2.4 DPI-CとVPIの使い分け

SystemVerilogでは、C言語との連携方法としてVPIの他に**DPI-C（Direct Programming Interface for C）**も提供されています。両者は目的と機能が異なるため、適切に使い分けることが重要です。

| 比較項目 | DPI-C | VPI |
|---------|-------|-----|
| **主な用途** | C関数の直接呼び出し | シミュレーション内部へのアクセス |
| **学習コスト** | 低い | 高い |
| **パフォーマンス** | 高速（直接呼び出し） | やや低速（API経由） |
| **設計階層アクセス** | 不可 | 可能 |
| **コールバック** | 不可 | 可能 |
| **カスタムシステムタスク** | 不可 | 可能 |
| **信号の動的アクセス** | 限定的 | 完全対応 |
| **既存Cライブラリ連携** | 容易 | やや複雑 |
| **規格** | IEEE 1800 | IEEE 1364/1800 |

**DPI-Cを選ぶべき場合**:
- 既存のCライブラリ（暗号化、圧縮、数値計算など）を呼び出したい場合
- SystemVerilogとC間で単純なデータ受け渡しを行う場合
- パフォーマンスが最重要な場合

**VPIを選ぶべき場合**:
- シミュレーションの設計階層を動的に走査したい場合
- 信号値の変化にコールバックで反応したい場合
- カスタムシステムタスク（`$mytask`）を定義したい場合
- シミュレーションの制御（停止、強制値設定など）が必要な場合

実際のプロジェクトでは、両方を組み合わせて使うことも一般的です。たとえば、DPI-Cでリファレンスモデルを呼び出し、VPIでカスタムデバッグタスクを実装するといった構成が考えられます。

---

## 15.3 VPIの基本API

### 15.3.1 オブジェクトへのアクセス

VPIでオブジェクトにアクセスするための基本関数は3つあります。

**`vpi_handle`** — 1対1の関係でオブジェクトを取得します。

```c
// 親モジュールの取得
vpiHandle module = vpi_handle(vpiModule, net_handle);

// スコープの取得
vpiHandle scope = vpi_handle(vpiScope, task_handle);
```

**`vpi_handle_by_name`** — 名前を指定してオブジェクトを取得します。

```c
// 階層パス名でオブジェクトを取得
vpiHandle sig = vpi_handle_by_name("top.dut.u_ctrl.state", NULL);
if (sig == NULL) {
    vpi_printf("ERROR: 信号が見つかりません\n");
    return;
}
```

**`vpi_iterate` / `vpi_scan`** — 1対多の関係でオブジェクト群を走査します。

```c
// モジュール内のすべてのネットを列挙
vpiHandle iter = vpi_iterate(vpiNet, module_handle);
if (iter != NULL) {
    vpiHandle net;
    while ((net = vpi_scan(iter)) != NULL) {
        vpi_printf("  Net: %s\n", vpi_get_str(vpiFullName, net));
    }
    // vpi_scanがNULLを返した時点でイテレータは自動解放される
}
```

イテレータの使用における重要な注意点があります。

- `vpi_scan` が `NULL` を返すと、イテレータは自動的に解放されます
- ループを途中で抜ける場合は、`vpi_free_object(iter)` を呼んでイテレータを明示的に解放する必要があります

```c
// イテレータの途中解放が必要な例
vpiHandle iter = vpi_iterate(vpiReg, module);
if (iter != NULL) {
    vpiHandle reg;
    int count = 0;
    while ((reg = vpi_scan(iter)) != NULL) {
        count++;
        if (count >= 10) {
            vpi_free_object(iter);  // 必ず解放する
            break;
        }
    }
}
```

### 15.3.2 プロパティの取得

オブジェクトのプロパティを取得するための関数群を紹介します。

**`vpi_get`** — 整数プロパティを取得します。

```c
// オブジェクトタイプの取得
PLI_INT32 obj_type = vpi_get(vpiType, handle);

// ベクタのサイズ取得
PLI_INT32 size = vpi_get(vpiSize, net_handle);

// ネットのタイプ取得
PLI_INT32 net_type = vpi_get(vpiNetType, net_handle);

vpi_printf("Type: %d, Size: %d bits\n", obj_type, size);
```

**`vpi_get_str`** — 文字列プロパティを取得します。

```c
// オブジェクト名
const char *name = vpi_get_str(vpiName, handle);

// フルパス名
const char *full_name = vpi_get_str(vpiFullName, handle);

// ファイル名（定義された場所）
const char *file = vpi_get_str(vpiFile, handle);

vpi_printf("Name: %s, FullName: %s, File: %s\n", name, full_name, file);
```

**`vpi_get_value`** — 信号の現在値を取得します。

```c
// バイナリ形式で値を取得
s_vpi_value value;
value.format = vpiBinStrVal;
vpi_get_value(signal_handle, &value);
vpi_printf("Binary value: %s\n", value.value.str);

// 整数形式で値を取得
value.format = vpiIntVal;
vpi_get_value(signal_handle, &value);
vpi_printf("Integer value: %d\n", value.value.integer);

// 16進数形式で値を取得
value.format = vpiHexStrVal;
vpi_get_value(signal_handle, &value);
vpi_printf("Hex value: %s\n", value.value.str);
```

値のフォーマットは多数用意されています。

| フォーマット | 定数名 | 説明 |
|------------|--------|------|
| バイナリ文字列 | `vpiBinStrVal` | `"01xz"` 形式の文字列 |
| 8進数文字列 | `vpiOctStrVal` | 8進数形式の文字列 |
| 10進数文字列 | `vpiDecStrVal` | 10進数形式の文字列 |
| 16進数文字列 | `vpiHexStrVal` | 16進数形式の文字列 |
| 整数 | `vpiIntVal` | 32ビット整数 |
| 実数 | `vpiRealVal` | 倍精度浮動小数点 |
| スカラ | `vpiScalarVal` | `vpi0`, `vpi1`, `vpiX`, `vpiZ` |
| ベクタ | `vpiVectorVal` | `s_vpi_vecval` 配列 |
| 時間 | `vpiTimeVal` | `s_vpi_time` 構造体 |

### 15.3.3 値の設定

**`vpi_put_value`** を使って信号に値を設定できます。値の設定モードにはいくつかの種類があります。

```c
// 即座に値を設定（デポジット）
static void force_signal_value(vpiHandle sig, int val) {
    s_vpi_value value;
    s_vpi_time  time;

    value.format = vpiIntVal;
    value.value.integer = val;

    time.type = vpiSimTime;
    time.high = 0;
    time.low  = 0;

    // vpiNoDelay: 即時反映
    vpi_put_value(sig, &value, &time, vpiNoDelay);
}

// 遅延付きで値を設定
static void set_value_with_delay(vpiHandle sig, int val, int delay) {
    s_vpi_value value;
    s_vpi_time  time;

    value.format = vpiIntVal;
    value.value.integer = val;

    time.type = vpiSimTime;
    time.high = 0;
    time.low  = delay;

    // vpiInertialDelay: 慣性遅延で反映
    vpi_put_value(sig, &value, &time, vpiInertialDelay);
}

// フォース（強制代入）
static void force_signal(vpiHandle sig, int val) {
    s_vpi_value value;
    s_vpi_time  time;

    value.format = vpiIntVal;
    value.value.integer = val;
    time.type = vpiSimTime;
    time.high = 0;
    time.low  = 0;

    vpi_put_value(sig, &value, &time, vpiForceFlag);
}

// リリース（フォース解除）
static void release_signal(vpiHandle sig) {
    s_vpi_value value;
    s_vpi_time  time;

    value.format = vpiIntVal;
    time.type = vpiSimTime;
    time.high = 0;
    time.low  = 0;

    vpi_put_value(sig, &value, &time, vpiReleaseFlag);
}
```

値設定のフラグ一覧は以下の通りです。

| フラグ | 説明 |
|--------|------|
| `vpiNoDelay` | 即座に値を設定（0遅延） |
| `vpiInertialDelay` | 慣性遅延で値を設定 |
| `vpiTransportDelay` | 伝搬遅延で値を設定 |
| `vpiPureTransportDelay` | 純粋な伝搬遅延 |
| `vpiForceFlag` | 強制代入（`force` 文相当） |
| `vpiReleaseFlag` | 強制代入の解除（`release` 文相当） |

### 15.3.4 コールバックの登録

VPIのコールバック機構を使うと、特定のイベントが発生した際にC関数を自動的に呼び出すことができます。コールバックは `vpi_register_cb` で登録します。

```c
#include "vpi_user.h"

// 値変化コールバック関数
static int value_change_cb(p_cb_data cb_data) {
    s_vpi_value val;
    val.format = vpiHexStrVal;
    vpi_get_value(cb_data->obj, &val);

    s_vpi_time *time = cb_data->time;
    vpi_printf("[%0d] %s changed to %s\n",
        time->low,
        vpi_get_str(vpiFullName, cb_data->obj),
        val.value.str);
    return 0;
}

// 値変化コールバックの登録
static void register_value_change(vpiHandle signal) {
    s_cb_data   cb_data;
    s_vpi_time  time;
    s_vpi_value value;

    time.type     = vpiSimTime;
    value.format  = vpiHexStrVal;

    cb_data.reason    = cbValueChange;   // 値変化イベント
    cb_data.cb_rtn    = value_change_cb; // コールバック関数
    cb_data.obj       = signal;          // 監視対象
    cb_data.time      = &time;
    cb_data.value     = &value;
    cb_data.user_data = NULL;

    vpiHandle cb_handle = vpi_register_cb(&cb_data);
    if (cb_handle == NULL) {
        vpi_printf("ERROR: コールバック登録に失敗\n");
    }
}
```

主要なコールバック理由（`reason`）は以下の通りです。

| コールバック理由 | 定数名 | 説明 |
|----------------|--------|------|
| 値変化 | `cbValueChange` | 信号値が変化した時 |
| シミュレーション開始 | `cbStartOfSimulation` | シミュレーション開始時 |
| シミュレーション終了 | `cbEndOfSimulation` | シミュレーション終了時 |
| タイムステップ先頭 | `cbAtStartOfSimTime` | 各タイムステップの先頭 |
| Read-Writeフェーズ | `cbReadWriteSynch` | NBA領域での同期 |
| Read-Onlyフェーズ | `cbReadOnlySynch` | リアクティブ領域での同期 |
| 遅延後 | `cbAfterDelay` | 指定遅延後 |
| 次のタイムステップ | `cbNextSimTime` | 次のシミュレーション時刻 |

### 15.3.5 システムタスクの登録

VPIの最も一般的な用途の一つが、カスタムシステムタスクの定義です。`vpi_register_systf` を使って独自の `$mytask` を登録できます。

```c
#include "vpi_user.h"

// システムタスクの実装関数（calltf）
static int my_hello_calltf(char *user_data) {
    vpi_printf("Hello from VPI system task!\n");
    return 0;
}

// システムタスクの引数チェック関数（compiletf）
static int my_hello_compiletf(char *user_data) {
    // 引数がないことを確認
    vpiHandle systf_handle = vpi_handle(vpiSysTfCall, NULL);
    vpiHandle arg_iter = vpi_iterate(vpiArgument, systf_handle);
    if (arg_iter != NULL) {
        vpi_free_object(arg_iter);
        vpi_printf("ERROR: $my_hello はパラメータを取りません\n");
        vpi_control(vpiFinish, 1);
    }
    return 0;
}

// システムタスクの登録関数
static void register_my_hello(void) {
    s_vpi_systf_data systf_data;

    systf_data.type        = vpiSysTask;       // タスク（戻り値なし）
    systf_data.sysfunctype = 0;
    systf_data.tfname      = "$my_hello";       // タスク名
    systf_data.calltf      = my_hello_calltf;   // 実行関数
    systf_data.compiletf   = my_hello_compiletf;// コンパイル時チェック
    systf_data.sizetf      = NULL;
    systf_data.user_data   = NULL;

    vpi_register_systf(&systf_data);
}

// スタートアップルーチン配列
void (*vlog_startup_routines[])(void) = {
    register_my_hello,
    NULL  // 必ずNULLで終端
};
```

`s_vpi_systf_data` の主要フィールドは以下の通りです。

| フィールド | 説明 |
|-----------|------|
| `type` | `vpiSysTask`（タスク）または `vpiSysFunction`（関数） |
| `sysfunctype` | 関数の場合の戻り値型（`vpiIntFunc`, `vpiRealFunc` 等） |
| `tfname` | システムタスク/関数名（`$` で始まる） |
| `calltf` | 実行時に呼ばれる関数ポインタ |
| `compiletf` | コンパイル時チェック用の関数ポインタ |
| `sizetf` | 戻り値のビット幅を返す関数ポインタ（関数の場合） |
| `user_data` | ユーザ定義データへのポインタ |

---

## 15.4 VPIプログラミング実践

### 15.4.1 カスタムシステムタスクの作成

実用的なカスタムシステムタスクの例として、指定した信号のビット幅と現在値を表示する `$sig_info` タスクを作成します。

```c
#include <stdio.h>
#include <string.h>
#include "vpi_user.h"

// $sig_info の実装
static int sig_info_calltf(char *user_data) {
    vpiHandle systf_h, arg_iter, arg_h;
    s_vpi_value val;

    // 呼び出し元のハンドルを取得
    systf_h = vpi_handle(vpiSysTfCall, NULL);
    arg_iter = vpi_iterate(vpiArgument, systf_h);

    if (arg_iter == NULL) {
        vpi_printf("ERROR: $sig_info には引数が必要です\n");
        return 0;
    }

    // 各引数について情報を表示
    while ((arg_h = vpi_scan(arg_iter)) != NULL) {
        const char *name = vpi_get_str(vpiFullName, arg_h);
        PLI_INT32 type   = vpi_get(vpiType, arg_h);
        PLI_INT32 size   = vpi_get(vpiSize, arg_h);

        val.format = vpiHexStrVal;
        vpi_get_value(arg_h, &val);

        vpi_printf("--- Signal Info ---\n");
        vpi_printf("  Name : %s\n", name);
        vpi_printf("  Type : %d\n", type);
        vpi_printf("  Width: %d bits\n", size);
        vpi_printf("  Value: 0x%s\n", val.value.str);
        vpi_printf("-------------------\n");
    }

    return 0;
}

// コンパイル時チェック：最低1つの引数が必要
static int sig_info_compiletf(char *user_data) {
    vpiHandle systf_h = vpi_handle(vpiSysTfCall, NULL);
    vpiHandle arg_iter = vpi_iterate(vpiArgument, systf_h);
    if (arg_iter == NULL) {
        vpi_printf("COMPILE ERROR: $sig_info requires at least one argument\n");
        vpi_control(vpiFinish, 1);
    } else {
        vpi_free_object(arg_iter);
    }
    return 0;
}

static void register_sig_info(void) {
    s_vpi_systf_data data;
    data.type        = vpiSysTask;
    data.tfname      = "$sig_info";
    data.calltf      = sig_info_calltf;
    data.compiletf   = sig_info_compiletf;
    data.sizetf      = NULL;
    data.user_data   = NULL;
    vpi_register_systf(&data);
}
```

SystemVerilog側での使用例は以下の通りです。

```systemverilog
module test;
    logic [7:0]  data;
    logic [31:0] addr;
    logic        valid;

    initial begin
        data  = 8'hA5;
        addr  = 32'hDEAD_BEEF;
        valid = 1'b1;

        #10;
        $sig_info(data, addr, valid);
    end
endmodule
```

### 15.4.2 信号値のモニタリング

VPIのコールバック機構を使って、特定の信号の値変化を監視するモニタを実装します。

```c
#include "vpi_user.h"
#include <stdlib.h>
#include <string.h>

// モニタ用データ構造体
typedef struct {
    char signal_name[256];
    int  change_count;
    int  prev_value;
} monitor_data_t;

// 値変化コールバック
static int monitor_cb(p_cb_data cb_data) {
    monitor_data_t *mdata = (monitor_data_t *)cb_data->user_data;
    s_vpi_value val;
    s_vpi_time  time_s;

    // 現在時刻を取得
    time_s.type = vpiSimTime;
    vpi_get_time(cb_data->obj, &time_s);

    // 現在値を取得
    val.format = vpiIntVal;
    vpi_get_value(cb_data->obj, &val);

    mdata->change_count++;

    vpi_printf("[Time=%0d] %s: %0d -> %0d (変化回数: %0d)\n",
        time_s.low,
        mdata->signal_name,
        mdata->prev_value,
        val.value.integer,
        mdata->change_count);

    mdata->prev_value = val.value.integer;
    return 0;
}

// $vpi_monitor タスクの実装
static int vpi_monitor_calltf(char *user_data) {
    vpiHandle systf_h, arg_iter, sig_h;
    s_cb_data   cb;
    s_vpi_time  time_s;
    s_vpi_value val;

    systf_h  = vpi_handle(vpiSysTfCall, NULL);
    arg_iter = vpi_iterate(vpiArgument, systf_h);

    if (arg_iter == NULL) {
        vpi_printf("ERROR: $vpi_monitor requires a signal argument\n");
        return 0;
    }

    sig_h = vpi_scan(arg_iter);
    vpi_free_object(arg_iter);

    // モニタデータの初期化
    monitor_data_t *mdata = (monitor_data_t *)malloc(sizeof(monitor_data_t));
    strncpy(mdata->signal_name,
            vpi_get_str(vpiFullName, sig_h),
            sizeof(mdata->signal_name) - 1);
    mdata->change_count = 0;

    val.format = vpiIntVal;
    vpi_get_value(sig_h, &val);
    mdata->prev_value = val.value.integer;

    // コールバック登録
    time_s.type    = vpiSimTime;
    val.format     = vpiIntVal;

    cb.reason    = cbValueChange;
    cb.cb_rtn    = monitor_cb;
    cb.obj       = sig_h;
    cb.time      = &time_s;
    cb.value     = &val;
    cb.user_data = (char *)mdata;

    vpi_register_cb(&cb);

    vpi_printf("Monitor registered for %s\n", mdata->signal_name);
    return 0;
}
```

### 15.4.3 シミュレーション制御

VPIを使ってシミュレーションの制御を行うことができます。`vpi_control` 関数は、シミュレーションの停止や終了を要求します。

```c
#include "vpi_user.h"

// シミュレーション終了
static void finish_simulation(int return_code) {
    vpi_printf("シミュレーションを終了します (code=%d)\n", return_code);
    vpi_control(vpiFinish, return_code);
}

// シミュレーション一時停止
static void stop_simulation(void) {
    vpi_printf("シミュレーションを一時停止します\n");
    vpi_control(vpiStop, 0);
}

// 条件付き終了タスク：$check_and_finish(signal, expected)
static int check_and_finish_calltf(char *user_data) {
    vpiHandle systf_h, arg_iter, sig_h, exp_h;
    s_vpi_value sig_val, exp_val;

    systf_h  = vpi_handle(vpiSysTfCall, NULL);
    arg_iter = vpi_iterate(vpiArgument, systf_h);

    sig_h = vpi_scan(arg_iter);
    exp_h = vpi_scan(arg_iter);
    vpi_free_object(arg_iter);

    sig_val.format = vpiIntVal;
    vpi_get_value(sig_h, &sig_val);

    exp_val.format = vpiIntVal;
    vpi_get_value(exp_h, &exp_val);

    if (sig_val.value.integer != exp_val.value.integer) {
        vpi_printf("FAIL: %s = %0d, expected %0d\n",
            vpi_get_str(vpiFullName, sig_h),
            sig_val.value.integer,
            exp_val.value.integer);
        vpi_control(vpiFinish, 1);
    } else {
        vpi_printf("PASS: %s = %0d\n",
            vpi_get_str(vpiFullName, sig_h),
            sig_val.value.integer);
    }

    return 0;
}
```

また、シミュレーション時刻の取得も重要な機能です。

```c
// 現在のシミュレーション時刻を取得
static void print_sim_time(void) {
    s_vpi_time time_s;

    // シミュレーション時間
    time_s.type = vpiSimTime;
    vpi_get_time(NULL, &time_s);
    vpi_printf("Simulation time: %0d (high=%0d, low=%0d)\n",
        time_s.low, time_s.high, time_s.low);

    // スケールされた実数時間
    time_s.type = vpiScaledRealTime;
    vpi_get_time(NULL, &time_s);
    vpi_printf("Scaled real time: %f\n", time_s.real);
}
```

### 15.4.4 ネットリストの走査

VPIの強力な機能の一つが、設計階層全体を動的に走査できることです。以下は完全な設計階層ダンプの実装例です。

```c
#include "vpi_user.h"
#include <stdio.h>

// インデント用ヘルパー
static void print_indent(int depth) {
    for (int i = 0; i < depth; i++) {
        vpi_printf("  ");
    }
}

// ポート情報の表示
static void dump_ports(vpiHandle module, int depth) {
    vpiHandle iter = vpi_iterate(vpiPort, module);
    if (iter == NULL) return;

    vpiHandle port;
    while ((port = vpi_scan(iter)) != NULL) {
        PLI_INT32 direction = vpi_get(vpiDirection, port);
        const char *dir_str;
        switch (direction) {
            case vpiInput:  dir_str = "input";  break;
            case vpiOutput: dir_str = "output"; break;
            case vpiInout:  dir_str = "inout";  break;
            default:        dir_str = "unknown"; break;
        }
        print_indent(depth);
        vpi_printf("Port: %s [%s] (%d bits)\n",
            vpi_get_str(vpiName, port),
            dir_str,
            vpi_get(vpiSize, port));
    }
}

// ネット情報の表示
static void dump_nets(vpiHandle module, int depth) {
    vpiHandle iter = vpi_iterate(vpiNet, module);
    if (iter == NULL) return;

    vpiHandle net;
    while ((net = vpi_scan(iter)) != NULL) {
        s_vpi_value val;
        val.format = vpiHexStrVal;
        vpi_get_value(net, &val);

        print_indent(depth);
        vpi_printf("Net: %s [%d bits] = 0x%s\n",
            vpi_get_str(vpiName, net),
            vpi_get(vpiSize, net),
            val.value.str);
    }
}

// 再帰的な階層走査
static void dump_hierarchy(vpiHandle scope, int depth) {
    print_indent(depth);
    vpi_printf("Module: %s (def: %s)\n",
        vpi_get_str(vpiName, scope),
        vpi_get_str(vpiDefName, scope));

    dump_ports(scope, depth + 1);
    dump_nets(scope, depth + 1);

    // 子モジュールを再帰的に走査
    vpiHandle iter = vpi_iterate(vpiModule, scope);
    if (iter != NULL) {
        vpiHandle child;
        while ((child = vpi_scan(iter)) != NULL) {
            dump_hierarchy(child, depth + 1);
        }
    }
}

// $dump_design タスクの実装
static int dump_design_calltf(char *user_data) {
    vpiHandle top_iter = vpi_iterate(vpiModule, NULL);
    if (top_iter == NULL) {
        vpi_printf("ERROR: トップレベルモジュールが見つかりません\n");
        return 0;
    }

    vpi_printf("========== Design Hierarchy ==========\n");
    vpiHandle top;
    while ((top = vpi_scan(top_iter)) != NULL) {
        dump_hierarchy(top, 0);
    }
    vpi_printf("======================================\n");

    return 0;
}
```

### 15.4.5 VPIアプリケーションの登録

VPIアプリケーションをシミュレータに登録するには、`vlog_startup_routines` 配列を定義します。この配列は、シミュレータがVPIライブラリをロードする際に自動的に呼び出されます。

![VPIコールバックフロー](/images/systemverilog-complete-guide/ch15_vpi_callback.drawio.png)

```c
#include "vpi_user.h"

// 各システムタスクの登録関数（前節で定義済み）
extern void register_sig_info(void);
extern void register_vpi_monitor(void);
extern void register_dump_design(void);
extern void register_check_and_finish(void);

// スタートアップルーチン配列
// シミュレータはこの配列を走査し、各関数を順に呼び出す
void (*vlog_startup_routines[])(void) = {
    register_sig_info,
    register_vpi_monitor,
    register_dump_design,
    register_check_and_finish,
    NULL  // 必ずNULLで終端する
};
```

コンパイルとシミュレータへの登録方法はシミュレータによって異なります。主要なシミュレータでの手順は以下の通りです。

**VCS（Synopsys）の場合**:

```c
// コンパイル
// vcs -sverilog +vpi -load ./my_vpi.so design.sv testbench.sv

// Makefileの例
// VPI_FLAGS = +vpi -load ./my_vpi.so
// COMPILE   = vcs -sverilog $(VPI_FLAGS) -o simv
```

**Questa/ModelSim（Siemens EDA）の場合**:

```c
// コンパイルと実行
// gcc -shared -fPIC -o my_vpi.so my_vpi.c -I$(MTI_HOME)/include
// vsim -pli ./my_vpi.so work.test
```

**Xcelium（Cadence）の場合**:

```c
// コンパイルと実行
// xrun -sv -loadvpi ./my_vpi.so:vlog_startup_routines design.sv testbench.sv
```

