# ユニフォームブロックのメモリレイアウト

OpenGLやVulkanのシェーディング言語であるGLSL。そのGLSLの機能の１つであるユニフォームブロック（CPUから渡すデータのかたまり）のメモリレイアウトの仕様がややこしく、筆者もこれがらみで時々ミスをすることがあります。

そんなメモリレイアウトについて情報をまとめてみました。

## メモリレイアウトと std140

ユニフォームブロックのメモリレイアウトとは、ユニフォームブロックとして渡される構造体に含まれる変数のアドレス配置のことを言います。例えば vec4 は必ず16の倍数のオフセットアドレスに格納される、といったものです。

もしかすると、メモリレイアウトよりもメモリアライメントと呼んだ方がしっくりくる人もいるかもしれません。公式のマニュアルでは Memory Layout と呼んでいるので、本記事の表記もそれにあわせることとします。

さて、そんなメモリレイアウトですが std140 形式というものがあります。 std430 形式というのもありますが今回は std140 のお話です。下記のようにコードを書くと、そのユニフォームブロックは std140 というルールに従ってメモリレイアウトが決定されます。

```glsl
layout(std140) uniform Data {
    mat4 mtx;
    vec4 vec;
};
```

このメモリレイアウトの仕様を把握しておかないと、C++からデータを渡すときに値がおかしくなるといった不具合を起こしてしまいます。

## std140 の仕様

std140 の仕様はこちらに掲載されています。

[OpenGL 4.5] 7.6.2.2 Standard Uniform Block Layout
https://khronos.org/registry/OpenGL/specs/gl/glspec45.core.pdf#page=158

まとめはこちら。

- 基本
  - int uint float bool : 4バイトアライメント
  - vec2 ivec2 uvec2 bvec2 : 8バイトアライメント
  - vec3 ivec3 uvec3 bvec3 : 16バイトアライメント
  - vec4 ivec4 uvec4 bvec4 : 16バイトアライメント
  - double : 8バイトアライメント
  - dvec2 : 16バイトアライメント
  - dvec3 dvec4 : ~~24~~16バイトアライメント
- 配列（アライメントは基本と同じ＆サイズは配列1要素につき16バイトの倍数に繰り上げ）
  - float[4] : 4バイトアライメント（サイズは 16 * 4 = 64バイト）
  - vec2[4] : 8バイトアライメント（サイズは 16 * 4 = 64バイト）
  - vec3[4] : 16バイトアライメント（サイズは 16 * 4 = 64バイト）
  - vec4[4] : 16バイトアライメント（サイズは 16 * 4 = 64バイト）
- 行列（ベクトルの配列と同じ仕様）
  - mat3 : vec3[3] と同じ。
  - mat4 : vec4[4] と同じ。
  - mat4x3 : (行優先)vec4\[3] (列優先)vec3\[4] と同じ。
- 行列の配列
  - mat3[8] : vec3[3 * 8] と同じ。
  - mat4[8] : vec4[3 * 8] と同じ。
  - mat4x3[8] : (行優先)vec4\[3 * 8] (列優先)vec3\[4 * 8] と同じ。
- ユーザー定義型
  - struct なメンバ変数 : 16バイトアライメント（structの持つメンバ変数の中で一番大きいアライメントを16バイトの倍数に繰り上げたものなので実質16バイトアライメント）
  - struct 配列なメンバ変数 : アライメントは↑と同じ（サイズは配列とルールが同じで16バイトの倍数に繰り上げ）

以上を踏まえて、つまづきポイントを補足していきます。

## よく使うスカラ型は4バイトアライメント（boolも含まれるので注意）

int、uint、float は4バイトなので4バイトアライメントです。

そして bool も4バイトアライメントになります。これよくミスります。bool自体は1bitあれば表現できるのですが、以下の条件により4バイトアライメントになります。

> (公式ドキュメントより)
> If the member is a scalar consuming N basic machine units, the base alignment is N

多くのGPUは4バイト(32bit)が基本単位になるため、boolも4バイトアライメントになる、という理由です。

このような仕様のためC++の構造体との不一致がおこりやすくなります。

この問題に対して、例えばboolは使わずuintで運用する、というのはシンプルな解決方法でオススメです。

ちなみに、スカラ型の中でも double のみ8バイトアライメントになります。

## 配列はC++の構造体と一致しなくなるケースが多い

配列の１要素が16バイト（vec4のサイズ）の倍数に繰り上げられます。ですので、C++だと float[4] と書くと16バイトになりますが std140 では64バイトになり、同じように構造体を記述するとズレが発生します。

この問題に対して、例えばvec4シリーズやmat4x*シリーズ（※行優先の場合）の配列しか書かない運用にする、というのがシンプルな解決方法になります。

## std430 では配列の仕様がC++と同じ

std430 形式は基本 std140 と同じですが配列の仕様が変わります。

具体的には、配列の「サイズは配列1要素につき16バイトの倍数に繰り上げ」という仕様がなくなり、C++の配列と同じような感覚で扱えます。

## (余談) std140 を指定しても std140 にならないことがある

メジャーなグラフィックボードのドライバ（nVidia・AMD・インテル）ではまずないのですが、一部の環境ではドライバにバグがありstd140の仕様通りに動かない環境があるようです。

筆者は仮想マシン環境で体験したことがありますが、そのときはmat4x3など後発のGLSLで導入された使い勝手が良い型で起こったと記憶しています。ドライバにもよると思うのですが、vec4などシンプルな型でのみ構成しておくとこのようなトラブルを回避できるかもしれません。

std140 関連ではないですが、一部のAndroidOS機でGLSLコードが意図通りな処理にならないという話も聞いたことがあります、ドライバまわりはちょっと恐いですね。

## おわり

以上、メモリレイアウトのお話でした。このような仕様は自分自身もそうですしチーム全員に全部覚えてもらうのも大変なため、筆者はミスが起こらないような運用にしがちです。そんな運用の仕方をしているのでメモリレイアウトについて熟知できていなかったのですが、この記事を書くことで復習できて良かったです。もし誤った情報を書いていたらツッコミいただけると嬉しいです。

リンク：[ゲームプログラマの小話-目次](http://www.10106.net/~hoboaki/wiki/index.php?%E3%82%B2%E3%83%BC%E3%83%A0%E3%83%97%E3%83%AD%E3%82%B0%E3%83%A9%E3%83%9E%E3%81%AE%E5%B0%8F%E8%A9%B1)

