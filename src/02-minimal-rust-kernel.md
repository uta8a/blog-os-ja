# A Minimal Rust Kernel

**この記事は "Writing an OS in Rust" の翻訳版です。オリジナルは[こちら][original]になります。**

[original]: https://os.phil-opp.com/

この記事では、x86 アーキテクチャ用の小さな 64-bit カーネルを Rust でつくります。一つ前の記事の ["freestanding Rust binary"] をもとに、ブート可能なディスクイメージを作成し、画面に何か文字を出力します。

["freestanding Rust binary"]: ./01-freestanding-rust-binary.md

このブログの内容は [GitHub] 上で公開・開発されています。何か問題や質問などがあれば issue をたててください (訳注: リンクは原文(英語)のものになります)。この記事の完全なソースコードは [`post-01` ブランチ][post branch]にあります。

[GitHub]: https://github.com/phil-opp/blog_os
[post branch]: https://github.com/phil-opp/blog_os/tree/post-02

## 起動プロセス

コンピュータを起動すると、マザーボードの [ROM] に記録されているファームウェアコードの実行が開始されます。このコードは [パワーオンセルフテスト][power-on self-test] (POST)を実行し、利用可能な RAM を検出し、CPU とハードウェアを事前に初期化します。その後、ブート可能なディスクを検出し OS カーネルのブートを開始します。

[ROM]: https://en.wikipedia.org/wiki/Read-only_memory
[power-on self-test]: https://en.wikipedia.org/wiki/Power-on_self-test

x86 には、"Basic Input/Output System"(いわゆる **[BIOS]**)とより新しい"Unified Extensible Firmware Interface"(いわゆる **[UEFI]**)の2つのファームウェア標準があります。BIOS は古く時代遅れになっていますが、シンプルで1980年以降のどの x86 マシンでもサポートされています。一方、UEFI はより現代的で多くの機能を備えていますが、セットアップがより複雑です(少なくとも私から見ると)。

[BIOS]: https://en.wikipedia.org/wiki/BIOS
[UEFI]: https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface

現在、このブログでは BIOS のみをサポートしていますが、UEFI のサポートも予定されています。もし UEFI のサポートを手伝っていただけるのであれば、[GitHub issue] をご覧になってください。

[GitHub issue]: https://github.com/phil-opp/blog_os/issues/349

### BIOS の起動

ほとんどすべての x86 システムが BIOS ブートをサポートしています。これにはエミュレートされた BIOS を使用する新しい UEFI ベースのマシンも含まれます。これは過去のすべてのマシンで同じブートロジックを使用できるため非常に便利です。しかし、この広範な互換性は同時に BIOS ブートの最大の欠点でもあります。なぜなら、1980年代の古いブートローダでも動作できるように、ブート前に CPU が[リアルモード][real mode]と呼ばれる 16-bit 互換モードに設定されるからです。

最初から説明してきましょう:

コンピュータの電源を入れると、マザーボードにある特殊なフラッシュメモリから BIOS がロードされます。BIOS はハードウェアのセルフテストと初期化ルーチンを実行し、ブート可能なディスクを探します。見つかった場合、ディスクに格納されている実行可能コードの先頭 512 バイトにあるブートローダに制御が渡されます。ほとんどのブートローダは512バイトよりも大きいので、一般にブートローダは512バイトに収まる小さな第一段階とその後第一段階によって呼び出される第二段階に分割されます。

ブートローダはディスク上のカーネルイメージの場所を探し出し、それをメモリにロードする必要があります。また、CPU を 16-bit 互換の[リアルモード][real mode]から 32-bit 互換の[プロテクトモード][protected mode]に切り替えてから、64-bit レジスタと完全なメインメモリを使用できる 64-bit 互換の[ロングモード][long mode]に切り替えていく必要があります。3つ目の役割は、BIOS に特定の情報(メモリマップなど)を問い合わせ、それを OS カーネルに渡すことです。

[real mode]: https://ja.wikipedia.org/wiki/%E3%83%AA%E3%82%A2%E3%83%AB%E3%83%A2%E3%83%BC%E3%83%89
[protected mode]: https://en.wikipedia.org/wiki/Protected_mode
[long mode]: https://en.wikipedia.org/wiki/Long_mode

ブートローダを書くのは、アセンブリ言語と「このマジックナンバーをこのプロセッサレジスタに書き込む」といったような本筋から外れた多くの工程を要するので、少し面倒です。そのため、この記事ではブートローダの作成については触れず、カーネルにブートローダを自動的に追加する [bootimage] というツールを利用します。

[bootimage]: https://github.com/rust-osdev/bootimage

ブートローダを自分で作成することに興味のある方へ: そのためのブログシリーズがすでに計画されています。ご期待下さい！ <!-- , check out our “_[Writing a Bootloader]_” posts, where we explain in detail how a bootloader is built. -->

#### マルチブート

すべての OS が、単一の OS とのみ互換性がある独自のブートローダを実装するのを避けるために、[フリーソフトウェア財団][Free Software Foundation]は1995年に [Multiboot] と呼ばれるオープンなブートローダの規格を作成しました。この規格ではすべての Multiboot に対応するブートローダが同じく Multiboot に対応したすべての OS を読み込むことができるように、ブートローダと OS 間のインターフェースが定義されています。例えば [GNU GRUB] はこの実装例の一つであり、これは Linux システムで最も一般的なブートローダです。

[Free Software Foundation]: https://en.wikipedia.org/wiki/Free_Software_Foundation
[Multiboot]: https://wiki.osdev.org/Multiboot
[GNU GRUB]: https://en.wikipedia.org/wiki/GNU_GRUB

カーネルを Multiboot に対応させるには、カーネルファイルの先頭にいわゆる[Multiboot ヘッダ][Multiboot header]を追加するだけです。これにより、非常に簡単に GRUB で OS を起動させることができます。しかしながら、GRUB と Multiboot 規格にはいくつか問題もあります:

[Multiboot header]: https://www.gnu.org/software/grub/manual/multiboot/multiboot.html#OS-image-format

- 32-bit 互換のプロテクトモードのみをサポートします。64-bit 互換のロングモードに切り替えるには、CPU の設定を行う必要があります。
- カーネルではなくブートローダを単純にするように設計されています。例えば、GRUB が Multiboot ヘッダを見つけられるように、カーネルは[調整されたデフォルトのページサイズ][adjusted default page size]でリンクされる必要があります。また、カーネルに渡される[ブート情報][boot information]がクリーンな抽象化をサポートする代わりに、アーキテクチャに依存した多くの構造を含むことです。
- GRUB や Multiboot 規格についてはほとんど文書化されていません。
- カーネルファイルからブート可能なディスクイメージを作成するには GRUB がホストシステムにインストールされている必要があります。これにより、Windows または Mac での開発がより難しくなります。

[adjusted default page size]: https://wiki.osdev.org/Multiboot#Multiboot_2
[boot information]: https://www.gnu.org/software/grub/manual/multiboot/multiboot.html#Boot-information-format

このような欠点があるため、今回は GRUB や Multiboot 規格は使用しないことにしました。しかしながら、[bootimage] に Multiboot のサポートを追加して、カーネルを GRUB 上で読み込めるようにする予定です。もし Multiboot 準拠のカーネルにご興味があれば、このブログシリーズの[第1版][first edition]を調べてみてください。

[first edition]: ./first-edition/_index.md

### UEFI

(現時点では UEFI をサポートしていませんが、ぜひやりたいと思っています！もしお手伝い頂ける場合は、[GitHub issue]にてお知らせください。)

[GitHub issue]: (https://github.com/phil-opp/blog_os/issues/349)

## 最小限のカーネル

コンピュータがどのように起動するかはわかったので、次はいよいよ最小限のカーネルを作成していきます。ここでの目標は、起動時に画面に「Hello World!」を出力するディスクイメージを作成することです。一つ前の記事である ["freestanding Rust binary"] を前提として進めていきます。

["freestanding Rust binary"]: ./01-freestanding-rust-binary.md

覚えているでしょうか、私達は `cargo` を使って独立したバイナリを作成しました。ですが、OS によって異なるエントリポイント名とコンパイルフラグが必要でした。これはデフォルトでは `cargo` がホストシステム(つまり、今あなたが実行しているシステム)用にビルドするためです。例えば Windows 上で動作するカーネルにはあまり意味がないので、この仕組みは私達が求めるものではありません。代わりに、明確に定義されたターゲットシステム用にコンパイルします。

### Nightly チャンネルの Rust をインストールする

Rust には3つのリリースチャンネルがあります: _stable_、_beta_、そして _nightly_ です。[TRPL] はこれらのチャンネルの違いを詳しく説明しているので、少し時間をとってチェックしてみてください。OS をつくるためには、nightly チャンネルでしか利用できない実験的な機能が必要になるので、Rust の nightly バージョンをインストールする必要があります。

Rust のバージョンを管理するために [rustup] を使うことを強くおすすめします。stable や beta、nightly バージョンのコンパイラを同時に管理でき、アップデートも簡単です。rustup を使うと、`rustup override add nightly` を実行することで、カレントディレクトリで nightly バージョンのコンパイラを使用することができます。または、`nightly` と一緒に `rust-toolchain` と呼ばれるファイルをプロジェクトのルートディレクトリに追加することができます。nightly バージョンがインストールされているかどうかは `rustc --version` を実行することで分かります。うまくインストールされていれば、バージョン番号の最後に `-nightly` が含まれているはずです。

[TRPL]: https://doc.rust-lang.org/book/appendix-07-nightly-rust.html#choo-choo-release-channels-and-riding-the-trains
[rustup]: https://www.rustup.rs/

nightly コンパイラでは、ファイルの先頭でいわゆる _feature flags_ を使用することでさまざまな実験的機能を利用することができます。例えば、`main.rs` の一番上の行に `#![feature(asm)]` を書くことでインラインアセンブリの実験的な [`asm!` macro] を利用できるようになります。このような実験的機能は非常に不安定で、将来の Rust バージョンで予告なしに変更あるいは削除される可能性があることに注意してください。なので、今回は絶対に必要なときのみこのような実験的機能を使うことにします。

[`asm!` macro]: https://doc.rust-lang.org/nightly/unstable-book/language-features/asm.html

### Target Specification
Cargo supports different target systems through the `--target` parameter. The target is described by a so-called _[target triple]_, which describes the CPU architecture, the vendor, the operating system, and the [ABI]. For example, the `x86_64-unknown-linux-gnu` target triple describes a system with a `x86_64` CPU, no clear vendor and a Linux operating system with the GNU ABI. Rust supports [many different target triples][platform-support], including `arm-linux-androideabi` for Android or [`wasm32-unknown-unknown` for WebAssembly](https://www.hellorust.com/setup/wasm-target/).

[target triple]: https://clang.llvm.org/docs/CrossCompilation.html#target-triple
[ABI]: https://stackoverflow.com/a/2456882
[platform-support]: https://forge.rust-lang.org/platform-support.html

For our target system, however, we require some special configuration parameters (e.g. no underlying OS), so none of the [existing target triples][platform-support] fits. Fortunately, Rust allows us to define our own target through a JSON file. For example, a JSON file that describes the `x86_64-unknown-linux-gnu` target looks like this:

```json
{
    "llvm-target": "x86_64-unknown-linux-gnu",
    "data-layout": "e-m:e-i64:64-f80:128-n8:16:32:64-S128",
    "arch": "x86_64",
    "target-endian": "little",
    "target-pointer-width": "64",
    "target-c-int-width": "32",
    "os": "linux",
    "executables": true,
    "linker-flavor": "gcc",
    "pre-link-args": ["-m64"],
    "morestack": false
}
```

Most fields are required by LLVM to generate code for that platform. For example, the [`data-layout`] field defines the size of various integer, floating point, and pointer types. Then there are fields that Rust uses for conditional compilation, such as `target-pointer-width`. The third kind of fields define how the crate should be built. For example, the `pre-link-args` field specifies arguments passed to the [linker].

[`data-layout`]: https://llvm.org/docs/LangRef.html#data-layout
[linker]: https://en.wikipedia.org/wiki/Linker_(computing)

We also target `x86_64` systems with our kernel, so our target specification will look very similar to the one above. Let's start by creating a `x86_64-blog_os.json` file (choose any name you like) with the common content:

```json
{
    "llvm-target": "x86_64-unknown-none",
    "data-layout": "e-m:e-i64:64-f80:128-n8:16:32:64-S128",
    "arch": "x86_64",
    "target-endian": "little",
    "target-pointer-width": "64",
    "target-c-int-width": "32",
    "os": "none",
    "executables": true,
}
```

Note that we changed the OS in the `llvm-target` and the `os` field to `none`, because we will run on bare metal.

We add the following build-related entries:


```json
"linker-flavor": "ld.lld",
"linker": "rust-lld",
```

Instead of using the platform's default linker (which might not support Linux targets), we use the cross platform [LLD] linker that is shipped with Rust for linking our kernel.

[LLD]: https://lld.llvm.org/

```json
"panic-strategy": "abort",
```

This setting specifies that the target doesn't support [stack unwinding] on panic, so instead the program should abort directly. This has the same effect as the `panic = "abort"` option in our Cargo.toml, so we can remove it from there.

[stack unwinding]: http://www.bogotobogo.com/cplusplus/stackunwinding.php

```json
"disable-redzone": true,
```

We're writing a kernel, so we'll need to handle interrupts at some point. To do that safely, we have to disable a certain stack pointer optimization called the _“red zone”_, because it would cause stack corruptions otherwise. For more information, see our separate post about [disabling the red zone].

[disabling the red zone]: ./second-edition/extra/disable-red-zone/index.md

```json
"features": "-mmx,-sse,+soft-float",
```

The `features` field enables/disables target features. We disable the `mmx` and `sse` features by prefixing them with a minus and enable the `soft-float` feature by prefixing it with a plus. Note that there must be no spaces between different flags, otherwise LLVM fails to interpret the features string.

The `mmx` and `sse` features determine support for [Single Instruction Multiple Data (SIMD)] instructions, which can often speed up programs significantly. However, using the large SIMD registers in OS kernels leads to performance problems. The reason is that the kernel needs to restore all registers to their original state before continuing an interrupted program. This means that the kernel has to save the complete SIMD state to main memory on each system call or hardware interrupt. Since the SIMD state is very large (512–1600 bytes) and interrupts can occur very often, these additional save/restore operations considerably harm performance. To avoid this, we disable SIMD for our kernel (not for applications running on top!).

[Single Instruction Multiple Data (SIMD)]: https://en.wikipedia.org/wiki/SIMD

A problem with disabling SIMD is that floating point operations on `x86_64` require SIMD registers by default. To solve this problem, we add the `soft-float` feature, which emulates all floating point operations through software functions based on normal integers.

For more information, see our post on [disabling SIMD](./second-edition/extra/disable-simd/index.md).

#### Putting it Together
Our target specification file now looks like this:

```json
{
  "llvm-target": "x86_64-unknown-none",
  "data-layout": "e-m:e-i64:64-f80:128-n8:16:32:64-S128",
  "arch": "x86_64",
  "target-endian": "little",
  "target-pointer-width": "64",
  "target-c-int-width": "32",
  "os": "none",
  "executables": true,
  "linker-flavor": "ld.lld",
  "linker": "rust-lld",
  "panic-strategy": "abort",
  "disable-redzone": true,
  "features": "-mmx,-sse,+soft-float"
}
```

### Building our Kernel
Compiling for our new target will use Linux conventions (I'm not quite sure why, I assume that it's just LLVM's default). This means that we need an entry point named `_start` as described in the [previous post]:

[previous post]: ./second-edition/posts/01-freestanding-rust-binary/index.md

```rust
// src/main.rs

#![no_std] // don't link the Rust standard library
#![no_main] // disable all Rust-level entry points

use core::panic::PanicInfo;

/// This function is called on panic.
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}

#[no_mangle] // don't mangle the name of this function
pub extern "C" fn _start() -> ! {
    // this function is the entry point, since the linker looks for a function
    // named `_start` by default
    loop {}
}
```

Note that the entry point needs to be called `_start` regardless of your host OS. The Windows and macOS entry points from the previous post should be deleted.

We can now build the kernel for our new target by passing the name of the JSON file as `--target`:

```
> cargo build --target x86_64-blog_os.json

error[E0463]: can't find crate for `core`
```

It fails! The error tells us that the Rust compiler no longer finds the [`core` library]. This library contains basic Rust types such as `Result`, `Option`, and iterators, and is implicitly linked to all `no_std` crates.

[`core` library]: https://doc.rust-lang.org/nightly/core/index.html

The problem is that the core library is distributed together with the Rust compiler as a _precompiled_ library. So it is only valid for supported host triples (e.g., `x86_64-unknown-linux-gnu`) but not for our custom target. If we want to compile code for other targets, we need to recompile `core` for these targets first.

#### Cargo xbuild
That's where [`cargo xbuild`] comes in. It is a wrapper for `cargo build` that automatically cross-compiles `core` and other built-in libraries. We can install it by executing:

[`cargo xbuild`]: https://github.com/rust-osdev/cargo-xbuild

```
cargo install cargo-xbuild
```

The command depends on the rust source code, which we can install with `rustup component add rust-src`.

Now we can rerun the above command with `xbuild` instead of `build`:

```
> cargo xbuild --target x86_64-blog_os.json
   Compiling core v0.0.0 (/…/rust/src/libcore)
   Compiling compiler_builtins v0.1.5
   Compiling rustc-std-workspace-core v1.0.0 (/…/rust/src/tools/rustc-std-workspace-core)
   Compiling alloc v0.0.0 (/tmp/xargo.PB7fj9KZJhAI)
    Finished release [optimized + debuginfo] target(s) in 45.18s
   Compiling blog_os v0.1.0 (file:///…/blog_os)
    Finished dev [unoptimized + debuginfo] target(s) in 0.29 secs
```

We see that `cargo xbuild` cross-compiles the `core`, `compiler_builtin`, and `alloc` libraries for our new custom target. Since these libraries use a lot of unstable features internally, this only works with a [nightly Rust compiler]. Afterwards, `cargo xbuild` successfully compiles our `blog_os` crate.

[nightly Rust compiler]: ./second-edition/posts/01-freestanding-rust-binary/index.md#installing-rust-nightly

Now we are able to build our kernel for a bare metal target. However, our `_start` entry point, which will be called by the boot loader, is still empty. So let's output something to screen from it.

### Set a Default Target

To avoid passing the `--target` parameter on every invocation of `cargo xbuild`, we can override the default target. To do this, we create a [cargo configuration] file at `.cargo/config` with the following content:

[cargo configuration]: https://doc.rust-lang.org/cargo/reference/config.html

```toml
# in .cargo/config

[build]
target = "x86_64-blog_os.json"
```

This tells `cargo` to use our `x86_64-blog_os.json` target when no explicit `--target` argument is passed. This means that we can now build our kernel with a simple `cargo xbuild`. For more information on cargo configuration options, check out the [official documentation][cargo configuration].

### Printing to Screen
The easiest way to print text to the screen at this stage is the [VGA text buffer]. It is a special memory area mapped to the VGA hardware that contains the contents displayed on screen. It normally consists of 25 lines that each contain 80 character cells. Each character cell displays an ASCII character with some foreground and background colors. The screen output looks like this:

[VGA text buffer]: https://en.wikipedia.org/wiki/VGA-compatible_text_mode

![screen output for common ASCII characters](https://upload.wikimedia.org/wikipedia/commons/6/6d/Codepage-737.png)

We will discuss the exact layout of the VGA buffer in the next post, where we write a first small driver for it. For printing “Hello World!”, we just need to know that the buffer is located at address `0xb8000` and that each character cell consists of an ASCII byte and a color byte.

The implementation looks like this:

```rust
static HELLO: &[u8] = b"Hello World!";

#[no_mangle]
pub extern "C" fn _start() -> ! {
    let vga_buffer = 0xb8000 as *mut u8;

    for (i, &byte) in HELLO.iter().enumerate() {
        unsafe {
            *vga_buffer.offset(i as isize * 2) = byte;
            *vga_buffer.offset(i as isize * 2 + 1) = 0xb;
        }
    }

    loop {}
}
```

First, we cast the integer `0xb8000` into a [raw pointer]. Then we [iterate] over the bytes of the [static] `HELLO` [byte string]. We use the [`enumerate`] method to additionally get a running variable `i`. In the body of the for loop, we use the [`offset`] method to write the string byte and the corresponding color byte (`0xb` is a light cyan).

[iterate]: https://doc.rust-lang.org/stable/book/ch13-02-iterators.html
[static]: https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html#the-static-lifetime
[`enumerate`]: https://doc.rust-lang.org/core/iter/trait.Iterator.html#method.enumerate
[byte string]: https://doc.rust-lang.org/reference/tokens.html#byte-string-literals
[raw pointer]: https://doc.rust-lang.org/stable/book/ch19-01-unsafe-rust.html#dereferencing-a-raw-pointer
[`offset`]: https://doc.rust-lang.org/std/primitive.pointer.html#method.offset

Note that there's an [`unsafe`] block around all memory writes. The reason is that the Rust compiler can't prove that the raw pointers we create are valid. They could point anywhere and lead to data corruption. By putting them into an `unsafe` block we're basically telling the compiler that we are absolutely sure that the operations are valid. Note that an `unsafe` block does not turn off Rust's safety checks. It only allows you to do [four additional things].

[`unsafe`]: https://doc.rust-lang.org/stable/book/ch19-01-unsafe-rust.html
[four additional things]: https://doc.rust-lang.org/stable/book/ch19-01-unsafe-rust.html#unsafe-superpowers

I want to emphasize that **this is not the way we want to do things in Rust!** It's very easy to mess up when working with raw pointers inside unsafe blocks, for example, we could easily write behind the buffer's end if we're not careful.

So we want to minimize the use of `unsafe` as much as possible. Rust gives us the ability to do this by creating safe abstractions. For example, we could create a VGA buffer type that encapsulates all unsafety and ensures that it is _impossible_ to do anything wrong from the outside. This way, we would only need minimal amounts of `unsafe` and can be sure that we don't violate [memory safety]. We will create such a safe VGA buffer abstraction in the next post.

[memory safety]: https://en.wikipedia.org/wiki/Memory_safety

## Running our Kernel

Now that we have an executable that does something perceptible, it is time to run it. First, we need to turn our compiled kernel into a bootable disk image by linking it with a bootloader. Then we can run the disk image in the [QEMU] virtual machine or boot it on real hardware using an USB stick.

### Creating a Bootimage

To turn our compiled kernel into a bootable disk image, we need to link it with a bootloader. As we learned in the [section about booting], the bootloader is responsible for initializing the CPU and loading our kernel.

[section about booting]: #the-boot-process

Instead of writing our own bootloader, which is a project on its own, we use the [`bootloader`] crate. This crate implements a basic BIOS bootloader without any C dependencies, just Rust and inline assembly. To use it for booting our kernel, we need to add a dependency on it:

[`bootloader`]: https://crates.io/crates/bootloader

```toml
# in Cargo.toml

[dependencies]
bootloader = "0.6.0"
```

Adding the bootloader as dependency is not enough to actually create a bootable disk image. The problem is that we need to link our kernel with the bootloader after compilation, but cargo has no support for [post-build scripts].

[post-build scripts]: https://github.com/rust-lang/cargo/issues/545

To solve this problem, we created a tool named `bootimage` that first compiles the kernel and bootloader, and then links them together to create a bootable disk image. To install the tool, execute the following command in your terminal:

```
cargo install bootimage --version "^0.7.3"
```

The `^0.7.3` is a so-called [_caret requirement_], which means "version `0.7.3` or a later compatible version". So if we find a bug and publish version `0.7.4` or `0.7.5`, cargo would automatically use the latest version, as long as it is still a version `0.7.x`. However, it wouldn't choose version `0.8.0`, because it is not considered as compatible. Note that dependencies in your `Cargo.toml` are caret requirements by default, so the same rules are applied to our bootloader dependency.

[_caret requirement_]: https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html#caret-requirements

For running `bootimage` and building the bootloader, you need to have the `llvm-tools-preview` rustup component installed. You can do so by executing `rustup component add llvm-tools-preview`.

After installing `bootimage` and adding the `llvm-tools-preview` component, we can create a bootable disk image by executing:

```
> cargo bootimage
```

We see that the tool recompiles our kernel using `cargo xbuild`, so it will automatically pick up any changes you make. Afterwards it compiles the bootloader, which might take a while. Like all crate dependencies it is only built once and then cached, so subsequent builds will be much faster. Finally, `bootimage` combines the bootloader and your kernel to a bootable disk image.

After executing the command, you should see a bootable disk image named `bootimage-blog_os.bin` in your `target/x86_64-blog_os/debug` directory. You can boot it in a virtual machine or copy it to an USB drive to boot it on real hardware. (Note that this is not a CD image, which have a different format, so burning it to a CD doesn't work).

#### How does it work?
The `bootimage` tool performs the following steps behind the scenes:

- It compiles our kernel to an [ELF] file.
- It compiles the bootloader dependency as a standalone executable.
- It links the bytes of the kernel ELF file to the bootloader.

[ELF]: https://en.wikipedia.org/wiki/Executable_and_Linkable_Format
[rust-osdev/bootloader]: https://github.com/rust-osdev/bootloader

When booted, the bootloader reads and parses the appended ELF file. It then maps the program segments to virtual addresses in the page tables, zeroes the `.bss` section, and sets up a stack. Finally, it reads the entry point address (our `_start` function) and jumps to it.

### Booting it in QEMU

We can now boot the disk image in a virtual machine. To boot it in [QEMU], execute the following command:

[QEMU]: https://www.qemu.org/

```
> qemu-system-x86_64 -drive format=raw,file=bootimage-blog_os.bin
warning: TCG doesn't support requested feature: CPUID.01H:ECX.vmx [bit 5]
```

This opens a separate window with that looks like this:

![QEMU showing "Hello World!"](qemu.png)

We see that our "Hello World!" is visible on the screen.

### Real Machine

It is also possible to write it to an USB stick and boot it on a real machine:

```
> dd if=target/x86_64-blog_os/debug/bootimage-blog_os.bin of=/dev/sdX && sync
```

Where `sdX` is the device name of your USB stick. **Be careful** to choose the correct device name, because everything on that device is overwritten.

After writing the image to the USB stick, you can run it on real hardware by booting from it. You probably need to use a special boot menu or change the boot order in your BIOS configuration to boot from the USB stick. Note that it currently doesn't work for UEFI machines, since the `bootloader` crate has no UEFI support yet.

### Using `cargo run`

To make it easier to run our kernel in QEMU, we can set the `runner` configuration key for cargo:

```toml
# in .cargo/config

[target.'cfg(target_os = "none")']
runner = "bootimage runner"
```

The `target.'cfg(target_os = "none")'` table applies to all targets that have set the `"os"` field of their target configuration file to `"none"`. This includes our `x86_64-blog_os.json` target. The `runner` key specifies the command that should be invoked for `cargo run`. The command is run after a successful build with the executable path passed as first argument. See the [cargo documentation][cargo configuration] for more details.

The `bootimage runner` command is specifically designed to be usable as a `runner` executable. It links the given executable with the project's bootloader dependency and then launches QEMU. See the [Readme of `bootimage`] for more details and possible configuration options.

[Readme of `bootimage`]: https://github.com/rust-osdev/bootimage

Now we can use `cargo xrun` to compile our kernel and boot it in QEMU. Like `xbuild`, the `xrun` subcommand builds the sysroot crates before invoking the actual cargo command. The subcommand is also provided by `cargo-xbuild`, so you don't need to install an additional tool.

## What's next?

In the next post, we will explore the VGA text buffer in more detail and write a safe interface for it. We will also add support for the `println` macro.