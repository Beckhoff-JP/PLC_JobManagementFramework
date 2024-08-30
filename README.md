# TwinCAT PLC ジョブフレームワーク

PLC上で様々な制御上のアクティビティを定義するフレームワークです。`FB_Executor`と呼ばれる実行エンジンによって`InterfaceFuture`インターフェースを実装した`Future`という単位のオブジェクトを実行します。

![](https://beckhoff-jp.github.io/TwinCATHowTo/_images/task_time_chart.png)

|メソッド|説明|終了条件|
|-|-|-|
|init()|オブジェクト内部の変数を初期化します。|init()の戻り値がTRUEとなる|
|execute()|実際に処理を定義します。|execute()の戻り値がTRUEとなる|
|quit()|処理後の処理を定義します。|quit()の戻り値がTRUEとなる|

`FB_Executor`と`Future`オブジェクトが組み合わせたものを「ジョブ」と呼びます。ジョブそのものは特定のアクチュエータに対する特定の動作モード毎に定義しますが、これら複数のジョブを組み合わせて動作させるための「コンテナ」という形式のFutureオブジェクトもあります。

コンテナには、組み合わせたジョブを実行させる方法の違いによる4種類が用意されています。

|コンテナ名|説明|
|---|---|
|FB_BatchJobContainer|あらかじめ登録したジョブを順次実行するコンテナです。|
|FB_ParallelJobContainer|あらかじめ登録したジョブを同時並列で実行するコンテナです。|
|FB_QueueJobContainer|順次実行するコンテナですが、実行中も随時ジョブを登録することができます。|
|FB_ParallelQueueJobContainer|同時並列で実行するコンテナですが、実行中も随時ジョブを登録することができます。|

このコンテナを用いることで、複数のジョブを順次実行、または、並列実行させることができます。また、コンテナには、子コンテナを含む事もでき、階層状の複雑なジョブを構成することができます。

![](https://beckhoff-jp.github.io/TwinCATHowTo/_images/sample_code_job_structure.png)

## インストール

1. 以下のTwinCATプロジェクトをGithubリポジトリからクローンしてください。

    [https://github.com/Beckhoff-JP/PLC_JobManagementFramework](https://github.com/Beckhoff-JP/PLC_JobManagementFramework)

2. TwinCAT XAEでプロジェクトを開き、[ライブラリ作成手順](https://beckhoff-jp.github.io/TwinCATHowTo/library/make_library.html#twincat)に従ってライブラリファイル`JobManagement.library`を保存してください。

3. [こちらの手順に従い](https://beckhoff-jp.github.io/TwinCATHowTo/library/use_library.html#section-use-library)目的のTwinCATのPLCプロジェクトに保存したライブラリをインストール・追加します。

## フレームワークの使い方

### Futureファンクションブロックの作成

1. まず、Futureファンクションブロックを定義します。`InterfaceFuture`を実装すると全てのメソッドを定義する必要がありますが、あらかじめ最低限のロジックが実装された`FB_AbstructFuture` 抽象クラスを継承して定義する方が楽です。

    ![](https://beckhoff-jp.github.io/TwinCATHowTo/_images/2024-07-06-18-25-53.png)


3. 作成したファンクションブロックに``{attribute `enable_dynamic_creation`}``を追加します。
    
    ``` pascal
    {attribute 'enable_dynamic_creation'}
    FUNCTION_BLOCK FB_TaskMoveToPosition EXTENDS FB_AbstructFuture
    VAR
    :
    ```

4. ファンクションブロックのメンバ変数を定義します。

    ``` pascal
    {attribute 'enable_dynamic_creation'}
    FUNCTION_BLOCK FB_TaskMoveToPosition EXTENDS FB_AbstructFuture
    VAR
        _axis           : AXIS_REF;
        _target_position : MC_REAL;
        _velocity       : MC_REAL;
        _acceralation   : MC_REAL;
        _deceralation   : MC_REAL;
        _jark           : MC_REAL;
        _buffer_mode    : MC_BufferMode;
        _options        : ST_MoveOptions;
        _mc_move_abs    : MC_MoveAbsolute;
        _mc_reset       : MC_Reset;
    END_VAR
    ```

5. 入出力変数や独自パラメータを受け渡すメソッドやプロパティを追加します。入出力変数は、REFERENCEにより参照渡しする必要があります。

    ``` pascal
    METHOD set_parameter : BOOL
    VAR_INPUT
        axis           : REFERENCE TO AXIS_REF;
        target_position : MC_REAL;
        velocity       : MC_REAL;
        acceralation   : MC_REAL;
        deceralation   : MC_REAL;
        jark           : MC_REAL;
    END_VAR

    _axis REF= axis;
    _target_position := target_position;
    _velocity := velocity;
    _acceralation := acceralation;
    _deceralation := deceralation;
    _jark := jark;
    ```

6. `init()`と`execute()`メソッドを定義します。MethodをAddし、Name欄の選択肢にはオーバライド可能なメソッドが一覧されますので、実装したい任意のメソッドを選びます。

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

    例） init()　使用する変数の初期化等を行う
    ``` pascal
    METHOD init : BOOL
    VAR
    END_VAR

    _mc_move_abs.Execute := FALSE;
    _mc_reset.Execute := FALSE;
    init := TRUE;
    ```

    例） execute() 実際の制御を定義
    ``` pascal
    METHOD execute : BOOL
    VAR_INST // メソッド内だけで使用できる
        state : UDINT
    END_VAR


    CASE _state OF
    0:
        _mc_move_abs.Position := _target_position_;
        _mc_move_abs.Velocity := _velocity_;
        _mc_move_abs.BufferMode := _buffer_mode;
        _mc_move_abs.Execute := TRUE;
        IF _mc_move_abs.Active  THEN
            _mc_move_abs.Execute := FALSE;
            state := 1;
        ELSIF _mc_move_abs.CommandAborted OR _mc_move_abs.Error THEN
            _error_id := _mc_move_abs.ErrorID;  // _error_id はFB_AbstructFuture で定義されている。
            execute := TRUE;                    // エラー中断時は_error_idを0以外の値にしてexecute = TRUEを返す。
        END_IF
    1:
        IF _mc_move_abs.Done THEN
            _mc_move_abs.Execute := FALSE;
            _error_id := 0;
            execute := TRUE;  // 正常終了時は_error_idを0にしてexecute = TRUEを返す。
        ELSIF _mc_move_abs.CommandAborted OR _mc_move_abs.Error THEN
            _error_id := _mc_move_abs.ErrorID;
            execute := TRUE; // エラー中断時は_error_idを0以外の値にしてexecute = TRUEを返す。
        END_IF
    END_CASE

    _mc_move_abs(); // call MC_MoveAbsolute every cycle
    _mc_reset(); // call MC_Reset every cycle
    ```



他にも必要に応じて [API](https://beckhoff-jp.github.io/TwinCATHowTo/job_management/library_document.html#interfacefuture) に示された他ののメソッドを定義してください。上記では abort() の定義例が示されていませんが、`FB_Executor`は、`_error_id` が0以外の値で`init()`, `execute()`, `quit()`が終了すると、自動的に `abort()` を実行しますので、_error_id の値に応じたエラー発生時の処理を定義してください。

### ファクトリメソッド定義

コンテナオブジェクト、またはFB_executorへFutureを登録する方法は次の二つがあります。

* あらかじめFutureインスタンス変数を宣言して、`add_future`で追加する方法
* インスタンスを動的に生成してくれるファクトリファンクションブロックを使って、`create_future` を使った動的なインスタンス生成方法

ここでは後者のファクトリファンクションブロックを使ってインスタンス生成する場合に使うファクトリファンクションブロックの作り方を説明します。

まず、`InterfaceTaskCreator` を実装するファンクションブロックを作成します。生成したいインスタンスのファンクションブロック名にCreatorを付けた名前にすると良いでしょう。

このファンクションブロックにもメンバ変数、および、`set_parameter`のようなアクセッサを用意します。

``` pascal
FUNCTION_BLOCK FB_TaskMoveToPositionCreator IMPLEMENTS InterfaceTaskCreator
:
```

create_future メソッドを定義します。

``` pascal
METHOD create_future : InterfaceFuture
VAR_INPUT
    future_name :STRING;
END_VAR
VAR
    _p_future : POINTER TO FB_TaskMoveToPosition; // Futureファンクションブロック名を指定
END_VAR

_p_future := __NEW(FB_TaskMoveToPosition);  // Futureファンクションブロック名を指定
_p_future^.future_name := future_name;

// ここから インスタンス独自の設定（パラメータ設定等）を定義
_p_future^.set_parameter(
    axis      := _axis;
    target_position := _target_position;
    velocity       := _velocity;
    acceralation   := _acceralation;
    deceralation   := _deceralation;
    jark           := _jark;
);
// ここまで

create_future := _p_future^;
```

destroy_future メソッドを定義します。`_p_future` のポインタ型をFutureファクションブロック型に変更するだけで、その他はそのままコピーして使ってください。

``` pascal
METHOD destroy_future : InterfaceFuture
VAR_INPUT
    future : InterfaceFuture;
END_VAR
VAR
    _p_future : POINTER TO FutureCounterBlink; // ここだけFutureファンクションブロック名に変える。
END_VAR

IF __QUERYPOINTER(future, _p_future) THEN
	__DELETE(_p_future);
	IF _p_future = 0  THEN
		destroy_future := 0;
	END_IF
END_IF

```

### ジョブの作成

メインプログラムにおけるジョブの生成方法について説明します。まずメインジョブとなる`FB_executor`のインスタンスを作成します。その下に`FB_BatchJobContainer`を生成し、そこへ、これまで作成したFutureを使って複数の位置への位置決めを順次行うジョブを生成する例を示します。

``` pascal
PROGRAM MAIN
VAR
    main_job : FB_Executor;
    move_pos_creater : FB_TaskMoveToPositionCreator;
    Vel     : UDINT := 400;
    Acc     : UDINT := 4000;
    Dcc     : UDINT := 4000;
    Jerk    : UDINT := 20000;
    axis    : AXIS_REF;
    state   : UDINT;
    bStart  : BOOL; // シーケンススタート
    bStop   : BOOL; // 中断ボタン
    bReset  : BOOL; // ジョブのオールクリア
    bResume  : BOOL; // 再開ボタン
END_VAR

CASE state OF
    0:
        IF bStart THEN
            state := 1;
        END_IF

    1: // Job creating

        main_job.create_container(ContainerType.BATCH, 'MAIN JOB');

        // Move Postion1
        move_pos_creater.set_parameter(
            axis      := axis;
            target_position := 320.14;
            velocity       := vel;
            acceralation   := Acc;
            deceralation   := Dcc;
            jark           := Jark;
        );

        main_job.children.create_future(move_pos_creater, 'MOVE TO POSITION 1');

        // Move Postion2
        move_pos_creater.set_parameter(
            axis      := axis;
            target_position := 450.00;
            velocity       := vel;
            acceralation   := Acc;
            deceralation   := Dcc;
            jark           := Jark;
        );

        main_job.children.create_future(move_pos_creater, 'MOVE TO POSITION 2');

        // Move to home
        move_pos_creater.set_parameter(
            axis      := axis;
            target_position := 0.00;
            velocity       := vel;
            acceralation   := Acc;
            deceralation   := Dcc;
            jark           := Jark;
        );

        main_job.children.create_future(move_pos_creater, 'MOVE TO HOME');

        state := 2;
    2: // initialization
        IF main_job.init() THEN
            state := 3;
        END_IF

    3: // executing

        // execute()で実行しつつ、戻り値TRUEを監視。
        IF main_job.execute() AND fbJob.nErrorID = 0 THEN
            state := 0;　// エラー無しにTRUEが戻ると終了。
        END_IF

        // 初期化が完了し、start()で実行
        IF main_job.current_state = jobmgmt.E_FutureExecutionState.wait_for_process AND fbJob.ready THEN
            main_job.start();
        END_IF

        // 一時停止する
        IF bStop THEN
            main_job.abort();
        END_IF

        // 中止、オールクリア
        IF bReset THEN
            main_job.reset();
            state := 0;　
        END_IF

        // 再開
        IF main_job.current_state = jobmgmt.E_FutureExecutionState.abort AND bResume THEN
            main_job.start();
        END_IF
```

#### ジョブコンテナ作成方法

FB_Executorインスタンスのメソッド `create_container`により次の引数を指定してコンテナを生成します。

|引数|型|説明|
|---|---|---|
|container_type|ContainerType|生成するコンテナ型に対応したEnumの列挙子を指定します。|
|name|STRING|コンテナのfuture名を設定します。|

列挙しに設定可能な値は次の通りです。

|ContainerType|生成されるコンテナ型|
|---|---|
|BATCH|FB_BatchJobContainer|
|PARALLEL|FB_ParallelJobContainer|
|QUEUE|FB_QueueJobContainer|
|PARALLEL_QUEUE|FB_ParallelQueueJobContainer|

生成したコンテナオブジェクトへは、`children` プロパティを通じてアクセスする事ができます。

``` pascal
main_job.create_container(ContainerType.BATCH, 'MAIN JOB');
main_job.children.create_future(...);
```

#### ジョブ生成方法

ジョブ生成方法は2とおりあります。まず、変数宣言によって明示的に作成したInterfaceFutureの実装オブジェクトを直接`future`プロパティにセットする方法です。動作パターン毎に個別のインスタンスを作成する必要があります。

``` pascal
VAR
    move_pos1 : FB_TaskMoveToPosition; // pos1へ移動するための位置決めインスタンス
    move_pos2 : FB_TaskMoveToPosition; // pos2へ移動するための位置決めインスタンス
    move_home : FB_TaskMoveToPosition; // homeへ移動するための位置決めインスタンス
    main_job : FB_Executor;
    parameter_set :BOOL;
END_VAR

move_pos1.set_parameter(
            axis      := axis;
            target_position := 320.14;
            velocity       := vel;
            acceralation   := Acc;
            deceralation   := Dcc;
            jark           := Jark;
        );
move_pos1.future_name := 'MOVE TO POSITION 1';

move_pos2.set_parameter(
            axis      := axis;
            target_position := 450.00;
            velocity       := vel;
            acceralation   := Acc;
            deceralation   := Dcc;
            jark           := Jark;
        );
move_pos2.future_name := 'MOVE TO POSITION 2';

move_home.set_parameter(
            axis      := axis;
            target_position := 0;
            velocity       := vel;
            acceralation   := Acc;
            deceralation   := Dcc;
            jark           := Jark;
        );
move_home.future_name := 'MOVE TO HOME';
main_job.create_container(ContainerType.BATCH, 'MAIN JOB');
main_job.children.add_future(move_pos1);
main_job.children.add_future(move_pos2);
main_job.children.add_future(move_home);
```

もう一つが、インスタンス変数を定義せず、暗黙的にジョブを生成する方法です。FB_Executorインスタンスのメソッド `create_future` にファクトリクラスを通じて動的に生成しますので、宣言する変数は一つだけで構いません。

ファクトリクラスを通じて`create_container`実行前に必要な動作パラメータを都度設定します。

``` pascal
VAR
    move_pos_creater : FB_TaskMoveToPositionCreator;
    main_job : FB_Executor;
END_VAR



main_job.create_container(ContainerType.BATCH, 'MAIN JOB');

move_pos_creater.set_parameter(
            axis      := axis;
            target_position := 0.00;
            velocity       := vel;
            acceralation   := Acc;
            deceralation   := Dcc;
            jark           := Jark;
        );
main_job.children.create_future(move_pos_creater, 'MOVE TO HOME');
```

なお、ContainerTypeが`BATCH`, `PARALLEL`の場合、いちど生成したジョブは残留し続けますので、`main_job`は何度でも繰り返し実行可能です。配下のジョブ全てを削除するには、`reset()`を実行します。

``` pascal

IF main_job.reset() THEN
    // reset()からTRUEの戻り値が有れば全ジョブ削除完了
END_IF
```

ContainerTypeが`QUEUE`や`PARALLEL_QUEUE`の場合は、各子JOBが完了すると自動的にジョブは消滅します。

## 詳細

その他の本フレームワークについての詳細記事がありますのでご参照ください。

[https://beckhoff-jp.github.io/TwinCATHowTo/job_management/index.html](https://beckhoff-jp.github.io/TwinCATHowTo/job_management/index.html)
