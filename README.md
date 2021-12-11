# Python をテストベンチにして検証する

DPI-C と MMAP を使ったプロセス間通信を使って、Python で記述したテストベンチから Verilog task を呼び出します。

動作モデルが Python で記述してあったり、入力データや期待値データが Python で扱いやすいときには便利です。あと PYNQ を使うなら、うまく書けば検証用パターンを制御ソフトにそのまま使えるかも！

## 実装例

### 概要

今回使う Verilog task は 5種類。`tb.v` 内に定義しています。

```verilog
    export "DPI-C" task v_init;
    export "DPI-C" task v_finish;
    export "DPI-C" task v_write;
    export "DPI-C" task v_send;
    export "DPI-C" task v_receive;
```

これらの task は `top.cpp` から DPI-C インタフェースで呼び出されます。

Python で記述したテストベンチから呼び出す関数は以下の関数が対応します。

```
1: c_init
2: c_finish
3: c_write
4: c_send
5: c_receive
```

`top.py` に記述した上記の関数を呼び出すと MMAP を使ったプロセス間通信で `top.cpp` の関数にデータを渡し、そこから `tb.v` の Verilog task が呼ばれます。

MMAP を使ったプロセス間通信は、今回独自に定義しました。

1バイト目はタスクの番号で、それ以降はタスク独自です。Python 側はタスクを呼ぶときにタスク番号をセットして、タスク終了時に C++ 側が 0 をセットするのを待ちます。

### 起動時

最初に Verilog シミュレーションを起動します。`tb.v` 内の `initial` ブロックから、`top.cpp` の `c_tb()` 関数を呼び出します。`c_tb` 関数は MMAP を開いて、タスク要求のポーリングを開始します。

準備ができたら、Python 側で MMAP を開いてテストを開始します。

### レジスタライト

例えば 3番のタスクは AXI スレーブライトです。

Python 側の `top.py` では 4バイト目からの 4バイトにアドレス、8バイト目からの 4バイトにデータをセットしてから、0バイト目にタスク番号の 3をセットします。

```python
def c_write(address, data):
    mm[4:8] = address.to_bytes(4, byteorder='little')
    mm[8:12] = data.to_bytes(4, byteorder='little')
    mm[0:1] = b"\3"
    while mm[0:1] != b'\0':
        pass
    return
```

C++ 側の `top.cpp` ではタスクの要求をポーリングして、タスク番号 3がセットされたら 4バイト目に書かれたアドレスと、8バイト目に書かれたデータを引数に `tb.v` のタスク `v_write` を呼び出します。タスクの実行が終了したら、0バイト目に 0 を書き込みます。

```c++
    while(1){
        if(buf[0] != 0){
            略
            else if(buf[0] == 3){
                union int_char address, data;
                for(int i=0; i<4; i++){
                    address.c[i] = buf[i+4];
                    data.c[i] = buf[i+8];
                }
                v_write(address.i, data.i);
            }
            略
            buf[0] = 0;
        }
    }
```

呼び出された Verilog task は検証対象のインタフェースを操作します。ここでは AXI スレーブライトです。

```verilog
    task v_write(input int address, input int data);
        S_AXI_AWADDR = address;
        S_AXI_WDATA = data;
        S_AXI_AWVALID = 'b1;
        S_AXI_WVALID = 'b1;
        repeat(1) @(posedge clk);
        S_AXI_AWVALID = 'b0;
        S_AXI_WVALID = 'b0;
        repeat(1) @(posedge clk);
    endtask
```

### ストリームリード

5番のタスクは AXI ストリームリードです。戻り値のある例です。

Python 側の `top.py` では 4バイト目からの 4バイトに転送サイズをセットしてから、0バイト目にタスク番号の 5をセットします。(でも使っているのは4バイトだけみたいです…)

タスクが完了する (0バイト目が 0になる) のを待って、4バイト目から先の領域から 4バイト/ワードのデータをサイズで指定されたワード数だけ list にコピーして返します。

```python
def c_receive(num):
    list = [0] * num
    mm[4:8] = num.to_bytes(4, byteorder='little')
    mm[0:1] = b"\5"
    while mm[0:1] != b'\0':
        pass
    list = []
    for i in range(int.from_bytes(mm[4:8], byteorder='little')):
        list.append(int.from_bytes(mm[4*i+8:4*i+12], byteorder='little'))
    return np.array(list)
```

C++ 側の `top.cpp` ではタスク番号 5がセットされたら 4バイト目に書かれたアドレスと、8バイト目に書かれたサイズを引数に `tb.v` のタスク `v_recieve` を呼び出します。タスクの実行が終了したら、4バイト目以降にデータを書き込んでから、0バイト目に 0 を書き込みます。

```c++
            else if(buf[0] == 5){
                int array[64];
                union int_char data, size;
                for(int i=0; i<4; i++){
                    size.c[i] = buf[i+4];
                }
                v_receive(array, size.i);
                for(int i=0; i<size.i; i++){
                    data.i = array[i];
                    for(int j=0; j<4; j++){
                        buf[i*4+j+8] = data.c[j];
                    }
                }
            }
```

呼び出された Verilog task は検証対象のインタフェースを操作します。ここでは AXI ストリームスレーブリードです。

```verilog
    task v_receive(output int data[64], input int size);
        while(M_AXIS_TVALID== 'b0)
            repeat(1) @(posedge clk);
        for(int i=0; i<size; i+=1)begin
            data[i] = M_AXIS_TDATA;
            repeat(1) @(posedge clk);
        end
        repeat(1) @(posedge clk);
    endtask
```

### 終了時

Verilog シミュレーション側は、c++ の `c_tb()` 関数がちゃんと終わってから終了したほうが良さそうな感じです。つまり、`v_finish()` タスク中で `$finish` しないほうが良さそうです。

## サンプルの実行

### 準備

Windows にインストールした ModelSim と Python 3 を WSL から使います。

- ModelSim をインストール
  - gcc は ModelSim 付属のものを使う (intelFPGA_pro/20.3/modelsim_ase/gcc-4.2.1-mingw32vc12/bin/ 的なやつ)
- Python をインストール
  - [ここ](https://pythonlinks.python.jp/ja/index.html) からダウンロードした
    - 32bit 版で動作確認したが 64bitでも動くと思う (多分…)
  - Windowsに追加するパス↓ (うちの環境の例です)
  - ```
    C:\Users\tom01\AppData\Local\Programs\Python\Python36-32\Scripts\;
    C:\Users\tom01\AppData\Local\Programs\Python\Python36-32\;
    C:\intelFPGA_pro\20.3\modelsim_ase\win32aloem;
    C:\intelFPGA_pro\20.3\modelsim_ase\gcc-4.2.1-mingw32vc12\bin;
    ```

### 実行

`sim` ディレクトリで `build.sh` を実行します。`[Sample N output] = [Matix Inpit] dot [sample N Input]T` を計算して返します。

Model-SIM インストールパスの修正が必要です。波形の確認は `vsim.exe -gui vsim.wlf`

- Verilog シミュレータ起動
  - tb.v 内の c_tb() 呼び出しで tb.cpp 内の c_tb() を実行
- tb.py を python コマンドから起動
  - 初期化 `v_init`
  - Python スクリプトで乱数で行列を作って `v_send`
  - 以下を繰り返す (2回)
    - Python スクリプトで乱数で行列を作って `v_send`
    - Verilog で行列乗算を計算して結果を `v_receive`
    - Python スクリプトで期待値を計算して先の値と比較
  - 終了 `v_finish`
