# この文書について

この文書は [lwn.net](https://lwn.net/) において 2013 年に公開された Namespaces in Operation シリーズのパート 3  ([http://lwn.net/Articles/531419/](http://lwn.net/Articles/531419/))を翻訳したものです。

この文書のライセンスは原文と同じく、[Creative Commons CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/) のもとに提供されています。

# Namespaces in operation, part 3: PID namespaces

先の 2 つの記事 ([Part 1: namespaces overview](http://lwn.net/Articles/531114/) と [Part 2: the namespaces API](http://lwn.net/Articles/531381/)) に続いて，これからは PID 名前空間を見ていく．PID 名前空間によって隔離されるグローバルリソースはプロセス ID 番号の空間である．これは，異なる PID 名前空間に属するプロセスは同じプロセス ID を持つことが可能になるという事である．PID 名前空間は，コンテナ内のプロセスは，同じプロセス ID を持ったまま，ホスト間を移動可能なコンテナの実装に使われる．

伝統的な Linux (や UNIX) システム上のプロセスと同様に，PID 名前空間内ではプロセス ID はユニークであり，PID 1 から始まり順に割り当てられる．さらに，伝統的な Linux システムと同じく，PID 1 (init プロセス) は特別である．これは名前空間内で作られる最初のプロセスであり，名前空間内での特定の管理タスクを実行する．

## 最初の調査 

新しい PID 名前空間は `CLONE_NEWPID` フラグを指定して [`clone()`](http://man7.org/linux/man-pages/man2/clone.2.html) を呼ぶことで作られる．ここで `clone()` を使って新しい PID 名前空間を作る簡単なサンプルプログラムを示そう．このプログラムを使って，PID 名前空間の基本コンセプトのいくつかをまとめよう．プログラム (`pidns_init_sleep.c`) のソース全体は [ここ](http://lwn.net/Articles/532741/)にある．このシリーズの前の記事と同様に，簡潔化のために，記事中では完全なソースでは入っているエラーチェックのコードを省いている．

main プログラムは `clone()` を使った新しい PID 名前空間を作成し，その結果生成された子供の PID を表示する．

```
child_pid = clone(childFunc,
                child_stack + STACK_SIZE,   /* Points to start of
                                               downwardly growing stack */
                CLONE_NEWPID | SIGCHLD, argv[1]);

printf("PID returned by clone(): %ld\n", (long) child_pid);
```

新しい子プロセスは `childFunc()` 内で実行を開始する．これは `clone()` の最後の引数 (`argv[1]`) を自身の引数として受け取る．この引数の目的は後で明らかにする．

`chileFunc()` 関数はプロセス ID と `clone()` が作成した子プロセスの親プロセスの ID を表示する．そして，標準の `sleep` プログラムを実行して終了する．

```
printf("childFunc(): PID = %ld\n", (long) getpid());
printf("ChildFunc(): PPID = %ld\n", (long) getppid()); 
...
execlp("sleep", "sleep", "1000", (char *) NULL); 
```

`sleep` プログラムを実行する目的は，プロセスのリストで親から子プロセスを区別する事を簡単にするためである．

このプログラムを実行すると，出力の最初の行は以下のようになる．

```
$ su         # Need privilege to create a PID namespace
Password: 
# ./pidns_init_sleep /proc2
PID returned by clone(): 27656
childFunc(): PID  = 1
childFunc(): PPID = 0
Mounting procfs at /proc2
```

`pidns_init_sleep` の出力の最初の 2 行は，2 つの異なった PID 名前空間の見方から子プロセスの PID を示している．`clone()` を呼び出した側の名前空間と名前空間内の子プロセス側からである．言い換えると，子プロセスは 2 つの PID を持つということである．一つは親の名前空間の 27656，もう 1 つは `clone()` によって作られた PID 名前空間内の 1 である．

出力の次の行は，PID 名前空間内の子供側からのコンテキスト内での，子供の親プロセスの ID を示している (`getppid()` の返り値)．親の PID は 0 である．PID 名前空間の少し変わった所を示している．あとで述べるように，PID 名前空間は階層構造を形成している．プロセスは自身の PID 名前空間内と，その PID 名前空間以下にネストした子供の名前空間内に含まれるプロセスのみを見ることができる．`clone()` が作成した子供の親は異なる名前空間にいるので，子供は親を見ることができない．このため，`getppid()` はゼロとして親の PID を返すのである．

`pidns_init_sleep` の出力の最後の行の説明のため，`childFunc()` 関数の実装の議論の時にスキップした部分のコードに戻る必要がある．

## /proc/PID と PID 名前空間 

Linux システムのプロセスはそれぞれ ``/proc/PID`` ディレクトリを持っている．ここにはプロセスに関係する様々な情報を含む擬似ファイルを含む．この配置は直接 PID 名前空間のモデルに置き換えることができる．PID 名前空間の中では，`/proc/PID` ディレクトリは PID 名前空間内と子孫に含まれるプロセスの情報のみを表示する．

しかし，目に見える PID 名前空間に一致する `/proc/PID` ディレクトリを作成するために，proc ファイルシステム (短縮して "procfs") を PID 名前空間内からマウントする必要がある．PID 名前空間内で実行中のシェルから (おそらく `system()` ライブラリ関数で起動した)，以下のような形で mount コマンドを使用してこれを実行する事が可能である．

```
# mount -t proc proc /mount_point
```

もうひとつの方法として，procfs は `mount()` システムコールを使ってマウントすることも可能である．プログラムの `childFunc()` 関数内から以下のようにして．

```
mkdir(mount_point, 0555);       /* Create directory for mount point */
mount("proc", mount_point, "proc", 0, NULL);
printf("Mounting procfs at %s\n", mount_point);
```

`mount_point` 変数は `pidns_init_sleep` 起動時のコマンドライン引数で与えられる文字列で初期化される．

前の `pidns_init_sleep` を実行したサンプルのシェルセッションでは，新しい procfs として `/proc2` をマウントした．現実の利用では，説明したどちらかのテクニックを使って，procfs は (必要であれば) 通常は普通の場所にマウントされるだろう．しかし，我々のデモの間 `/proc2` に procfs をマウントすることは，システムの他のプロセスに問題が生じるのを防ぐ簡単な方法を提供してくれる．つまり，これらのプログラムは我々のテストプログラムと同じ名前空間にあるので，`/proc` にマウントされるファイルシステムを変更することは，目に見えない root PID 名前空間に `/proc/PID` ディレクトリを作成するシステムの他のプログラムに混乱を与えるだろう．

それゆえ，我々のシェルセッション内で `/proc` にマウントされた procfs は，親の PID 名前空間から見えるプロセスに対する PID サブディレクトリを示し，一方で `/proc2` にマウントされた procfs は，子供の PID 名前空間に存在するプロセスに対する PID サブディレクトリを示す．ちなみに，子供の PID 名前空間内のプロセスが `/proc` マウントポイントで見える PID ディレクトリで見えるにも関わらず，これらの PID は子供の PID 名前空間内のプロセスに対しては意味がないという事は重要な事である．これらの子プロセスから呼ばれたシステムコールは，それらが属する PID 名前空間のコンテキスト内の PID として解釈されるからである．

もし ps のような様々なツールが，子供の PID 名前空間内でも正しく動くことを期待するのであれば，伝統的な `/proc` マウントポイントにマウントされた procfs を持つことは必要なことである．なぜならこれらのツールは `/proc` で見つかる情報に依存しているからである．親の PID 名前空間が使う `/proc` マウントポイントに影響を与えずにこれを達成する方法は 2 つある．1 つ目は，もし子プロセスが `CLONE_NEWNS` フラグを使って作られているのであれば，子供はシステムの他とは異なるマウント名前空間にいる事になる．この場合，新しい procfs を `/proc` にマウントしても，問題を引き起こすことはない．他の方法として，`CLONE_NEWNS` フラグを使う代わりに，子供は `chroot()` を使って root ディレクトリを変更し，`/proc` に procfs をマウントするのである．

`pidns_init_sleep` を実行しているシェルセッションに戻ろう．プログラムを停止させ，親の名前空間のコンテキスト内で親子のプロセスの詳細をいくつか観察するために `ps` コマンドを使おう．

```
^Z                          Stop the program, placing in background
[1]+  Stopped                 ./pidns_init_sleep /proc2
# ps -C sleep -C pidns_init_sleep -o "pid ppid stat cmd"
  PID  PPID STAT CMD
27655 27090 T    ./pidns_init_sleep /proc2
27656 27655 S    sleep 600
```

最後の行の "PPID" の値 (27655) は，`sleep` を実行しているプロセスの親が `pidns_init_sleep` を実行しているプロセスであることを示している．

`readlink` コマンドを使って，`/proc/PID/ns/pid` シンボリックリンク ([先の記事](http://lwn.net/Articles/531381/#proc_pid_ns)で説明済み) の (異なる) 表示を表示させ，2 つのプロセスが異なる PID 名前空間にいることを見ることができる．

```
# readlink /proc/27655/ns/pid
pid:[4026531836]
# readlink /proc/27656/ns/pid
pid:[4026532412]
```

この時点で，新しい PID 名前空間内のプロセスに関する情報を得るのに，新しくマウントした procfs を使うこともできる．まず，以下のようなコマンドで名前空間内の PID のリストを取得できる．

```
# ls -d /proc2/[1-9]*
/proc2/1
```

以上のように，PID 名前空間はひとつのプロセスだけを含んでおり，その名前空間での PID は 1 である．また，`/proc/PID/status` ファイルを使った別の方法で，前のシェルセッションで見たプロセスに関する情報と同じものを取得可能である．

```
# cat /proc2/1/status | egrep '^(Name|PP*id)'
Name:   sleep
Pid:    1
PPid:   0
```

ファイルの PPid フィールドは 0 であり，子供に対する親プロセスの ID が 0 であると報告した `getppid()` の結果と一致する．

## ネストした PID 名前空間 

先に述べたように，PID 名前空間は親子関係で階層構造的にネストする．PID 名前空間内では，同じ名前空間内の他の全てのプロセスを見ることができるだけでなく，子孫の名前空間のメンバーの全てのプロセスも見る ("see") ことができる．ここで「見る」("see") とは PID を指定して操作するシステムコールを利用できることを意味する(例えばプロセスにシグナルを送る `kill()` の利用など)．子供の PID 名前空間内のプロセスは親の PID 名前空間内 (や消去された祖先の名前空間) に存在するプロセスを見ることはできない．

プロセスは，属している PID 名前空間から始まり，root PID 名前空間までに属する PID 名前空間の階層構造のぞれぞれのレイヤーごとに 1 つ PID を持つだろう．`getpid()` を呼ぶと，プロセスが属する名前空間に紐付いた PID が常に返ってくる．

プロセスが，そのプロセスが見える名前空間内のそれぞれで異なる PID を持っていることを示すために [ここ](http://lwn.net/Articles/532745/)のプログラムを見てみよう．簡潔な記述のために，ここではコードを通して見るよりは，プログラムがやっていることを簡単に説明しよう．

プログラムはネストされた PID 名前空間内で一連の子プロセスを再帰的に作成する．プログラムを起動する際に，何個の子供と PID 名前空間を作るのかをコマンドライン引数で指定する．

```
# ./multi_pidns 5
```

新しい子プロセスを作るのに加えて，それぞれの再帰ステップはユニークに名前が付けられたマウントポイントに procfs をマウントする．再帰の最後に，最後の子供が `sleep` を実行する．先のコマンドラインは以下のような出力が出る．

```
^Z                           Stop the program, placing in background
[1]+  Stopped            ./multi_pidns 5
# ls -d /proc4/[1-9]*        Topmost PID namespace created by program
/proc4/1  /proc4/2  /proc4/3  /proc4/4  /proc4/5
# ls -d /proc3/[1-9]*
/proc3/1  /proc3/2  /proc3/3  /proc3/4
# ls -d /proc2/[1-9]*
/proc2/1  /proc2/2  /proc2/3
# ls -d /proc1/[1-9]*
/proc1/1  /proc1/2
# ls -d /proc0/[1-9]*        Bottommost PID namespace
/proc0/1
```

全てを見ることができる名前空間内で，再帰の末尾 (例えば最も深くネストした名前空間内で `sleep` を実行しているプロセス) でプロセスの PID を見るには `grep` コマンドが適切だろう．

```
# grep -H 'Name:.*sleep' /proc?/[1-9]*/status
/proc0/1/status:Name:       sleep
/proc1/2/status:Name:       sleep
/proc2/3/status:Name:       sleep
/proc3/4/status:Name:       sleep
/proc4/5/status:Name:       sleep
```

言い換えると，最も深くネストした PID 名前空間 (`/proc0`) 内では，sleep を実行しているプロセスは PID 1 を持つ．そして，最も浅い PID 名前空間で作られた名前空間 (`/proc4`) では，プロセスは PID 5 を持つ．

もしこの記事で紹介したプログラムを実行した場合，プログラムはマウントポイントとマウントディレクトリをそのままにしていることに注意すること．プログラムが終了した後，以下のようなシェルコマンドが片付けをするのに十分である．

```
# umount /proc?
# rmdir /proc?
```

## 最後に 

この記事では，PID 名前空間についてかなり詳しく見た．次の記事では，PID 名前空間での `init` プロセスについての議論と説明を記載すると共に，PID 名前空間 API の他の詳細についても少し触れる．
