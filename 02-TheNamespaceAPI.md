# この文書について

この文書は [lwn.net](https://lwn.net/) において 2013 年に公開された Namespace in Operation シリーズのパート 1  ([http://lwn.net/Articles/531114/](http://lwn.net/Articles/531114/))を翻訳したものです。

この文書のライセンスは原文と同じく、[Creative Commons CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/) のもとに提供されています。

# Namespaces in operation, part 2: namespaces API 

名前空間はグローバルなシステムリソースを抽象的なものの中に包みこむ．隔離されたリソースのインスタンスを持つ名前空間内でプロセスを生成させるということである．名前空間は様々な目的に使われる．最も有名なのは軽量の仮想化テクニックであるコンテナの実装である．これは名前空間と名前空間 API の詳細を見ていく記事のシリーズの第 2 部である．この記事では名前空間 API を少し詳細に見ていく．そして多数のサンプルプログラムで実際の API を説明する．

名前空間の API は 3 つのシステムコール，clone(), unshare(), setns() と多数の /proc 以下のファイルから構成される．この記事では，この全てのシステムコールと /proc 以下のファイルをいくつか見ていく．操作するのがどのタイプの名前空間なのかを特定するために，3 つのシステムコールは CLONE_NEW* という定数を使用する．先の記事で挙られていた CLONE_NEWIPC, CLONE_NEWNS, CLONE_NEWNET, CLONE_NEWPID, CLONE_NEWUSER, CLONE_NEWUTS である．

## 新しい名前空間内で子プロセスを生成する: clone()

名前空間を作成する方法の 1 つが，新しいプロセスを生成するシステムコールである clone() を使うことによるものである．clone() は以下のようなプロトタイプを持つ．

```
int clone(int (*child_func)(void *), void *child_stack, int flags, void *arg);
```

基本的にclone() は，伝統的な UNIX の fork() システムコールのより一般的なバージョジョンである．その機能は flags により制御される．clone() の様々な操作をコントロールするための，全部で 20 以上の異なる CLONE_* フラグが存在する．そのフラグは仮想メモリやオープンするファイルディスクリプタやシグナル操作のようなリソースを親子で共有するかどうかのためのものを含む．呼び出しで CLONE_NEW* の一つが指定された場合，指定したタイプに一致する新しい名前空間が作成され，新しいプロセスがその名前空間のメンバーとして作成される．flags には複数の CLONE_NEW* ビットを指定することも可能である．

我々のサンプルプログラム (demo_uts_namespace.c) は UTS 名前空間を作るために CLONE_NEWUTS を指定して clone() を使っている．先週見たように，UTS 名前空間は 2 つのシステムの識別子であるホスト名と NIS ドメイン名を隔離する．これは sethostname() と setdomainname() を使って設定され，uname() システムコールを使って返される．全ソースコードは[ここ](http://lwn.net/Articles/531245/)にある．以下で，このプログラムのキーとなるいくつかの部分にフォーカスを当てる (簡潔な記述のために，サンプルの全コードには載っているエラーチェックを省いている)．

サンプルプログラムはコマンドラインオプションを 1 つ与える．実行すると，新しい UTS 名前空間で動く子プロセスを 1 つ作成する．名前空間内で，子プロセスはコマンドラインオプションで与えた文字列でホスト名を変更する．

メインプログラムの最初の重要な部分は子プロセスを作成する clone() コールである:

```
    child_pid = clone(childFunc, 
                      child_stack + STACK_SIZE,   /* Points to start of 
                                                     downwardly growing stack */ 
                      CLONE_NEWUTS | SIGCHLD, argv[1]);

    printf("PID of child created by clone() is %ld\n", (long) child_pid);
```

新しい子供はユーザ定義の関数 childFunc(); 内で実行される．この関数は引数として clone() の最後の引数 argv[1] を受け取る．flags 引数として CLONE_NEWUTS が指定されているので，子供は新たに UTS 名前空間を作成して実行される．

メインプログラムはそれから一瞬 sleep する．これは，子供に自身の UTS 名前空間でホスト名を変える時間を与える (雑な) 方法である．プログラムはそれから uname() を使って親の UTS 名前空間でホスト名を取得し，ホスト名を表示する

```
    sleep(1);           /* Give child time to change its hostname */

    uname(&uts);
    printf("uts.nodename in parent: %s\n", uts.nodename);
```

その間に childFunc() 関数は clone() によって作られた子供で実行され，最初に引数で与えられた値でホスト名を変更し，それを取得し，表示する．

```
    sethostname(arg, strlen(arg);
    
    uname(&uts);
    printf("uts.nodename in child:  %s\n", uts.nodename);
```

終了前に，子供はしばらくの間 sleep する．これは子供の UTS 名前空間が保持されたままにする効果がある．この間に，後で述べる実験をいくつか実行する．

プログラムを実行すると，親と子のプロセスが独立した UTS 名前空間を持つ事を実演する．

```
    $ su                   # Need privilege to create a UTS namespace
    Password: 
    # uname -n
    antero
    # ./demo_uts_namespaces bizarro
    PID of child created by clone() is 27514
    uts.nodename in child:  bizarro
    uts.nodename in parent: antero
```

他のほとんどの名前空間と同様に，UTS 名前空間の作成には特権が必要である (はっきり言うと CAP_SYS_ADMIN)．これは，set-user-ID アプリケーションが不適切なことを偽って行い，期待しないホスト名を持つようなシナリオを防ぐのに必要である．

他の可能性として，set-user-ID アプリケーションがロックファイルの一部としてホスト名を使うかもしれないことがある．もし特権を持たないユーザが UTS 名前空間で任意のホスト名を設定してアプリケーションを実行できたら，アプリケーションに色々な攻撃の可能性を与えることになるだろう．このことにより，非常に簡単にロックファイルの効果を消し，異なる UTS 名前空間で実行するアプリケーションのインスタンスの不正行為のトリガとなるだろう．加えて，悪意あるユーザが UTS 名前空間で set-user-ID アプリケーションをホスト名を設定して実行し，重要なファイルを上書きするロックファイルを生成するということも引き起こせる．(ホスト名はスラッシュを含む任意の文字を含むことができる)

## /proc/PID/ns ファイル

プロセスごとにそれぞれの名前空間に対して 1 つのファイルを含む /proc/PID/ns ディレクトリが存在する．Linux 3.8 以降，これらのファイルはぞれぞれ特殊なシンボリックリンクファイルになっている．これはプロセスに紐づく名前空間に対する操作を行うためのある種のハンドルとして用意されている．

```
    $ ls -l /proc/$$/ns         # $$ is replaced by shell's PID
    total 0
    lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 ipc -> ipc:[4026531839]
    lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 mnt -> mnt:[4026531840]
    lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 net -> net:[4026531956]
    lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 pid -> pid:[4026531836]
    lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 user -> user:[4026531837]
    lrwxrwxrwx. 1 mtk mtk 0 Jan  8 04:12 uts -> uts:[4026531838]
```

このシンボリックリンクの使い方の一つに，2 つのプロセスが同じ名前空間に属しているかどうかを判別する事がある．カーネルは，もし 2 つのプロセスが同じ名前空間に属している場合は，/proc/PID/ns 内の一致するシンボリックリンクに対して同じ inode 番号を返す事を保証する．inode 番号は stat() システムコール (内の st_ino フィールド) を使って取得できる．カーネルは /proc/PID/ns シンボリックリンクのそれぞれが，inode 番号が後に続く名前空間の名前を指すような文字列からなる名前を指し示すようにも構成します．ls -l コマのか readlink コマンドを使って名前を調べることができます．

さて，先の demo_uts_namespace プログラムを実行した前のシェルセッションに戻りましょう．親プロセスと子プロセスの /proc/PID/ns シンボリックリンクを見ることは，2 つのプロセスが同じ UTS 名前空間か違う UTS 名前空間かを調べる別の方法を提供します．

```
    ^Z                                # Stop parent and child
    [1]+  Stopped          ./demo_uts_namespaces bizarro
    # jobs -l                         # Show PID of parent process
    [1]+ 27513 Stopped         ./demo_uts_namespaces bizarro
    # readlink /proc/27513/ns/uts     # Show parent UTS namespace
    uts:[4026531838]
    # readlink /proc/27514/ns/uts     # Show child UTS namespace
    uts:[4026532338]
```

上記のように，/proc/PID/ns/uts の指す先は異なるので，2 つの UTS 名前空間は異なる事が示されています．

/proc/PID/ns シンボリックリンクは別の目的にも使われます．もしこれらのファイルの一つを open した場合，名前空間内のプロセスが全て終了しても，名前空間はファイルディスクリプタがオープンしている間は存在し続けます．同じ効果はシンボリックリンクをファイルシステムの別の場所にバインドマウントすることでも得られます．

```
    # touch ~/uts                            # Create mount point
    # mount --bind /proc/27514/ns/uts ~/uts
```

Linux 3.8 以前では，/proc/PID/ns 以下のファイルは前述のような特殊なシンボリックリンクではなくハードリンクでした．加えて ipc, net, uts ファイルしか提供されていませんでした．

## 既に存在する名前空間に参加する: setns()

参加しているプロセスが存在しない名前空間を維持する事は，あとでそこにプロセスを加えるつもりがある場合には有効な事である．これは setns() システムコールの仕事である．このシステムコールを呼び出して，既存の名前空間にプロセスを参加させる事が可能である:

```
    int setns(int fd, int nstype);
```

もっと正確に言うと，setns() は，特定の種類の名前空間のインスタンスから呼び出したプロセスを切り離し，同じ種類の名前空間の他のインスタンスにプロセスを関連付けるということを行う．

fd 引数は参加させたい名前空間を指定する．これは /proc/PID/ns ディレクトリ内のシンボリックリンクの一つが指し示すファイルディスクリプタである．このファイルディスクリプタは，これらのシンボリックリンクの一つを直接 open するか，リンクを bind マウントしたファイルを open するかで取得可能である．

nstype 引数で fd が参照する名前空間の種類を呼び出し元でチェックする事が可能である．もし，この引数がゼロと指定された場合は，チェックは行われない．これは，呼び出し時に名前空間の種類を知っている場合か，種類について気にしない場合に役に立つ．この少し後で議論するサンプルプログラム (ns_exec.c) は後者である．つまりどんな種類の名前空間でも動くようにデザインされている．代わりに CLONE_NEW* 定数のうちの 1 つを指定すると，カーネルは fd が指定した名前空間と同じ種類かどうかをチェックする．例えばこれは，呼び出し元が UNIX ドメインソケット経由でファイルディスクリプタを受け取り，参照する名前空間の種類を確認する必要がある場合などに役に立つ．

setns() と execve() (もしくは他の exec() 系の関数) を使用して，シンプルだが役に立つツールを作る事が可能になる: 指定した名前空間に参加し，その名前空間内でコマンドを実行するようなプログラムである．

我々のプログラム (ns_exec.c, ソースは[[http://lwn.net/Articles/531271/|こちら]]) は 2 つ以上のコマンドライン引数を受け取る．最初の引数は /proc/PID/ns/* シンボリックリンクのパス名である (もしくはシンボリックリンクを bind マウントしたファイルのパス名)．残りの引数はシンボリックリンクが指す名前空間内で実行するプログラムの名前と，そのコマンドに与えるコマンドライン引数である．プログラムのキーとなるステップは以下である．

```
    fd = open(argv[1], O_RDONLY);   /* 名前空間のデスクリプタを取得 */

    setns(fd, 0);                   /* 名前空間に参加 */

    execvp(argv[2], &argv[2]);      /* 名前空間内でコマンドを実行 */
```

名前空間内で実行すると面白いプログラムはもちろんシェルである．demo_uts_namespaces で作成した新しい UTS 名前空間内でシェルを実行するために， ns_exec プログラムで先に作成した UTS 名前空間に対する bind マウントを使用することができる．

```
    # ./ns_exec ~/uts /bin/bash     # ~/uts is bound to /proc/27514/ns/uts
    My PID is: 28788
```

ホスト名と /proc/PID/ns/uts ファイルの inode 番号を比較することで，このシェルが demo_uts_namespaces で作られた子プロセスと同じ UTS 名前空間内にいることが確認できる．

```
    # hostname
    bizarro
    # readlink /proc/27514/ns/uts
    uts:[4026532338]
    # readlink /proc/$$/ns/uts      # $$ is replaced by shell's PID
    uts:[4026532338]
```

以前までのバージョンのカーネルでは，setns() を使って mount, PID, ユーザ名前空間に参加する事はできなかった．しかし 3.8 以降で，全ての名前空間に参加が可能になった．

## 名前空間から抜ける: unshare()

名前空間 API の最後のシステムコールは unshare() である．

<code c>
    int unshare(int flags);
```

unshare() システムコールは clone() と同様の機能を提供する．しかし，呼び出したプロセス上での操作となる．つまり，CLONE_NEW* ビットを引数として取り新しい名前空間を作成し，呼び出し元を名前空間のメンバーにする (clone() と同様に，我々がここでは扱わない名前空間の操作以外の機能を提供する)．unshare() の主な目的は，(clone() が行うような) 新しいプロセスやスレッドを作る必要なしに名前空間 (や他) の副次的な効果を隔離することである．

clone() システムコールの他の効果はひとまず置いといて，

<code c>clone(..., CLONE_NEWXXX, ....);```

という形式での呼び出しと，名前空間という点で，大体同じとなるのは以下のようなものである．

<code c>
    if (fork() == 0)
        unshare(CLONE_NEWXXX);      /* Executed in the child process */
```

unshare() システムコールの使い方の一つは，unshare コマンドの実装にある．このコマンドは，シェルとは別の名前空間内でコマンドを実行できるというものである．おおまかなコマンドの実行方法は

```
    unshare [options] program [arguments]
```

options は arguments を引数として program を実行する前に unshare する名前空間を指定するコマンドラインのオプションである．

unshare コマンドの実装のキーとなる部分はわかりやすい．

<code c>
     /* Code to initialize 'flags' according to command-line options
        omitted */

     unshare(flags);

     /* Now execute 'program' with 'arguments'; 'optind' is the index
        of the next command-line argument after options */

     execvp(argv[optind], &argv[optind]);
```

unshare コマンドのシンプルな実装 (unshare.c) は [[http://lwn.net/Articles/531826/|以下]] である．

以下のシェルのセッションでは，unshare.c を使い，別のマウント名前空間でシェルを実行している．先週の記事で注意したようにマウント名前空間は，プロセスのグループから見えるファイルシステムのマウントポイントの組を隔離するものであり，異なるマウント名前空間内のプロセスは異なったファイルシステム構造が持てるというものである．

```
    # echo $$                             # シェルの PID を見る
    8490
    # cat /proc/8490/mounts | grep mq     # 名前空間内のマウントを一つ見る
    mqueue /dev/mqueue mqueue rw,seclabel,relatime 0 0
    # readlink /proc/8490/ns/mnt          # マウント名前空間の ID を見る
    mnt:[4026531840]
    # ./unshare -m /bin/bash              # 別のマウント名前空間で新しいシェルを実行
    # readlink /proc/$$/ns/mnt            # マウント名前空間の ID を見る
    mnt:[4026532325]
```

2 度実行した readlink コマンドの出力を比べると，2 つのシェルは別のマウント名前空間にいることが示されている．片方の名前空間のマウントポイントの組を変化させ，他の名前空間で変化が見えるかどうかをチェックすることで，2 つのプログラムが別の名前空間内にいることを別の方法でデモする．

```
    # umount /dev/mqueue                  # シェルからマウントポイントを削除
    # cat /proc/$$/mounts | grep mq       # マウントポイントが存在しないことを確認
    # cat /proc/8490/mounts | grep mq     # 他の名前空間ではまだ存在したままかどうか?
    mqueue /dev/mqueue mqueue rw,seclabel,relatime 0 0
```

最後の 2 つのコマンドの出力から分かるように，/dev/mqueue マウントポイントは片方のマウント名前空間では見えなくなるが，別の名前空間では存在したままである．

## 最後に

この記事では，名前空間API の基本的な部分を見て，それをどのように組み合わせて使うかを示した．後に続く記事では，他のいくつかの名前空間でより深い部分を見ていく．特に PID とユーザ名前空間を見る．ユーザ名前空間は，アプリケーションが，従来は特権アプリに限られていたカーネルインターフェースの使用するための新しい可能性を広げるものである．
