# メモリアライメントの話

コンシューマ機では割と知っておかないとおかしなことになるメモリのアライメントのお話です。自分の場合Windows上でゲームを作っていたときはほとんど意識していなかったため、コンシューマ機をやるときに「なにそれ？」となったトピックになります。

メモリアライメントはバイナリデータを読み込んでC++コードからアクセスしたり、適当なアドレスにnewしたデータをキャストする場合などに意識する必要が出てきます。

## アライメントとは

アライメントは alignment と書きます。英語の辞書を引くと『整列』といった意味で載っています。

アライメントを気にせずにデータをメモリ上に配置＆アクセスすると、パフォーマンスが遅くなったり、ハードウェアによってはCPUが例外をはいて強制終了する、といったことがおこります。

例えば int や float など、4バイトの変数はメモリアドレス的に4バイトの境界に置かないといけない、というCPUの制限があるハードがあります。この場合 **int や float は4バイトアライメントされたメモリアドレスに配置される必要がある** と言えます。いきなりメモリアドレスやら境界の話が出てきました。そのあたり、次の項で説明していきます。


## メモリアドレスとアライメント

最近のメモリは32bitや64bitのアドレスで扱うことが多いです。32bitマシンですとメモリアドレスは 0x20004C00 といった表記をします。

さて、先ほどの例で **int や float は4バイトアライメントされたメモリアドレスに配置される必要がある**　と言いました。

4バイトアライメントとは、**メモリアドレスが4でぴったり割り切れるアドレス**のことを言います。0x1000 や 0x1004 は4バイトアライメントされたアドレスですが、0x1002 や 0x1005 は4バイトアライメントされたアドレスではありません。

8バイトアライメントでしたら8でぴったり割り切れるアドレス、128バイトなら128で割り切れるアドレス、という感じです。


## 代表的なアライメント

最近のCPUで多く要求されるアライメントを紹介します。

|型|バイト数|要求されるアライメント|
|---|---|---|
|int|4|4|
|float|4|4|
|double|8|8|
|char|1|1|
|short|2|2|
|void* (32bit環境)|4|4|
|void* (64bit環境)|8|8|
|size_t (32bit環境)|4|4|
|size_t (64bit環境)|8|8|

この表からも分かるように、C++の組み込み型はバイト数と要求されるアライメントは等しいことがほとんどです。

ちなみに、下記のような構造体を書くと各メンバ変数のアライメントやそれを抱える構造体のアライメントもコンパイラのほうで自動で計算してくれます。

```c++
struct Human
{
    char   age;       // offsetof: 0x00    
    int    money;     // offsetof: 0x04
    short  height;    // offsetof: 0x08
    char   birthYear; // offsetof: 0x0a
    double weight;    // offsetof: 0x10
};
Human h; // doubleの変数を内部で持っているため 8バイトアライメントされたアドレスに配置される
```

また、C++11ではアライメントを調べる方法も標準で提供されるようになっています。

参考：alignof(C++11)
https://cpprefjp.github.io/lang/cpp11/alignof.html

## 特殊なアライメント

ハードウェアによってはCPU以外のデバイスとデータをやりとりする際に特殊なアライメントを要求されることがあります。

例えばこんな感じです。

- ファイルリードする際のデータの書き込み先のバッファは32バイトアライメントされている必要がある、
- GPUが参照するテクスチャは4096バイトアライメントされている必要がある。

特殊なアライメントが必要な場合は各OSのSDKにアライメントの定数値が公開されていることが多いので、それを使って適切なアライメントになるようにしてあげましょう。

## 誤ってアライメントミスしてしまう例

例えばこのようなアロック関数があったとします。

```c++
typedef unsigned char  byte_t;
void* myAlloc(size_t aSize)
{
    static byte_t localBuf[1024];
    static size_t localBufPos = 0;
    void* result = &localBuf[localBufPos];
    localBufPos += aSize;
    return result;
}

void func()
{
    int* a = (int*)myAlloc(sizeof(int)); // 4バイト確保
    char* b = (char*)myAlloc(sizeof(char)); // 1バイト確保
    int* c = (int*)myAlloc(sizeof(int)); // 危ない！！ 4バイト確保するが4バイトアライメントされていない値が返ってくる
}
```

アライメントを意識していないので、cのアドレスは4バイトアライメントされていないアドレスになってしまいます。

オリジナルの　alloc 関数を使う場合は、alloc関数の引数にアライメントの値を渡せるようにしたり、受け取った側がアライメントが正しくなるようにアドレスを補正してあげるといったことが必要になります。

## どうしてアライメントが要求されるのか

どうしてアライメントが要求されるのかというとハードウェア都合としか言えませんしケースバイケースです。

とはいえ、実例があったほうが想像しやすいと思うので１つだけ書いてみます。

例えば32バイトアライメントすると、メモリアドレスの下位4bitはゼロであることが確定します。そうすると32bit環境だとするとメモリアドレスは28bitで表現できるようになり、4bit余ります。その余った4bitには命令コードといった付加情報を付け加えることができるようになります。もし32bit全てをアドレスで使ってしまうとそういった付加情報を付け加えるためもう32bit用意しないといけなくなります。

アライメントの制約を設けることで『○○のアドレスの内容をレジスタにコピー』といった命令を32bitで表現できるようになり、その結果メモリ消費量や計算量が最適化される、といったストーリーが想像できます。

実際そんな低レイヤーのコードを書いたことはないのですが、1例としてこのようなケースもある気がします。

## マルチプラットフォーム対応

アライメントの要求はハードウェアによって変わります。例えば、ファイルリードバッファのアライメントについてAというゲーム機ではアライメントは128、Bというゲーム機では32というような感じです。

マルチプラットフォームなゲームエンジンを作る場合は、アプリ制作側にはこの数値はあまり意識させないようエンジン側で吸収したり、アライメントの数値をアプリ側から取得できるようにしたり、うまいこと対応しましょう。

## おわり

リンク：[ゲームプログラマの小話-目次](http://www.10106.net/~hoboaki/wiki/index.php?%E3%82%B2%E3%83%BC%E3%83%A0%E3%83%97%E3%83%AD%E3%82%B0%E3%83%A9%E3%83%9E%E3%81%AE%E5%B0%8F%E8%A9%B1)

