# この文書について

この文書は [lwn.net](https://lwn.net/) において 2013 年に公開された Namespaces in Operation シリーズのパート 1  ([http://lwn.net/Articles/531114/](http://lwn.net/Articles/531114/))を翻訳したものです。

この文書のライセンスは原文と同じく、[Creative Commons CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/) のもとに提供されています。

# Namespaces in operation, part 1: namespace 概観

Linux 3.8 のマージウィンドウでは，Eric Biederman のかなり大きなサイズの[ユーザ名前空間と関連するパッチ](http://lwn.net/Articles/528078/)群が受け入れられているのが目に入る。完了するにはまだ細かい点が残っているものの (例えば多数の Linux ファイルシステムがまだユーザ名前空間対応していない)、ユーザ名前空間の実装は機能的な部分では完成した。

ユーザ名前空間の実装の完成は、いくつかの理由から、少し大きな出来事である。最初に、(Linux 2.6.23 で) ユーザ名前空間の最初の一歩を踏み出してから 5 年余りも費やしているという事実が示すように、最も複雑な名前空間の実装が完了したことを示す。次に、名前空間の実装は現在、存在する名前空間のほとんどの実装はほぼ完成しているという、ある意味 "stable point" にいるということである。これは名前空間の実装が全て完了したことを意味するわけではない: 将来的に他の名前空間が追加されるかもしれない。そして、現在存在する名前空間に対する拡張もなされるだろう。例えば[カーネルログに対する名前空間による隔離](http://lwn.net/Articles/527342/)の追加のような。最後に、ユーザ名前空間の実装における最近の変更は、名前空間の使われ方の視点から、大きな変革のようなものであると言える。それは、Linux 3.8 から、特権を持たないプロセスがユーザ名前空間を作り、その中で完全な特権を持つことが出来るのである。これは同様に、他のタイプの名前空間もユーザ名前空間の中で作成できるということである。

それゆえ現時点は、名前空間の全体像を眺め、名前空間 API の使い方を見る丁度良い時であるように見える。これは、それを行う記事のシリーズの最初のものである。この記事では、現在利用可能な名前空間の概略を示す。続く記事では、どのように名前空間の API をプログラムの中で使えるかを示す。

## 名前空間

現在、Linux には 6 つの異なる名前空間が実装されている。それぞれの名前空間の目的は、名前空間内のプロセスが、グローバルリソースの隔離されたインスタンスを持つように見える抽象化を行なって、特定のグローバルリソースをラップすることである。名前空間の一般的な目的の一つは、プロセスのグループからは自分たちだけがシステム上に存在しているように見える機能を提供する、軽量な仮想化のツールである[コンテナ](http://lwn.net/Articles/524952/)の実装をサポートすることである (他の目的の場合も同様な機能である)。

以下の議論では、実装された順に (少なくとも実装が完成した順に) 名前空間を挙げていく。括弧内に挙られた `CLONE_NEW*` 指示子は、以降の記事で述べる名前空間の API (`clone(), unshare(), setns()`) を使うときの名前空間を示すのに使われる定数名である。

[マウント名前空間](http://lwn.net/2001/0301/a/namespaces.php3) (`CLONE_NEWNS`, Linux 2.4.19) はプロセスグループから見えるマウントポイントの組を隔離する。マウント名前空間の追加で、[`mount()`](http://man7.org/linux/man-pages/man2/mount.2.html)と[`umount()`](http://man7.org/linux/man-pages/man2/umount.2.html)システムコールは、システム上の全てのプロセスから見えるグローバルなマウントポイントに対する操作を止め、代わりに呼び出したプロセスに紐付いたマウント名前空間にだけ影響を与える操作を行う。

マウント名前空間が呼び出されると、chroot jail と同様の環境が作成される。しかし、`chroot()` システムコールの使用と比べると、マウント名前空間は、このようなタスクに対してよりセキュアで柔軟性がある。他にマウント名前空間の[より洗練された使い方](http://www.ibm.com/developerworks/linux/library/l-mount-namespaces/index.html)をすることも可能である。例えば、分離されたマウント名前空間は、マスター・スレーブ関係で設定することも可能である。なので、マウントイベントは自動的にある名前空間から別の名前空間に伝搬する。これは、例えば、ある名前空間内でマウントされた光学ディスクデバイスを、自動的に他の名前空間で出現させることもできる。

マウント名前空間は、2002 年に Linux に実装された名前空間の最初のものである。この事実が "NEWNS" という他に比べると一般的な名前 ("new namespace" の略) が付いている理由である。その時点では、誰も他の異なるタイプの名前空間が将来必要になってくるであろう事を考えていなかったのである。

[UTS名前空間](http://lwn.net/Articles/179345/) (`CLONE_NEWUTS`, Linux 2.6.19) は 2 つのシステムの識別子を隔離する。その識別子とは、[`uname()`](http://man7.org/linux/man-pages/man2/uname.2.html) システムコールによって返される `nodename` と `domainname` である。この名前は `sethostname()` と `setdomainname()` システムコールで設定される。コンテナのコンテキストでは、UTS 名前空間機能は、コンテナごとに自分のホスト名と NIS ドメインネームを持つことができるようになる。これは、名前に基づいたアクションを行う初期化や設定スクリプトで役に立つ。"UTS" とは `uname()` システムコールに渡される構造体 `struct utsname` に由来する。構造体の名前は "UNIX Time-sharing System" に由来する。

[IPC名前空間](http://lwn.net/Articles/187274/) (`CLONE_NEWIPC`, Linux 2.6.19) は特定の interprocess communication (IPC) リソース、すなわち、[System V IPC](http://man7.org/linux/man-pages/man7/svipc.7.html)オブジェクfトと (Linux 2.6.30 以降の) [POSIX message queues](http://man7.org/linux/man-pages/man7/mq_overview.7.html)を隔離する。これらの IPC メカニズムの共通の特徴は、IPC オブジェクトをファイルシステムのパス名以外の仕組みで特定するということである。IPC 名前空間はそれぞれ自身の System V IPC 識別子と POSIX メッセージキューファイルシステムを持つ。

[PID名前空間](http://lwn.net/Articles/259217/) (`CLONE_NEWPID`, Linux 2.6.24) はプロセス ID 番号の空間を隔離する。言い換えると、異なる PID 名前空間内のプロセス同士は同じ PID を持つことが可能となるということである。PID名前空間の主要な利点は、コンテナ内のプロセスが同じプロセス ID を持ったまま、コンテナがホスト間をマイグレーションする事が可能になる事である。PID 名前空間により、様々なシステム初期化のタスクを管理したり、プロセス終了時の孤児プロセスを刈り取ったりする『全てのプロセスの祖先』である `init` (PID 1) をコンテナごとに持つことも可能になる。

特定の PID 名前空間の視点からすると、あるプロセスは 2 つの PID を持つことになる。つまり、名前空間内の PID とホストシステム上の名前空間外の PID である。PID 名前空間はネスト可能である。あるプロセスは自身が属する PID 名前空間から root PID 名前空間に至るまでの階層構造のそれぞれ階層に対して 1 つの PID を持つ。プロセスは自身の属する PID 名前空間と、ネストした下位の PID 名前空間のプロセスからのみ見える (例えば `kill()` でシグナルを送ったりするなど)。

[ネットワーク名前空間](http://lwn.net/Articles/219794/) (`CLONE_NEWNET`, Linux 2.6.24 で実装され、大部分の実装が完成したのが大体 Linux 2.6.29) はネットワークに関係するシステムリソースの隔離機能を提供する。したがって、それぞれのネットワーク名前空間は自身のネットワークデバイス、IPアドレス、IPルーティングテーブル、`/proc/net` ディレクトリ、ポート番号などを持つ。

ネットワーク名前空間は，ネットワークの視点からコンテナを便利にする。つまりコンテナはそれぞれ、自身の (仮想的な) ネットワークデバイスを持つことが可能であり、名前空間ごとのポート番号空間にバインドするアプリケーションを持つことが可能になる。ホストシステム内の適切なルーティングルールがパケットを特定のコンテナに関係するネットワークデバイスを送ることが可能になる。したがって、例えば複数のコンテナ化されたウェブサーバが同じホスト上で動くことも可能であり、それぞれのサーバは自身の (コンテナごとの) ネットワーク名前空間内でポート 80 番をバインドしているということが可能である。

[ユーザ名前空間](http://lwn.net/Articles/528078/) (`CLONE_NEWUSER`, Linux 2.6.23 で実装が始まり、Linux 3.8 で完成) は、ユーザとグループ ID 番号の空間を隔離する。言い換えると、プロセスのユーザとグループ ID がユーザ名前空間の内部と外部で異なる値を取る事が可能になる。最も興味深いケースとして、ユーザ名前空間の外ではプロセスが通常の特権を持たないユーザを持ち、同時に名前空間内ではユーザ ID が 0 を持つというケースがある。これは、プロセスがユーザ名前空間内では root の特権をフルに持ちながら、名前空間外では特権のない操作を行うという事を意味する。

Linux 3.8 から、非特権プロセスがユーザ名前空間を作成することが可能となる。これで、アプリケーションにとって面白い新しい可能性が多数広がる。非特権プロセスがユーザ名前空間内では root 特権を保持できるので、非特権プロセスは今まで root に限られていた機能へのアクセスが可能になる。Eric Biederman はユーザ名前空間の実装を安全で正しく行うために多大な努力を払った。しかし、この作業で行われた変更は巧妙で広大な範囲に及ぶ。それゆえ、ユーザ名前空間はまだ未知のセキュリティ問題を持ち、将来的に修正が続くかもしれない。

## 最後に

最初に Linux に名前空間が実装されて 10 年余りになる。それ以来、名前空間のコンセプトは、従来はシステム全体のスコープであった様々なグローバルリソースの隔離を行うためのより一般的なフレームワークへと広がってきた。その結果名前空間は、コンテナ形式の軽量な仮想化システムを完成させるための基礎を提供するようになった。名前空間のコンセプトは広がってきたので、関連する API も拡張された。単一のシステムコール (`clone()`) と 1,2 の `/proc` ファイルから、いくつかの他のシステムコールとより多数の `/proc` 以下のファイルへと。この API の詳細はこの記事の後に続くテーマの記事で説明する。

## シリーズの索引

以下はこのシリーズでこの後公開される記事のリストとサンプルプログラムである:

* [Part 2: the namespaces API](http://lwn.net/Articles/531381/)
    * [demo_uts_namespaces.c](http://lwn.net/Articles/531245/): demonstrate the use of UTS namespaces
    * [ns_exec.c](http://lwn.net/Articles/531271/): join a namespace using setns() and execute a command
    * [unshare.c](http://lwn.net/Articles/531826/): unshare namespaces and execute a command; similar in concept to unshare(1)
* [Part 3: PID namespaces](http://lwn.net/Articles/531419/)
    * [pidns_init_sleep.c](http://lwn.net/Articles/532741/): demonstrate PID namespaces
	* [multi_pidns.c](http://lwn.net/Articles/532745/): create a series of child processes in nested PID namespaces
* [Part 4: more on PID namespaces](http://lwn.net/Articles/532748/)
    * [ns_child_exec.c](http://lwn.net/Articles/533492/): create a child process that executes a shell command in new namespace(s)
	* [simple_init.c](http://lwn.net/Articles/533493/): a simple init(1)-style program to be used as the init program in a PID namespace
	* [orphan.c](http://lwn.net/Articles/533494/): demonstrate that a child becomes orphaned and is adopted by the init process when its parent exits
	* [ns_run.c](http://lwn.net/Articles/533495/): join one or more namespaces using setns() and execute a command in those namespaces, possibly inside a child process; similar in concept to nsenter(1)
* [Part 5: user namespaces](http://lwn.net/Articles/532593/)
    * [demo_userns.c](http://lwn.net/Articles/539941/): simple program to create a user namespace and display process credentials and capabilities
	* [userns_child_exec.c](http://lwn.net/Articles/539940/): create a child process that executes a shell command in new namespace(s); similar to ns_child_exec.c, but with additional options for use with user namespaces
* [Part 6: more on user namespaces](http://lwn.net/Articles/540087/)
    * [userns_setns_test.c](http://lwn.net/Articles/541230/): test the operation of setns() from two different user namespaces.
* [Part 7: network namespaces](http://lwn.net/Articles/580893/)
* [Mount namespaces and shared subtrees](http://lwn.net/Articles/689856/)
* [Mount namespaces, mount propagation, and unbindable mounts](http://lwn.net/Articles/690679/)
