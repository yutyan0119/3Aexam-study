# ファイルシステムの役割
二次記憶装置を簡便な共通のインタフェースで読めるようにする

# API
`open(path, flags, mode)`
- path: ファイルのパス
- flags: ファイルを開く方法
  - `O_RDONLY`: 読み込み専用
  - `O_WRONLY`: 書き込み専用
  - `O_RDWR`: 読み書き両方
  - `O_APPEND`: ファイルの末尾に追記
  - `O_CREAT`: ファイルが存在しない場合は作成
  - `O_EXCL`: ファイルが存在する場合はエラー
  - `O_TRUNC`: ファイルを空にする

などなど
ファイルディスクリプタにはファイルオフセット（次の読み書きが行われる場所）の情報が含まれている。

`read(fd, buf, sz)`
- fd: ファイルディスクリプタ
- buf: 読み込んだデータを格納するバッファ
- sz: 読み込むバイト数

実際に読み込まれたバイト数を返し、オフセットをその分だけ進ませる。

`write(fd, buf, sz)`
- fd: ファイルディスクリプタ
- buf: 書き込むデータが格納されたバッファ
- sz: 書き込むバイト数

場合によってはファイルが伸長する。実際に書き込まれたバイト数を返し、ファイルオフセットがその分だけ進む。

`lseek(fd, offset, whence)`
- fd: ファイルディスクリプタ
- offset: オフセット
- whence: オフセットの基準
  - `SEEK_SET`: ファイルの先頭
  - `SEEK_CUR`: 現在のオフセット
  - `SEEK_END`: ファイルの末尾

ファイルのオフセットを引数でしたものに変更する。

`posix_fallocate(fd, offset, len)`
`fallocate(fd, mode, offset, len)`
- fd: ファイルディスクリプタ
- mode: ファイルの拡張方法
  - `FALLOC_FL_KEEP_SIZE`: ファイルサイズを変更しない
  - `FALLOC_FL_PUNCH_HOLE`: ファイルの中身を空にする
- offset: ファイルの先頭からのオフセット
- len: ファイルの拡張する長さ

ファイルの[offset, offset+len]バイトの範囲を必要ならばファイルを伸長し、その範囲を0で埋める。

`ftruncate(fd, len)`
- fd: ファイルディスクリプタ
- len: ファイルのサイズ

ファイルの大きさをlenバイトに短縮する

`close(fd)`
- fd: ファイルディスクリプタ
 
openしたファイルを閉じる

`unlink(path)`
- path: ファイルのパス

ファイルを消す

# `mmap`
`void*a = mmap(addr, len, prot, flags, fd, offset)`
- addr: メモリのアドレス
- len: メモリの長さ
- prot: メモリの保護
  - `PROT_READ`: 読み込み可能
  - `PROT_WRITE`: 書き込み可能
  - `PROT_EXEC`: 実行可能
- flags: メモリのマッピング方法
  - `MAP_SHARED`: メモリ領域が複数のプロセスで共有される
    - 書き込みがファイルへ反映され、プロセス間で共有される。
  - `MAP_PRIVATE`: マッピングがコピーオンライトになる。
    - 書き込みがファイルへ反映されず、プロセス間でも共有されない。
- fd: ファイルディスクリプタ
- offset: ファイルの先頭からのオフセット

基本はfdに対応するバイトのoffsetからlenバイトをaddrにマッピングするという動作をする。fdが-1のときはただメモリを割り当てる。a[i]が、ファイルの(offset + i)バイト目を指すようになる。
すなわち、メモリが新たに割り当てられることにもなる。

以下の役割を持つ。
1. ファイルをメモリ上にあるかのように読み書きする
2. メモリを割り当てる（sbrk）
3. 割り当てるアドレスを指定する
4. 書き込み保護を設定する

## ファイル読み出し

```c
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>

int main() {
  int fd = open("test.txt", O_RDONLY);
  struct stat st;
  fstat(fd, &st);
  char *p = mmap(NULL, st.st_size, PROT_READ, MAP_SHARED, fd, 0);
  for (long i = 0; i < st.st_size; i++) {
    printf("%c", p[i]);
  }
  munmap(p, st.st_size);
  close(fd);
}
```

## ファイル書き出し

```c
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>

int main() {
  int fd = open("test.txt", O_RDWR|O_TRUNC|O_CREAT, 0777);
  posix_fallocate(fd, 0, 100000000); //サイズを100MBにする
  struct stat st;
  fstat(fd, &st);
  char *p = mmap(NULL, st.st_size, PROT_WRITE, MAP_SHARED, fd, 0);
  for (long i = 0; i < st.st_size; i++) {
    p[i] = i % 128;
  }
  munmap(p, st.st_size);
  close(fd);
}
```

## メモリ割り当て

```c
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>

int main() {
  char *p = mmap(NULL, 100000000, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
  munmap(p, 100000000);
}
```

1. 保護属性を指定出来る
2. 個別に開放出来る
3. 割り当てるアドレスを第一引数で指定出来る

といった利点がある。

## 実装概要
1. [a, a+sz) を論理的に割り当てる
2. ファイルとの対応関係も記録する
3. ページテーブル上では、ページは不在としておく。
4. ページフォルト発生時にアドレス空間記述表を見て、ファイルに対応した領域であればファイルの対応する領域を読み込む

## 共有マッピングとプライベートマッピング
違いは物理メモリをどう使うか。

### `MAP_SHARED`
- 同じ場所に対しては同じ物理アドレスを用いる（実はキャッシュをそのまま使う）

### `MAP_PRIVATE`
- 書き込みが実際に起こるまでは、物理メモリを共有し、書き込みが発生したら異なる物理ページを利用する。（コピーオンライト）

※readは各プロセスが異なる物理アドレスを用いる

## 効果的な場面
1. ファイルのごく一部を飛び飛びにアクセスする
   1. lseek+readは面倒、readですべて読み込むのは無駄なのでmmapでファイル全体をマップする
2. 多くのプロセスが同じファイルをアクセスする
   1. 同一領域に対する物理ページは共有される（MAP_SHARED）
   2. 書き込まれていない領域に対する物理ページは共有(MAP_PRIVATE)
   3. readは各プロセスが異なる物理アドレスを用いる

3. プログラムの読み込み
   1. 上にあげた性質をもつため。

# コピーオンライト

変更するまでは物理メモリを共有する方式のこと。物理ページを共有しつつ、MMUの機能で書き込み不可の属性をつけておく。書き込み時に保護例外を発生させ、OSが対応する物理メモリをコピーし、書き込まれたページ→新しい物理ページにマッピングを変更する。



# キャッシュ
一度読んだファイルをメモリ上に保持し、一定期間保持する。

1. read要求された部分がキャッシュにない
   1. カーネル内にキャッシュのための領域を割り当てる
   2. 二次記憶からキャッシュ上に読み込む
   3. プロセスのページにもコピー
2. キャッシュ上にある
   1. キャッシュからプロセスのページにコピーする

# 先読み
プロセスが要求していない部分を先に読んでおくこと。実際にはファイルの読み出しが連続した部分で何回か行われた時点で発動する。

