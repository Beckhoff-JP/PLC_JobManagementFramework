# TwinCAT PLC ジョブフレームワーク

PLC上で様々な制御上のアクティビティを定義するフレームワークです。フレームワークが提供する `FB_Executor` と呼ばれるジョブ実行エンジンによって、ユーザが `InterfaceFuture`インターフェースを実装した `Future` と呼ばれる単位のオブジェクトを実行します。`Future` では3つのメソッドを実装する必要があり、ジョブ実行エンジンが次図のような時系列で定義したFutureを順次実行します。

![](https://beckhoff-jp.github.io/TwinCATHowTo/_images/task_time_chart.png)

|メソッド|説明|終了条件|
|-|-|-|
|init()|オブジェクト内部の変数を初期化します。|init()の戻り値がTRUEとなる|
|execute()|実際に処理を定義します。|execute()の戻り値がTRUEとなる|
|quit()|処理後の処理を定義します。|quit()の戻り値がTRUEとなる|

この `FB_Executor`と`Future`オブジェクトが組み合わせたものを「ジョブ」と呼びます。ジョブを動作の基本単位として、複数のジョブを組み合わせたものを「コンテナ」と呼びます。

コンテナには、実行方法に応じて次の4種類が用意されています。

|コンテナ名|説明|ContainerType|
|---|---|---|
|FB_BatchJobContainer|あらかじめ登録したジョブを順次実行するコンテナです。|BATCH|
|FB_ParallelJobContainer|あらかじめ登録したジョブを同時並列で実行するコンテナです。|PARALLEL|
|FB_QueueJobContainer|順次実行するコンテナですが、実行中も随時ジョブを登録することができます。|QUEUE|
|FB_ParallelQueueJobContainer|同時並列で実行するコンテナですが、実行中も随時ジョブを登録することができます。|PARALLEL_QUEUE|

このコンテナを用いることで、複数のジョブを順次実行、または、並列実行させることができます。また、コンテナ自身も `Future` を実装したものとなっています。したがって、次図のとおり入れ子状のジョブ構成にすることが可能です。

このように複雑な階層のジョブを構成することができます。

![](https://beckhoff-jp.github.io/TwinCATHowTo/_images/sample_code_job_structure.png)

## ライブラリ化とライブラリインストール

本プロジェクトは、以下の手順でライブラリ化ができます。さまざまなTwinCATプロジェクトに対して本ライブラリをロードする事で汎用的にフレームワークを使った実装が可能となります。

1. 以下のTwinCATプロジェクトをGithubリポジトリからクローンしてください。

    [https://github.com/Beckhoff-JP/PLC_JobManagementFramework](https://github.com/Beckhoff-JP/PLC_JobManagementFramework)

2. TwinCAT XAEでプロジェクトを開き、[ライブラリ作成手順](https://beckhoff-jp.github.io/TwinCATHowTo/library/make_library.html#twincat)に従ってライブラリファイル`JobManagement.library`を保存してください。

3. [こちらの手順に従い](https://beckhoff-jp.github.io/TwinCATHowTo/library/use_library.html#section-use-library)目的のTwinCATのPLCプロジェクトに保存したライブラリをインストール・追加します。

# フレームワークを用いた実装マニュアル

まず、フレームワークを活用する開発者は、 `Future` と呼ばれる動作の最小単位を定義します。定義した Future オブジェクトを組み合わせてジョブを生成し、さまざまな複雑な制御を行います。

本フレームワークにはサンプルとして NC PTP を利用してモーション制御を行う基本的な Future を包含しています。このうち、`MC_MoveAbsolute` の軸動作をFuture化した `FutureMCMoveAbsolute` Futureオブジェクトを中心に、ジョブフレームワーク全般の使い方を説明します。

## 動作環境

下記の環境で構築しています。

* TwinCAT Build 4026

また、同梱しているサンプルモデルである、NC PTPによる各種モーション動作の Future を実装するには以下のWorkloadsやパッケージが必要です。

* モーション機能 TF5000 NC PTP (Workload)
* ユーザモードランタイム TF1700 (Workload)
* NCPTP用ユーザモードランタイムドライバ TwinCAT.XARUM.NCPTP (Package)

## Futureの新規作成

Futureとは、個々の動作の最小単位を定義するファンクションブロックです。最も基本となる型は `Interfacefuture` ですが、必要なメソッドだけをオーバライドすれば良いように、抽象メソッド `FB_AbstructFuture` が用意されています。次図のとおり継承したファンクションブロックを作成してください。

![](https://beckhoff-jp.github.io/TwinCATHowTo/_images/2024-07-06-18-25-53.png)


1. 作成したファンクションブロックに``{attribute `enable_dynamic_creation`}``を追加します。これは、動的にファンクションブロックを生成できるようにするためのおまじないです。また、内部変数としてこのファンクションブロックの機能に必要なインスタンス変数を定義してください。習慣的に内部変数にはアンダースコア`_`を付加しておき、アクセッサのシンボルと区別しておくと良いでしょう。
    
    ``` pascal
    {attribute 'enable_dynamic_creation'} // 動的なインスタンス生成を有効にする
    FUNCTION_BLOCK FutureMCMoveAbsolute EXTENDS FB_AbstructFuture
    VAR_INPUT
    END_VAR
    VAR_OUTPUT
    END_VAR
    VAR
        _fb_move_abs    : MC_MoveAbsolute;
        _mc_stop        : MC_Stop;
        _mc_reset       : MC_Reset;
        _abort_executor : FB_Executor;
        _axis           : REFERENCE TO AXIS_REF;
        _position       : MC_LREAL;
        _velocity       : MC_LREAL;
        _acceleration   : MC_LREAL;
        _deceleration   : MC_LREAL;
        _jerk           : MC_LREAL;
        _option         : ST_MoveOptions;
        _buffer_mode    : MC_BufferMode ;
        _error_id       : UDINT;
        _exec_state     : UDINT;
        _abort_state    : UINT;
    END_VAR
    ```

2. 内部変数に対する独自のアクセッサを用意します。入出力を含む場合、参照型（`REFERENCE TO`）にしてください。

    ``` pascal
    METHOD set_parameters : BOOL
    VAR_INPUT
        axis            : REFERENCE TO AXIS_REF;
        position        : MC_LREAL;
        velocity        : MC_LREAL;
        acceleration    : MC_LREAL;     // Set deceleration in case let moving axis halt.
        deceleration    : MC_LREAL;     // Set deceleration in case let moving axis halt.
        jerk            : MC_LREAL;     // Set jerk in case let moving axis halt.
        buffer_mode     : MC_BufferMode;
    END_VAR

    _axis REF= axis;        // 参照型の代入には REF= を使用します。
    _position := position;
    _velocity := velocity;
    _acceleration := acceleration;
    _deceleration := deceleration;
    _jerk := jerk;
    _buffer_mode := buffer_mode;
    ```

3. 最低限オーバライドする `init()`と`execute()` メソッドを再定義します。MethodをAddし、Name欄の選択肢にはオーバライド可能なメソッドが一覧されますので、実装したい任意のメソッドを選びます。

    ![](https://beckhoff-jp.github.io/TwinCATHowTo/_images/2024-06-26-18-59-57.png)


    各メソッドの完了時はいずれもBOOL型の戻り値をTRUEにしてください。

    ``` pascal
    : 
    IF <終了条件> THEN
        execute := TRUE
    ELSE
        execute := FALSE
    END_IF
    ```

    * `init()` のオーバライド

      使用する変数の初期化等を行う
        ``` pascal
        METHOD init : BOOL

        _exec_state := 0;
        _abort_state := 0;
        _error_id := _fb_move_abs.ErrorID;
        init := TRUE;
        ```

    * `execute()` のオーバライド
    
      `MC_MoveAbsolute`を使った実制御を定義
        ``` pascal
        METHOD execute : BOOL

        CASE _exec_state OF
            0:
                _abort_state := 0;
                _fb_move_abs(Axis := _axis,Execute := FALSE);
                _fb_move_abs(
                    Execute := TRUE,
                    Axis := _axis,
                    Position := _position,
                    Velocity := _velocity,
                    Acceleration := _acceleration,
                    Deceleration := _deceleration,
                    Jerk := _jerk,
                    BufferMode := _buffer_mode,
                    Options := _option
                );
                _exec_state := 1;
            1:
                IF _fb_move_abs.Busy THEN
                    _fb_move_abs(Axis := _axis,Execute := FALSE);
                END_IF
        END_CASE

        _error_id := _fb_move_abs.ErrorID;
        execute := _fb_move_abs.Done OR _fb_move_abs.Error;
        ```

    * `error_reset()` のオーバライド

        軸エラーは `_error_id` にセットされているので、何等かのエラー発生時にはfutureのエラーリセットによって `MC_Reset` を実行するように定義。

        ``` pascal
        METHOD error_reset : BOOL // Must be overrided if future has any errors.

        IF _error_id <> 0 THEN
            CASE _exec_state OF
                0:
                    _mc_reset(Axis := _axis, Execute := FALSE);
                    _mc_reset(Axis:= _axis, Execute := TRUE );
                    _exec_state := 1;
                1:
                    IF _mc_reset.Busy THEN
                        _mc_reset(Axis:= _axis,Execute := FALSE);
                    END_IF
            END_CASE
            
            _error_id := _mc_reset.ErrorID;
            error_reset := _mc_reset.Done;
        ELSE
            error_reset := TRUE;
        END_IF
        ```


他にも必要に応じて [API](https://beckhoff-jp.github.io/TwinCATHowTo/job_management/library_document.html#interfacefuture) に示された他ののメソッドを定義してください。上記では abort() の定義例が示されていませんが、`FB_Executor`は、`_error_id` が0以外の値で`init()`, `execute()`, `quit()`が終了すると、自動的に `abort()` を実行しますので、_error_id の値に応じたエラー発生時の処理を定義してください。

## ジョブの生成

ジョブ生成はどのタイミングで行っても良いのですが、 `init` 変数を使ってPLCのRUN後1サイクルだけ実行するブロックを作成し、その中でジョブの生成を行う例について示します。

次のとおり、ジョブツリーの最上位には、何等かの `FB_Executor` のインスタンスが必要です。下記のとおり `demo_job` 変数を宣言し、 init ブロックにてここにジョブを作成します。

``` pascal
VAR
    demo_job : FB_Executor;
    init : BOOL;
END_VAR

IF NOT init THEN
    <demo_job にジョブを構築していく>
    init := TRUE;
END_IF

```

### コンテナジョブの作成方法

単一のジョブだけでなく、並列実行や逐次実行のジョブを行うには、まずジョブコンテナを生成する必要があります。ジョブオブジェクト `FB_Executor` にはコンテナジョブを生成するメソッド `create_container` が用意されていますので、これを使ってコンテナを生成します。

メソッド呼び出し時には次の引数を指定します。

|引数|型|説明|
|---|---|---|
|container_type|ContainerType|生成するコンテナ型に対応したEnumの列挙子を指定します。|
|name|STRING|コンテナのfuture名を設定します。|

`container_type` に設定可能な値は次の通りです。

|ContainerType|生成されるコンテナ型|
|---|---|
|BATCH|FB_BatchJobContainer|
|PARALLEL|FB_ParallelJobContainer|
|QUEUE|FB_QueueJobContainer|
|PARALLEL_QUEUE|FB_ParallelQueueJobContainer|

たとえば、 `demo_job` に`DEMO_JOB`という名前のBATCHジョブコンテナを生成するには次の通り実行します。

``` pascal
demo_job.create_container(ContainerType.BATCH, 'DEMO_JOB');
```

生成したコンテナオブジェクトへは、`children` プロパティを通じてアクセスできます。たとえばコンテナオブジェクトに任意のジョブを追加するには次のとおり記述します。

``` pascal
demo_job.children.append_job(<ANY_JOB>);
```

コンテナオブジェクトが持つジョブ登録方法は次のパターンがあります。

|メソッド|引数の型|説明|
|---|---|---|
|append_job|FB_Executor|指定したジョブオブジェクトをコンテナへ追加する|
|spawn_job|InterfaceFuture|コンテナにジョブを生成した上で指定したFutureオブジェクト登録する。|

### ジョブ生成の実装例

前節で作成した `FutureMCMoveAbsolute` のFutureオブジェクトを、 `BATCH` ジョブコンテナ上に登録して絶対位置250 degと、-250 degの動作を10回繰り返す処理を定義します。

``` pascal
PROGRAM demo1
VAR
    init        : BOOL;
    axis1       : AXIS_REF;
    demo_job    : FB_Executor;
    recipro_moving  : ARRAY [1..10] OF FutureMCMoveAbsolute;
    i : UDINT;
END_VAR


// RUN 後 1 サイクルだけ実行
IF NOT init THEN
    demo_job.create_container(ContainerType.BATCH, 'DEMO_JOB');
    FOR i := 1 TO 10 DO
        IF i mod 2 > 0 THEN
            recipro_moving[i].set_parameters(axis1,250,200,0,0,0,0);
            recipro_moving[i].future_name := 'Move +250 deg, 200 deg/s';
        ELSE
            recipro_moving[i].set_parameters(axis1,-250,200,0,0,0,0);
            recipro_moving[i].future_name := 'Move -250 deg, 200 deg/s';
        END_IF
        demo_job.children.spawn_job(recipro_moving[i]);        
    END_FOR
    init := TRUE;
END_IF
```

## Task runnerを使って実行する

ジョブの実行エージェントとして`FB_TaskRunner`を用意しています。この使い方は基本的に次の通り実装します。

``` pascal
VAR
    runner : FB_TaskRunner;
END_VAR

// 作成済みのFuture形式のオブジェクトを追加。追加後は即座に実行。
runner.append_future(future : InterfaceFuture);

// 作成済みのジョブ型式のオブジェクトを追加。追加後は即座に実行。
runner.append_job(job : FB_Executor);

// 常時実行
runner();
```

このファンクションブロックを使い、次の通り`MC_Power`を使ってサーボONしたあと作成したジョブを実行するプログラムを実装します。

``` pascal
PROGRAM demo1
VAR
    :
    state : UDINT;
    fb_power : MC_Power;
    runner : FB_TaskRunner;
END_VAR

    :
    :

CASE state OF
    0: // サーボON
    s    fb_power(
            Axis:= axis1, 
            Enable:= TRUE, 
            Enable_Positive:= TRUE, 
            Enable_Negative:= TRUE, 
            Override:= 100 
        );
        IF fb_power.Status = TRUE THEN
            state := 1;
        END_IF
    1: // ジョブの実行登録
        runner.append_job(demo_job);
        state := 2;
    2: // ジョブの実行完了待ち
        IF demo_job.done THEN
            state := 3;
        END_IF
    3: // サーボOFF
        fb_power(
            Axis:= axis1, 
            Enable:= FALSE, 
            Enable_Positive:= FALSE, 
            Enable_Negative:= FALSE, 
            Override:= 100 
        );
END_CASE

runner();
```

## 入れ子のジョブにする

前回作成したジョブは単純な位置決めを繰り返すだけでしたが、次図のとおりブレンディングモードで動作させる場合についてご紹介します。

この動作を実現するには、2つの `MC_MoveAbsolute` のインスタンスを同一サイクル上で同時に実行します。これらのファンクションブロックを `Execute := TRUE` にする順番のうち、最初のファンクションブロックのBufferModeを `MC_BufferMode.MC_Aborting` に設定し、後半を次図に示す連続稼働条件のいずれかのブレンディングモードを設定します。これによって最初のファンクションブロックの終了した後、次のファンクションブロックを逐次連続的に実行するように、コマンドをバッファリンクできます。

たとえば2段目を `MC_BufferMode.MC_BlendingNext` と設定すると、初段の位置決めの目標位置に到達した時点で、次段の設定速度に到達なるような速度制御を行います。

![](https://infosys.beckhoff.com/content/1033/tcplclib_tc2_mc2/Images/png/18014399559118475__en-US__Web.png)

これをジョブフレームワークで実装するために、`PARALLEL` コンテナを構築し、その中に`MC_Aborting`, `MC_BlendingNext` のジョブをセットします。さらにこれを順次実行する `BATCH` コンテナを親ジョブとしてセットします。

``` pascal
PROGRAM demo2
VAR
    init        : BOOL;
    axis1       : AXIS_REF;
    demo_job    : FB_Executor;
    recipro_moving  : ARRAY [1..2, 1..10] OF FutureMCMoveAbsolute;
    _job_temp   : REFERENCE TO FB_Executor;     // Parallel jobコンテナのジョブへの参照変数
    i : UDINT;

    state : UDINT;
    fb_power : MC_Power;
    runner : FB_TaskRunner;
    option : ST_MoveOptions;
END_VAR


IF NOT init THEN
    demo_job.create_container(ContainerType.BATCH, 'DEMO_JOB');
    
    FOR i := 1 TO 10 DO
        _job_temp REF= demo_job.children.create_container(ContainerType.PARALLEL,'Blending move container');
        IF i mod 2 > 0 THEN
            recipro_moving[1, i].set_parameters(axis1,500,600,0,0,0,MC_BufferMode.MC_Aborting);
            recipro_moving[1, i].future_name := 'Move +500 deg, 600 deg/s';
            recipro_moving[2, i].set_parameters(axis1,1000,50,0,0,0,MC_BufferMode.MC_BlendingNext);
            recipro_moving[2, i].future_name := 'Move +1000 deg, 50 deg/s';
        ELSE
            recipro_moving[1, i].set_parameters(axis1,500,600,0,0,0,MC_BufferMode.MC_Aborting);
            recipro_moving[1, i].future_name := 'Move +500 deg, 600 deg/s';
            recipro_moving[2, i].set_parameters(axis1,0,50,0,0,0,MC_BufferMode.MC_BlendingNext);
            recipro_moving[2, i].future_name := 'Move +0 deg, 50 deg/s';
        END_IF
        _job_temp.children.spawn_job(recipro_moving[1, i]);        
        _job_temp.children.spawn_job(recipro_moving[2, i]);        
        demo_job.children.append_job(_job_temp);
    END_FOR
    init := TRUE;
END_IF


CASE state OF
    0:
        fb_power(
            Axis:= axis1, 
            Enable:= TRUE, 
            Enable_Positive:= TRUE, 
            Enable_Negative:= TRUE, 
            Override:= 100 
        );
        IF fb_power.Status = TRUE THEN
            state := 1;
        END_IF
    1:
        runner.append_job(demo_job);
        state := 2;
    2:
        IF demo_job.done THEN
            state := 3;
        END_IF
    3:
        fb_power(
            Axis:= axis1, 
            Enable:= FALSE, 
            Enable_Positive:= FALSE, 
            Enable_Negative:= FALSE, 
            Override:= 100 
        );
END_CASE

runner();

```

# より進んだ使い方

## ファクトリメソッド定義

これまでの例では、ジョブ、および、コンテナに登録可能なジョブに登録するFutureオブジェクトを別途、変数定義する必要がありました。これらは競合して実行されないように配列などを使って個別のインスタンス化を行っていました。

コンテナを用いて複雑なジョブツリーに合わせて、Futureインスタンスを別途用意するのはプログラム上、管理が煩雑になってしまいます。

これを避けるため動的にFutureを生成する **ファクトリパターン** を使ってFutureインスタンスを動的に生成する `InterfaceTaskCreator` を用意しています。

本フレームワークを利用する開発者は、独自の `Future` を実装すると同時に、 `InterfaceTaskCreator` を実装するFutureオブジェクト生成用ファンクションブロックを作成します。命名規則としては、実装した `Future` ファンクションブロックの名前に `Creator` を追加した名前にすると良いでしょう。

``` pascal
FUNCTION_BLOCK FutureMCMoveAbsoluteCreator IMPLEMENTS InterfaceTaskCreator
:
```

`create_future` メソッドを定義します。コメントで記載した部分は、定義したFutureオブジェクト毎に特殊な定義が必要な個所を示します。

それ以外は基本的にこの型に従ってください。

``` pascal
METHOD create_future : InterfaceFuture
VAR_INPUT
    future_name :STRING;
END_VAR
VAR
    _p_future : POINTER TO FutureMCMoveAbsolute; // Futureファンクションブロック名を指定
END_VAR

_p_future := __NEW(FutureMCMoveAbsolute);  // Futureファンクションブロック名を指定
_p_future^.future_name := future_name;

// ここから インスタンス独自の設定（パラメータ設定等）を定義
// Futureと同様のパラメータのセッタを別途設けて、その値をセットしてください。
_p_future^.set_parameter(
    axis      := _axis;
    target_position := _target_position;
    velocity        := _velocity;
    acceralation    := _acceralation;
    deceralation    := _deceralation;
    jark            := _jark;
    buffer_mode     := _buffer_mode;
);
// ここまで

create_future := _p_future^;
```

次に `destroy_future` メソッドを定義します。`_p_future` のポインタ型をFutureファクションブロック型に変更するだけで、その他はそのままこのまま流用してください。

``` pascal
METHOD destroy_future : InterfaceFuture
VAR_INPUT
    future : InterfaceFuture;
END_VAR
VAR
    _p_future : POINTER TO FutureMCMoveAbsolute; // ここだけFutureファンクションブロック名に変える。
END_VAR

IF __QUERYPOINTER(future, _p_future) THEN
    __DELETE(_p_future);
    IF _p_future = 0  THEN
        destroy_future := 0;
    END_IF
END_IF

```

### ジョブの作成

作成した `InterfaceTaskCreator` の実装ファンクションブロックを使ったインスタンスは、動的にジョブを生成するファンクションブロックですので同じFutureを生成するのであればインスタンスは一つだけ定義すれば良いです。

これを使ってジョブを生成するには、 `FB_Executor` および、 ジョブコンテナに用意された `create_job()`メソッドを利用します。引数に、 `InterfaceTaskCreator` オブジェクト、および、生成するFuture nameを指定します。

例としてdemo3プログラムの `JobCreateDemo` メソッドの内容を少しシンプルにしたものを次に示します。いままでのサンプルコードに従い、　BATCHジョブコンテナを生成しておいた　`demo_job` の `children` 以下に動的にジョブを生成するサンプルです。

ファクトリメソッド `FutureMCMoveAbsoluteCreator` 型のインスタンス一つに対して、 `create_job` メソッドを用いて都度異なるパラメータを設定し、動的に個別のジョブインスタンスを生成する実装例です。

``` pascal
PROGRAM demo3
VAR
    :
    mc_move_abs_future_creator : FutureMCMoveAbsoluteCreator;   // ファクトリメソッドを持つオブジェクト
    :
END_VAR

    :
// 生成されたJobを操作したい場合は、 create_jobの 直後で _job_temp 変数を通じて行う。
mc_move_abs_future_creator.set_parameters(stAxis1,1000,1000,0,0,0,0);
_job_temp REF= demo_job.children.create_job(mc_move_abs_future_creator,'go to position 1000');

mc_move_abs_future_creator.set_parameters(stAxis1,500,1000,0,0,0,0);
_job_temp REF= demo_job.children.create_job(mc_move_abs_future_creator,'go to position 500');

mc_move_abs_future_creator.set_parameters(stAxis1,0,1000,0,0,0,0);
_job_temp REF= demo_job.children.create_job(mc_move_abs_future_creator,'go to position 0');
    :

```

## 詳細

その他の本フレームワークについての詳細記事がありますのでご参照ください。

[https://beckhoff-jp.github.io/TwinCATHowTo/job_management/index.html](https://beckhoff-jp.github.io/TwinCATHowTo/job_management/index.html)
