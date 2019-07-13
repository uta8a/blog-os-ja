# はじめに

_Note_: This is the translations in Japanese of "Writing an OS in Rust". Original is [here][original].

[original]: https://os.phil-opp.com/

これは["Writing an OS in Rust"][original]の非公式日本語訳です。自分が["Writing an OS in Rust"][original]にとても興味があるのと、作者のPhilipp氏に了承をいただいたので、手ずから翻訳し公開しています。本サイトの内容の権利については、すべて原作者に帰属します。翻訳に間違いなどがあれば[こちら][issues]にご報告ください。興味があればぜひ[原文][original]の方も読むと面白いと思います。

[issues]: https://github.com/JohnTitor/blog-os-ja/issues
[original]: https://os.phil-opp.com/

## "Writing an OS in Rust" について

"Writing an OS in Rust" は、Rust で小さな OS(オペレーティングシステム)をつくるためのブログシリーズです。それぞれの記事は小さなチュートリアルとしてまとまっており、必要なコードはすべて記事内に含まれています。また [GitHub リポジトリ]でもコードを確認することができます。

[GitHub リポジトリ]: https://github.com/phil-opp/blog_os/

## 目次

### Bare Bones

- **[A Freestanding Rust Binary]**: 独自の OS カーネルを作成するための最初のステップは標準ライブラリにリンクしない Rust の実行可能ファイルを作成することです。これにより、基盤となる OS を必要とせずに、ベアメタル上で Rust コードを実行できるようになります。

[A Freestanding Rust Binary]: ./01-freestanding-rust-binary.html
