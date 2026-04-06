---
title: "付録"
---

# 付録

---

## A. 予約語一覧表

SystemVerilog IEEE 1800 規格で定義されている予約語（キーワード）の一覧を以下に示します。これらの識別子はユーザ定義の変数名やモジュール名として使用することはできません。

| 予約語 | 説明 |
|--------|------|
| `accept_on` | プロパティの受理条件を指定 |
| `alias` | ネット間の双方向接続を定義 |
| `always` | 継続的に実行されるプロセスブロック |
| `always_comb` | 組み合わせ回路用の常時実行ブロック |
| `always_ff` | フリップフロップ（順序回路）用の常時実行ブロック |
| `always_latch` | ラッチ用の常時実行ブロック |
| `and` | AND ゲートプリミティブ |
| `assert` | アサーション（検証用の条件チェック） |
| `assign` | 継続代入文 |
| `assume` | フォーマル検証における仮定条件 |
| `automatic` | 自動変数（スタック上に割り当て）を指定 |
| `before` | プログラムブロック間の実行順序を指定 |
| `begin` | シーケンシャルブロックの開始 |
| `bind` | 外部からモジュールインスタンスをバインド |
| `bins` | カバレッジビンの定義 |
| `binsof` | カバレッジビンの参照 |
| `bit` | 2状態1ビット型（0, 1） |
| `break` | ループからの脱出 |
| `buf` | バッファゲートプリミティブ |
| `bufif0` | 制御付きバッファ（制御信号が0で有効） |
| `bufif1` | 制御付きバッファ（制御信号が1で有効） |
| `byte` | 2状態8ビット符号付き整数型 |
| `case` | 条件分岐文（完全一致） |
| `casex` | 条件分岐文（x, z をドントケアとして扱う） |
| `casez` | 条件分岐文（z をドントケアとして扱う） |
| `cell` | ライブラリセルの指定 |
| `chandle` | C言語ポインタ型（DPI用） |
| `checker` | チェッカーの定義 |
| `class` | クラスの定義 |
| `clocking` | クロッキングブロックの定義 |
| `cmos` | CMOSゲートプリミティブ |
| `config` | コンフィグレーションブロックの定義 |
| `const` | 定数の宣言 |
| `constraint` | ランダム制約の定義 |
| `context` | DPIインポートのコンテキスト指定 |
| `continue` | ループの次の反復へスキップ |
| `cover` | カバレッジプロパティの定義 |
| `covergroup` | カバレッジグループの定義 |
| `coverpoint` | カバレッジポイントの定義 |
| `cross` | クロスカバレッジの定義 |
| `deassign` | 手続き的継続代入の解除 |
| `default` | case文等のデフォルト分岐 |
| `defparam` | パラメータ値の上書き（非推奨） |
| `design` | コンフィグレーション内のデザイン指定 |
| `disable` | タスクやブロックの無効化 |
| `dist` | ランダム制約における重み付き分布 |
| `do` | do-whileループの開始 |
| `edge` | 信号のエッジ（立ち上がり・立ち下がり両方） |
| `else` | if文の偽条件分岐 |
| `end` | シーケンシャルブロックの終了 |
| `endcase` | case文の終了 |
| `endchecker` | checkerブロックの終了 |
| `endclass` | classブロックの終了 |
| `endclocking` | clockingブロックの終了 |
| `endconfig` | configブロックの終了 |
| `endfunction` | functionの終了 |
| `endgenerate` | generateブロックの終了 |
| `endgroup` | covergroupの終了 |
| `endinterface` | interfaceの終了 |
| `endmodule` | moduleの終了 |
| `endpackage` | packageの終了 |
| `endprimitive` | primitiveの終了 |
| `endprogram` | programの終了 |
| `endproperty` | propertyの終了 |
| `endsequence` | sequenceの終了 |
| `endspecify` | specifyブロックの終了 |
| `endtable` | 真理値表の終了 |
| `endtask` | taskの終了 |
| `enum` | 列挙型の定義 |
| `event` | イベントデータ型 |
| `eventually` | プロパティが将来的に成立することを示す |
| `expect` | ブロッキングプロパティチェック |
| `export` | DPI関数のエクスポート、パッケージ項目の公開 |
| `extends` | クラスの継承 |
| `extern` | 外部定義されたメソッド・制約の宣言 |
| `final` | シミュレーション終了時に実行されるブロック |
| `first_match` | シーケンスの最初のマッチのみを採用 |
| `for` | forループ |
| `force` | 信号の強制代入 |
| `foreach` | 配列の全要素に対するループ |
| `forever` | 無限ループ |
| `fork` | 並列ブロックの開始 |
| `forkjoin` | fork-join（全スレッド完了まで待機） |
| `function` | 関数の定義 |
| `generate` | ジェネレートブロックの開始 |
| `genvar` | generate用の変数 |
| `global` | グローバルクロッキングの指定 |
| `highz0` | ドライブ強度（ハイインピーダンス0） |
| `highz1` | ドライブ強度（ハイインピーダンス1） |
| `if` | 条件分岐文 |
| `iff` | 条件付きイベント制御 |
| `ifnone` | specifyブロック内の条件なしパス |
| `ignore_bins` | 無視するカバレッジビン |
| `illegal_bins` | 不正なカバレッジビン |
| `implements` | インターフェースクラスの実装 |
| `implies` | プロパティの含意演算子 |
| `import` | パッケージからのインポート、DPI関数のインポート |
| `incdir` | インクルードディレクトリの指定 |
| `include` | ファイルのインクルード |
| `initial` | シミュレーション開始時に一度だけ実行されるブロック |
| `inout` | 双方向ポート宣言 |
| `input` | 入力ポート宣言 |
| `inside` | 集合メンバーシップ演算子 |
| `instance` | コンフィグレーション内のインスタンス指定 |
| `int` | 2状態32ビット符号付き整数型 |
| `integer` | 4状態32ビット符号付き整数型 |
| `interconnect` | インターコネクトネット型 |
| `interface` | インターフェースの定義 |
| `intersect` | シーケンスの交差 |
| `join` | 並列ブロックの終了（全スレッド待ち） |
| `join_any` | 並列ブロックの終了（いずれか1つのスレッド完了で継続） |
| `join_none` | 並列ブロックの終了（待機なし） |
| `large` | チャージ強度（大） |
| `let` | ローカルマクロ定義 |
| `liblist` | ライブラリリストの指定 |
| `library` | ライブラリの定義 |
| `local` | クラスメンバのアクセス制御（ローカル） |
| `localparam` | ローカルパラメータ（外部から変更不可） |
| `logic` | 4状態1ビット型（0, 1, x, z） |
| `longint` | 2状態64ビット符号付き整数型 |
| `macromodule` | モジュール定義（実装依存の最適化ヒント） |
| `matches` | パターンマッチング演算子 |
| `medium` | チャージ強度（中） |
| `modport` | インターフェースのポート方向定義 |
| `module` | モジュールの定義 |
| `nand` | NANDゲートプリミティブ |
| `negedge` | 信号の立ち下がりエッジ |
| `nettype` | ユーザ定義ネット型 |
| `new` | オブジェクトの生成（コンストラクタ呼び出し） |
| `nexttime` | 次のタイムステップでのプロパティ成立 |
| `nmos` | NMOSゲートプリミティブ |
| `nor` | NORゲートプリミティブ |
| `noshowcancelled` | キャンセルされたパルスを非表示にする |
| `not` | NOTゲートプリミティブ |
| `notif0` | 制御付きNOTゲート（制御信号が0で有効） |
| `notif1` | 制御付きNOTゲート（制御信号が1で有効） |
| `null` | ヌル値（オブジェクトハンドルの無効値） |
| `or` | ORゲートプリミティブ |
| `output` | 出力ポート宣言 |
| `package` | パッケージの定義 |
| `packed` | パックド配列・構造体の指定 |
| `parameter` | パラメータの定義 |
| `pmos` | PMOSゲートプリミティブ |
| `posedge` | 信号の立ち上がりエッジ |
| `primitive` | UDPプリミティブの定義 |
| `priority` | 優先度付きcase/if文 |
| `program` | テストベンチプログラムの定義 |
| `property` | プロパティの定義 |
| `protected` | クラスメンバのアクセス制御（派生クラスからアクセス可） |
| `pull0` | プル強度（0側） |
| `pull1` | プル強度（1側） |
| `pulldown` | プルダウン抵抗 |
| `pullup` | プルアップ抵抗 |
| `pulsestyle_ondetect` | パルス検出時のスタイル指定 |
| `pulsestyle_onevent` | パルスイベント時のスタイル指定 |
| `pure` | 純粋仮想メソッドの宣言 |
| `rand` | ランダム変数の宣言 |
| `randc` | 巡回ランダム変数の宣言 |
| `randcase` | ランダム条件分岐 |
| `randsequence` | ランダムシーケンスの生成 |
| `rcmos` | 抵抗付きCMOSゲート |
| `real` | 倍精度浮動小数点型 |
| `realtime` | 実数型の時間変数 |
| `ref` | 参照渡しポート宣言 |
| `reg` | 4状態変数型（Verilog由来、SystemVerilogではlogicの使用を推奨） |
| `reject_on` | プロパティの拒否条件を指定 |
| `release` | force文の解除 |
| `repeat` | 指定回数の繰り返し |
| `restrict` | フォーマル検証における制限条件 |
| `return` | 関数・タスクからの復帰 |
| `rnmos` | 抵抗付きNMOSゲート |
| `rpmos` | 抵抗付きPMOSゲート |
| `rtran` | 抵抗付き双方向スイッチ |
| `rtranif0` | 抵抗付き制御スイッチ（制御信号が0で有効） |
| `rtranif1` | 抵抗付き制御スイッチ（制御信号が1で有効） |
| `s_always` | 強いalwaysプロパティ演算子 |
| `s_eventually` | 強いeventuallyプロパティ演算子 |
| `s_nexttime` | 強いnexttimeプロパティ演算子 |
| `s_until` | 強いuntilプロパティ演算子 |
| `s_until_with` | 強いuntil_withプロパティ演算子 |
| `scalared` | スカラ分解可能なベクタ宣言 |
| `sequence` | シーケンスの定義 |
| `shortint` | 2状態16ビット符号付き整数型 |
| `shortreal` | 単精度浮動小数点型 |
| `showcancelled` | キャンセルされたパルスを表示する |
| `signed` | 符号付き型の指定 |
| `small` | チャージ強度（小） |
| `soft` | ソフト制約の指定 |
| `solve` | 制約解決の優先順位指定 |
| `specify` | タイミング指定ブロックの開始 |
| `specparam` | specify内のパラメータ |
| `static` | 静的変数（モジュールスコープに割り当て）を指定 |
| `string` | 文字列型 |
| `strong` | 強いプロパティ演算子 |
| `strong0` | ドライブ強度（強い0） |
| `strong1` | ドライブ強度（強い1） |
| `struct` | 構造体の定義 |
| `super` | 親クラスへの参照 |
| `supply0` | 電源ネット（GND） |
| `supply1` | 電源ネット（VDD） |
| `sync_accept_on` | 同期的プロパティ受理条件 |
| `sync_reject_on` | 同期的プロパティ拒否条件 |
| `table` | UDPの真理値表 |
| `tagged` | タグ付きユニオンの指定 |
| `task` | タスクの定義 |
| `this` | 現在のオブジェクトインスタンスへの参照 |
| `throughout` | シーケンス全体にわたる条件指定 |
| `time` | 64ビット符号なし整数型（時間用） |
| `timeprecision` | 時間精度の指定 |
| `timeunit` | 時間単位の指定 |
| `tran` | 双方向スイッチ |
| `tranif0` | 制御付き双方向スイッチ（制御信号が0で有効） |
| `tranif1` | 制御付き双方向スイッチ（制御信号が1で有効） |
| `tri` | トライステートネット |
| `tri0` | プルダウン付きトライステートネット |
| `tri1` | プルアップ付きトライステートネット |
| `triand` | ワイヤードANDトライステートネット |
| `trior` | ワイヤードORトライステートネット |
| `trireg` | 電荷保持型トライステートネット |
| `type` | 型パラメータの指定 |
| `typedef` | 型の別名定義 |
| `union` | 共用体の定義 |
| `unique` | 一意性保証付きcase/if文 |
| `unique0` | 一意性保証付きcase文（一致なしも許容） |
| `unsigned` | 符号なし型の指定 |
| `until` | プロパティのuntil演算子 |
| `until_with` | プロパティのuntil_with演算子 |
| `untyped` | 型なしパラメータ |
| `use` | コンフィグレーション内のセル使用指定 |
| `uwire` | 単一ドライバネット |
| `var` | 変数の明示的宣言 |
| `vectored` | ベクタとして扱うネット宣言 |
| `virtual` | 仮想インターフェース・仮想メソッドの宣言 |
| `void` | 戻り値なし型 |
| `wait` | レベルセンシティブなイベント待ち |
| `wait_order` | 指定順序でのイベント待ち |
| `wand` | ワイヤードANDネット |
| `weak` | 弱いプロパティ演算子 |
| `weak0` | ドライブ強度（弱い0） |
| `weak1` | ドライブ強度（弱い1） |
| `while` | whileループ |
| `wildcard` | ワイルドカード等価演算子の有効化 |
| `wire` | ワイヤネット型 |
| `with` | 配列操作メソッドの条件式、パターンマッチ |
| `within` | シーケンスの包含関係 |
| `wor` | ワイヤードORネット |
| `xnor` | XNORゲートプリミティブ |
| `xor` | XORゲートプリミティブ |

---

## B. 標準システムタスク・関数リファレンス

SystemVerilog で使用可能な主要なシステムタスクおよびシステム関数をカテゴリ別に整理します。システムタスクは `$` で始まる名前を持ち、シミュレータが提供する組み込み機能です。

### B.1 表示系

| タスク/関数 | 構文 | 説明 |
|-------------|------|------|
| `$display` | `$display(format, args...)` | フォーマット付きメッセージを表示し、末尾に改行を付加する |
| `$write` | `$write(format, args...)` | フォーマット付きメッセージを表示する（改行なし） |
| `$strobe` | `$strobe(format, args...)` | 現在のタイムステップの最後にメッセージを表示する |
| `$monitor` | `$monitor(format, args...)` | 引数の値が変化するたびにメッセージを表示する |
| `$displayb` | `$displayb(format, args...)` | デフォルトの基数を2進数にして表示する |
| `$displayh` | `$displayh(format, args...)` | デフォルトの基数を16進数にして表示する |
| `$displayo` | `$displayo(format, args...)` | デフォルトの基数を8進数にして表示する |

### B.2 ファイル入出力系

| タスク/関数 | 構文 | 説明 |
|-------------|------|------|
| `$fopen` | `fd = $fopen(filename, mode)` | ファイルを開き、ファイル記述子を返す |
| `$fclose` | `$fclose(fd)` | ファイルを閉じる |
| `$fwrite` | `$fwrite(fd, format, args...)` | ファイルにフォーマット付きで書き込む |
| `$fread` | `$fread(variable, fd)` | ファイルからバイナリデータを読み込む |
| `$fscanf` | `n = $fscanf(fd, format, args...)` | ファイルからフォーマット付きで読み込む |
| `$fgets` | `n = $fgets(str, fd)` | ファイルから1行を読み込む |
| `$feof` | `result = $feof(fd)` | ファイル終端に達したかどうかを返す |

### B.3 シミュレーション制御系

| タスク/関数 | 構文 | 説明 |
|-------------|------|------|
| `$finish` | `$finish(n)` | シミュレーションを終了する。引数で表示レベルを指定 |
| `$stop` | `$stop(n)` | シミュレーションを一時停止する |
| `$exit` | `$exit` | programブロックの実行を終了する |

### B.4 時間系

| タスク/関数 | 構文 | 説明 |
|-------------|------|------|
| `$time` | `t = $time` | 現在のシミュレーション時間を64ビット整数で返す |
| `$stime` | `t = $stime` | 現在のシミュレーション時間を32ビット整数で返す |
| `$realtime` | `t = $realtime` | 現在のシミュレーション時間を実数で返す |
| `$timeformat` | `$timeformat(unit, prec, suffix, width)` | 時間表示のフォーマットを設定する |

### B.5 数学系

| タスク/関数 | 構文 | 説明 |
|-------------|------|------|
| `$clog2` | `n = $clog2(x)` | 天井関数付き2を底とする対数（アドレス幅の計算に有用） |
| `$ln` | `r = $ln(x)` | 自然対数を返す |
| `$log10` | `r = $log10(x)` | 常用対数を返す |
| `$exp` | `r = $exp(x)` | eのx乗を返す |
| `$pow` | `r = $pow(x, y)` | xのy乗を返す |
| `$sqrt` | `r = $sqrt(x)` | 平方根を返す |
| `$floor` | `r = $floor(x)` | x以下の最大整数を返す |
| `$ceil` | `r = $ceil(x)` | x以上の最小整数を返す |

### B.6 変換系

| タスク/関数 | 構文 | 説明 |
|-------------|------|------|
| `$signed` | `val = $signed(x)` | 値を符号付きとして解釈する |
| `$unsigned` | `val = $unsigned(x)` | 値を符号なしとして解釈する |
| `$cast` | `$cast(dest, src)` | 動的型キャスト（成功時1、失敗時0を返す） |
| `$bits` | `n = $bits(expr)` | 式のビット幅を返す |
| `$typename` | `s = $typename(expr)` | 式の型名を文字列で返す |

### B.7 配列系

| タスク/関数 | 構文 | 説明 |
|-------------|------|------|
| `$dimensions` | `n = $dimensions(array)` | 配列の次元数を返す |
| `$left` | `n = $left(array, dim)` | 指定次元の左側インデックスを返す |
| `$right` | `n = $right(array, dim)` | 指定次元の右側インデックスを返す |
| `$low` | `n = $low(array, dim)` | 指定次元の最小インデックスを返す |
| `$high` | `n = $high(array, dim)` | 指定次元の最大インデックスを返す |
| `$increment` | `n = $increment(array, dim)` | 指定次元の増分方向を返す（1または-1） |
| `$size` | `n = $size(array, dim)` | 指定次元の要素数を返す |
| `$unpacked_dimensions` | `n = $unpacked_dimensions(array)` | アンパックド次元数を返す |

### B.8 ランダム系

| タスク/関数 | 構文 | 説明 |
|-------------|------|------|
| `$random` | `val = $random(seed)` | 32ビット符号付き疑似乱数を返す |
| `$urandom` | `val = $urandom(seed)` | 32ビット符号なし疑似乱数を返す |
| `$urandom_range` | `val = $urandom_range(max, min)` | 指定範囲内の符号なし疑似乱数を返す |

### B.9 メモリ読み込み系

| タスク/関数 | 構文 | 説明 |
|-------------|------|------|
| `$readmemb` | `$readmemb(file, mem, start, end)` | 2進テキストファイルからメモリ配列に読み込む |
| `$readmemh` | `$readmemh(file, mem, start, end)` | 16進テキストファイルからメモリ配列に読み込む |
| `$writememb` | `$writememb(file, mem, start, end)` | メモリ配列を2進テキストファイルに書き出す |
| `$writememh` | `$writememh(file, mem, start, end)` | メモリ配列を16進テキストファイルに書き出す |

### B.10 波形ダンプ系

| タスク/関数 | 構文 | 説明 |
|-------------|------|------|
| `$dumpfile` | `$dumpfile(filename)` | VCDダンプファイル名を指定する |
| `$dumpvars` | `$dumpvars(levels, scope...)` | ダンプ対象の変数を指定する |
| `$dumpoff` | `$dumpoff` | 波形ダンプを一時停止する |
| `$dumpon` | `$dumpon` | 波形ダンプを再開する |
| `$dumpall` | `$dumpall` | 全変数の現在値をダンプする |
| `$dumpflush` | `$dumpflush` | ダンプバッファをファイルにフラッシュする |

### B.11 アサーション系

| タスク/関数 | 構文 | 説明 |
|-------------|------|------|
| `$fatal` | `$fatal(level, format, args...)` | 致命的エラーメッセージを出力しシミュレーションを終了する |
| `$error` | `$error(format, args...)` | エラーメッセージを出力する |
| `$warning` | `$warning(format, args...)` | 警告メッセージを出力する |
| `$info` | `$info(format, args...)` | 情報メッセージを出力する |

### B.12 型操作系

| タスク/関数 | 構文 | 説明 |
|-------------|------|------|
| `$isunbounded` | `result = $isunbounded(expr)` | パラメータ値が`$`（無限大）かどうかを返す |

---

## C. 演算子優先順位表

SystemVerilog の演算子を優先度の高い順に以下の表に示します。優先度が同じ演算子はグループとしてまとめています。結合規則は、同じ優先度の演算子が複数並んだ場合にどちらから評価するかを示します。

| 優先度 | 演算子 | 名称 | 結合規則 |
|--------|--------|------|----------|
| 1（最高） | `()` | 括弧 | - |
| 2 | `::` | スコープ解決演算子 | 左から右 |
| 3 | `[]` | ビット選択・配列インデックス | 左から右 |
| 3 | `[:]` `[+:]` `[-:]` | パート選択 | 左から右 |
| 4 | `.` | メンバアクセス | 左から右 |
| 5 | `()` | 関数・メソッド呼び出し | 左から右 |
| 6 | `!` `~` `&` `|` `^` `~&` `~|` `~^` `^~` | 単項演算子（論理NOT、ビットNOT、縮約演算） | 右から左 |
| 6 | `+` `-` | 単項プラス・マイナス | 右から左 |
| 7 | `++` `--` | インクリメント・デクリメント | - |
| 8 | `**` | べき乗 | 左から右 |
| 9 | `*` `/` `%` | 乗算・除算・剰余 | 左から右 |
| 10 | `+` `-` | 加算・減算 | 左から右 |
| 11 | `<<` `>>` `<<<` `>>>` | 論理シフト・算術シフト | 左から右 |
| 12 | `<` `<=` `>` `>=` | 比較演算子 | 左から右 |
| 12 | `inside` | 集合メンバーシップ | 左から右 |
| 13 | `==` `!=` | 論理等価・不等価 | 左から右 |
| 13 | `===` `!==` | ケース等価・不等価（x, zも比較） | 左から右 |
| 13 | `==?` `!=?` | ワイルドカード等価・不等価 | 左から右 |
| 14 | `&` | ビットAND | 左から右 |
| 15 | `^` `~^` `^~` | ビットXOR・XNOR | 左から右 |
| 16 | `\|` | ビットOR | 左から右 |
| 17 | `&&` | 論理AND | 左から右 |
| 18 | `\|\|` | 論理OR | 左から右 |
| 19 | `?:` | 三項条件演算子 | 右から左 |
| 20 | `->` | イベントトリガ | 右から左 |
| 20 | `->>` | ノンブロッキングイベントトリガ | 右から左 |
| 21 | `=` | 代入 | 右から左 |
| 21 | `+=` `-=` `*=` `/=` `%=` | 複合代入演算子 | 右から左 |
| 21 | `&=` `\|=` `^=` | ビット複合代入演算子 | 右から左 |
| 21 | `<<=` `>>=` `<<<=` `>>>=` | シフト複合代入演算子 | 右から左 |
| 22（最低） | `{}` `{{}}` | 連結・繰り返し連結 | - |

---

## D. 逆引きインデックス

SystemVerilog で「こういうことをしたいが、どう書けばよいか」という疑問に対して、目的から構文や機能を引けるように整理しました。各項目は「やりたいこと」と「使用する構文・機能」の対応を示します。

| # | やりたいこと | 使用する構文・機能 | 記述例 |
|---|-------------|-------------------|--------|
| 1 | ビット幅を取得したい | `$bits()` | `$bits(data)` で変数 `data` のビット幅を取得 |
| 2 | 配列をソートしたい | `.sort()` メソッド | `array.sort()` で昇順ソート |
| 3 | ステートマシンを作りたい | `enum` + `always_ff` | `enum` で状態を定義し、`always_ff` で遷移を記述 |
| 4 | クロック立ち上がりで動作する回路を書きたい | `always_ff @(posedge clk)` | `always_ff @(posedge clk) begin ... end` |
| 5 | 組み合わせ回路を書きたい | `always_comb` | `always_comb begin ... end` |
| 6 | パラメータでビット幅を可変にしたい | `parameter` + `localparam` | `module m #(parameter WIDTH=8)(...);` |
| 7 | 2次元配列を宣言したい | パックド/アンパックド配列 | `logic [7:0] mem [0:255];` |
| 8 | 複数の信号をまとめたい | `struct packed` | `typedef struct packed { logic [7:0] a, b; } my_t;` |
| 9 | 動的配列を使いたい | 動的配列 `[]` | `int dyn_arr[]; dyn_arr = new[10];` |
| 10 | 連想配列を使いたい | 連想配列 `[key_type]` | `int assoc[string]; assoc["key"] = 42;` |
| 11 | キューを使いたい | キュー `[$]` | `int q[$]; q.push_back(1); q.pop_front();` |
| 12 | 文字列を整数に変換したい | `.atoi()` メソッド | `int n = str.atoi();` |
| 13 | 特定の条件を満たす配列要素を抽出したい | `.find_with()` / `find()` | `result = array.find(x) with (x > 10);` |
| 14 | ランダムテストを書きたい | `class` + `rand` + `constraint` | `rand bit [7:0] data; constraint c { data < 100; }` |
| 15 | インターフェースを使ってポートを簡素化したい | `interface` + `modport` | `interface bus_if; ... modport master(...); endinterface` |
| 16 | 別モジュールの内部信号を参照したい | 階層パス参照 | `top.sub_inst.internal_sig` |
| 17 | テストベンチからメモリに初期値を読み込みたい | `$readmemh` / `$readmemb` | `$readmemh("data.hex", mem_array);` |
| 18 | 波形をファイルに保存したい | `$dumpfile` + `$dumpvars` | `$dumpfile("wave.vcd"); $dumpvars(0, top);` |
| 19 | アドレスに必要なビット幅を計算したい | `$clog2()` | `localparam ADDR_W = $clog2(DEPTH);` |
| 20 | 信号の立ち上がり/立ち下がりの両方を検出したい | `edge` / `posedge or negedge` | `always_ff @(posedge clk or negedge rst_n)` |
| 21 | generate文で繰り返しインスタンスを生成したい | `generate` + `for` + `genvar` | `genvar i; generate for (i=0; i<N; i++) begin : gen ... end endgenerate` |
| 22 | 列挙型の全値をループで処理したい | `.first()` `.last()` `.next()` | `for (state_t s = s.first; s != s.last; s = s.next) ...` |
| 23 | パッケージで定数や型を共有したい | `package` + `import` | `package pkg; ... endpackage` / `import pkg::*;` |
| 24 | 非同期リセット付きフリップフロップを書きたい | `always_ff` + `negedge rst_n` | `always_ff @(posedge clk or negedge rst_n) if (!rst_n) ... else ...` |
| 25 | 条件に応じてハードウェアを生成したい | `generate` + `if` | `generate if (MODE == 1) begin : gen_a ... end endgenerate` |
| 26 | 型をパラメータ化したい | `parameter type` | `module m #(parameter type T = logic [7:0])(...);` |
| 27 | fork-joinで並列処理を書きたい | `fork` / `join` / `join_any` / `join_none` | `fork begin ... end begin ... end join` |
| 28 | 配列の最大値・最小値を求めたい | `.max()` / `.min()` メソッド | `max_val = array.max(); min_val = array.min();` |
| 29 | 配列の要素数を取得したい | `$size()` / `.size()` | `$size(array)` または `array.size()` |
| 30 | 配列をシャッフルしたい | `.shuffle()` メソッド | `array.shuffle();` |
| 31 | クラスの継承を行いたい | `extends` | `class Child extends Parent; ... endclass` |
| 32 | 仮想メソッドでポリモーフィズムを実現したい | `virtual` メソッド | `virtual function void do_something(); ... endfunction` |
| 33 | 値の範囲をチェックしたい | `inside` 演算子 | `if (val inside {[0:100]}) ...` |
| 34 | 設計のアサーションチェックを入れたい | `assert property` | `assert property (@(posedge clk) req |-> ##[1:3] ack);` |
| 35 | カバレッジを収集したい | `covergroup` + `coverpoint` | `covergroup cg @(posedge clk); coverpoint data; endgroup` |
| 36 | 配列を逆順にしたい | `.reverse()` メソッド | `array.reverse();` |
| 37 | 特定の条件で配列要素を合計したい | `.sum()` メソッド + `with` | `total = array.sum() with (item > 0 ? item : 0);` |
| 38 | 繰り返し連結でパターンを生成したい | `{n{pattern}}` | `assign bus = {4{8'hFF}};` で32ビットの全1を生成 |
| 39 | シミュレーション時間を表示したい | `$time` / `$realtime` | `$display("Time=%0t", $time);` |
| 40 | ユニークな分岐を保証したい | `unique case` / `priority case` | `unique case (sel) ... endcase` |

---

> 本付録は SystemVerilog IEEE Std 1800-2017 に基づいて作成しています。シミュレータや合成ツールによっては、一部の機能がサポートされていない場合があります。実際の使用時には各ツールのドキュメントも合わせてご参照ください。
