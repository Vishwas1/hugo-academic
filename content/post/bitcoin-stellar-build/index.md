+++
title = "Debugging Stellar and Bitcoin code bases integration build issue"
subtitle = ""
summary = "In this blog I am talking about how did we debug and fix build issue when we tried to integrate Stellar and Bitcoind code bases"
date = 2019-03-07T10:00:00+05:30
draft = false
authors = ["Vishwas Anand"]
featured = false
tags = ["blockchain", "stellar", "bitcoin", "autotools", "c++", "automake", "autoconf", "build"]
categories = ["Blockchain"]
# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder. 
[image]
  # Caption (optional)
  caption = ""

  # Focal point (optional)
  # Options: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight
  focal_point = ""
+++

# Introduction

Currently, our Blockchain team is working on **Zagg Protocol**, which is a privacy-preserving protocol on public blockchain for tracking assets and values using a hybrid model of both Accounts and UTXOs. More about **Zagg** can be learned on its official [website](https://zaggprotocol.com/). Best fit blockchain for the business use case of **Zagg** is [**Stellar**](https://www.stellar.org/), but the ecosystem of Stellar (which is an account based model) does not support privacy - transactions are public. 
Zagg needed privacy and one of the way to achieve private or shielded transaction is by using notion of **Zero Knowledge proof**(ZKP), which is a method by which one party (the prover) can prove to another party (the verifier) that they know a value x, without conveying any information apart from the fact that they know the value x. 

In order to implement ZKP, we had to use the UTXO based blockchain model. Hence we boiled down on using a hybrid model - Account as well as UTXO type. To implement UTXO functionality, we built [**Bitcoin-core**](https://github.com/bitcoin/bitcoin) blockchain code alongside [**Stellar-core**](https://github.com/stellar/stellar-core). In this blog I wanted to talk about the following thing: 

- How did we build bitcoin-core with the stellar-core code? 
- What all approaches did we take to integrate these two code bases?
- What all challenges did we face while building them together? 
- Finally, what actions we had taken to solve those issues?

**Note**: If you want to understand the basic concepts behind UTXO and Account based blockchain models, you can checkout out my blog [here](https://labs.imaginea.com/post/utxo/) or else can watch my talk on this topic on [youtube](https://www.youtube.com/watch?v=7fNfCSDvCHw&t). Beside this, those who wants to understand **ZKP**, can go through a short but very informative blog written by one of my collegue, *Abhijit Sinha*, [here](https://labs.imaginea.com/post/zkp/) 

## Key Take Away 

Take away from this blog for you should be to understand the build environment of c++ code bases (however,   very basic) and to understand the flow or action to take in order to integrate and build two different code bases. Consider this is as an example to understand that, you might not face a similar issue or might not be working on the same problem statement but the concept and ideas can be learned.

> If you are an experienced c++ developer, then this blog might not be very helpful for you but you can still read, validate and can drop your valuable suggestions or comments on the approach which we have taken :)

## Problem Statement

To integrate *Bitcoin-core* with *Stellar-core* to leverage UTXO functionality from Bitcoin-core code base. 

## Proposed Solution

- Convert Bitcoin-core into a static library (.a file, essentially an archive of object(.o) files).
- Include Bitcoin-core as a submodule to Stellar-core.
- Link Bitcoin-core library and its dependencies to stellar-core executable.

## Implementation

In the first version ([v0.1](https://github.com/zagg-protocol/zagg-core#protocol-v01)) we wanted to integrate Bitcoin code base into the Stellar fork to achieve dual accounts balance and UTXO balances. [Here](https://github.com/zagg-protocol/zagg-core#development-approach) is details of other versions of this protocol. In this blog, we are going to focus on v0.1 only.


### Let's brush up our basics first!

#### C++ project structure

Usually, a c++ project with Autotools configuration has the following structure:

```
root\
    lib\
        submodule1\
        submodule2\
        MakeFile.am
    src\
        component1\
            comp1.cpp
            comp1.h
        component2\
        main.cpp
        Makefile.am
    .gitmodules
    .gitignore
    Makefile.am
    configure.ac
    autogen.sh
```


This the very a basic project structure and it can go to a very complex one, but just to give an idea let's bear with it. I won't give you details of these files but you can always checkout out my dedicated [repository](https://github.com/Vishwas1/cppBoilerplate) for c++ boilerplate. Files like, `Makefile.am`, `configure.ac` etc. are there for [Autotools](https://www.gnu.org/software/automake/manual/html_node/Autotools-Introduction.html) or [GNU Build Tools](https://en.wikipedia.org/wiki/GNU_Build_System). Basically, Autotools is a set of tools, `Autoconf`, `Automake`, `make utility` and `Libtools`,  used to build c++ project. 

#### Introduction to Autotools

- Autoconf generates `configure` script based on content of `configure.ac` file. 
- When `./configure` script runs, it generates `config.status` script.
- `Automake` generates `Makefile.in` based on content of `Makefile.am` file.
- `config.status` script takes input `Makefile.in` and convert into output file, `Makefile`, which are appropriate for that build environment
- Finally, `make` utility uses `Makefile` to generate executable program from source code.

```
configure.ac            Makefile.am              
    | [Autoconf]            | [Automake]
configure               Makefile.in
    | execute               |
configure.status <----------|
    | 
Makefile
    | [make]
(executable)

```

> `Configure` script is responsible for getting ready to build the software for your specific system. It makes sure all of the dependencies for the rest of the build and install process are available, and finds out whatever it needs to know to use those dependencies.

As you can see, as a developer, we only have to work on `configure.ac` and `Makefile.am` file, rest are autogenerated files by Autotools. Now, if you have to understand any c++ project, you should start looking from `configure.ac` file, present inside the root directory. In that file, you will find a [`m4`]() macro, `AC_CONFIG_FILES`, where we tell that what all Makefiles we have to build. 

`Makefile` directs `make` utility on how to compile and link a program. This is the file where we can find all dependencies, libraries, program files related to program(s). Usually this file present in the `src` directory of the project. Take a look at Bitcoin's [Makefile.am](https://github.com/bitcoin/bitcoin/blob/master/src/Makefile.am).You can go through [this](https://thoughtbot.com/blog/the-magic-behind-configure-make-make-install) link to understand Autotools better.

### Introduction to Bitcoin-core code base

The Bitcoin code base is a combination of essentially four programs `bitcoind`, `bitcoin-cli`, `bitcoin-tx` and `bitcoin-qt`. You can see these programs mentioned in [src/Makefile.am](https://github.com/bitcoin/bitcoin/blob/master/src/Makefile.am) file, search for `bin_PROGRAMS` macro.

```
if BUILD_BITCOIND
  bin_PROGRAMS += bitcoind
endif
if BUILD_BITCOIN_CLI
  bin_PROGRAMS += bitcoin-cli
endif
if BUILD_BITCOIN_TX
  bin_PROGRAMS += bitcoin-tx
endif
```

Bitcoin's developers have converted modules (consensus, server, cli, wallet, common, crypto etc.) into static libraries (.a files) so that they can be shared and used across all these programs. In order to understand how these files are managed, as I said, start looking into the *Makefile.am*, present inside src directory. For example:

- In the bitcoin's `src/Makefile.am` file, we have `bitcoind` right? Let's search this text in that file.  You will get `bitcoind_SOURCES` and `bitcoind_LDADD` macros for the source file of that program (basically the file where main() is defined) and linked static libraries (.a files) to `bitcoind` program respectively. 

- Now let's look for one of the linked libraries, say `libbitcoin_server`, you will find `libbitcoin_server_a_CPPFLAGS`, `libbitcoin_server_a_CXXFLAGS` and `libbitcoin_server_a_SOURCES` macros. In `libbitcoin_server_a_SOURCES` macro, you have all `.cpp` and `.h` files defined there.

### And we started!

Now that we have a pretty good idea about Bitcoin code base and we have also revised concepts of Autools, it is the time to start integrating *Bitcoin* into *Stellar*. First, we had to convert *Bitcoin* into one library. So that we can import this one library into the stellar-core code and use it.

**Identifying right program**: When I say Bitcoin, I only mean `bitcoind`, since this is the core bitcoin program which is required, others like wallets, cli etc are external components of code which is needed for users to interact with bitcoin network. Therefore, we disabled all other programs to be built by passing `--disable-*` flag with `./configure` script.

But as I said, whole Bitcoin's code is divided into multiple submodules and these submodules are converted into individual static libraries. We needed one library. So created one library out of all these libraries and called this static library, `libbitcoin_all.a`:

```
libbitcoin_all_a_LIBADD = \
   $(LIBBITCOIN_SERVER) \
   $(LIBBITCOIN_COMMON) \
   $(LIBUNIVALUE) \
   $(LIBBITCOIN_UTIL) \
   $(LIBBITCOIN_CONSENSUS) \
   $(LIBBITCOIN_CRYPTO) \
   $(LIBLEVELDB) \
   $(LIBLEVELDB_SSE42) \
   $(LIBMEMENV) \
   $(LIBSECP256K1)

```
and then we can link this library with stellar using `stellar_LDADD` macro, present in src/Makefile.am. See the example below:

 ```
 stellar_core_LDADD = 
    .
    ..
    $(libbitcoin_all_LIBS) 
```


If you want to learn how to create a simple static library in c++, you can refer to [this](https://github.com/Vishwas1/democpplib) repository of mine. In order to form this library, we made necessary changes to `src/Makefile.am` of *bitcoind* code base and built the project. Now that we have one static library file (`libbitcoin_all.a`), we linked that to `stellar-core` program via stellar's src/Makefile, called required methods of Bitcoin from Stellar code after importing the `.h` files and tried building the project (Stellar with linked Bitcoind library) as a whole but it didn't work :( 
    
We were getting  the following error:

```
(SET 1)
/home/vishwas/zagg-core/src/main/main.cpp:198: undefined reference to `SendRawTransactionZagg()'
main/stellar_core-main.o: In function `AppInit':
/home/vishwas/zagg-core/src/main/main.cpp:68: undefined reference to `gArgs'
/home/vishwas/zagg-core/src/main/main.cpp:68: undefined reference to `ArgsManager::ParseParameters(int, char const* const*, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >&)'
/home/vishwas/zagg-core/src/main/main.cpp:74: undefined reference to `gArgs'
...
..
.
(SET 2)
init.cpp:(.text+0x90c): undefined reference to `fsbridge::fopen(boost::filesystem::path const&, char const*)'
init.cpp:(.text+0xaf2): undefined reference to `GetPidFile()'
../lib/bitcoin/src/libbitcoin_server.a(libbitcoin_server_a-init.o): In function `HandleSIGHUP(int)':
init.cpp:(.text+0xd22): undefined reference to `g_logger'
...
..
.
```
Full error is present in [this](https://gist.github.com/Vishwas1/6dbc5a9704d173061931666784511e8a) gist.

### Debugging the issue

We all know, whenever we get some issue, we should understand the bug first and try to find out the root cause before we sit to fix it. Let's try to do that without any further delay.

- Most of these errors are `undefined reference` error. As you can see, When I am trying to call bitcoin's method `SendRawTransactionZagg()` I am getting this error. This error means that compiler is able to find the declaration of this method (since we included the header files) but unable to find out the definition of it. A similar error was happening with other methods too. 

- If you noticed the next set of errors, again `undefined reference`, but this time they are related to `boost` libraries, a dependency library of bitcoin, which means that I am not linking `boost-filesystem` library with the Stellar's build. 

- It was clear from these errors that this is linking problem, which is the root cause. From here, we got two directions to move into:
    1. Check if there is any problem with `libbitcoin_all.a`?
    2. We need to find out all dependencies library of bitcoin and link them with stellar. This we assumed by looking into `boost` error.

#### Step 1

The first thing which came into our mind is, the method which we were trying to call, `SendRawTransactionZagg()`, is actually present in the `libbitcoin_all.a` library or not. As I said, this static library is nothing but archival of object files, so ideally that method should exist in that library.

We needed to uncompress it, but before we even do that, first let's check out weather that library is even present or was successfully created? 

```
cd bitcoin
find . -name '*.a'
```

Results:

```
./src/.libs/libbitcoinconsensus.a
./src/libbitcoin_util.a
./src/univalue/.libs/libunivalue.a
./src/libbitcoin_wallet.a
./src/libbitcoin_consensus.a
./src/libbitcoin_common.a
./src/libbitcoin_all.a
./src/crypto/libbitcoin_crypto_shani.a
...
..
.

```

As you can see we have that library present. Now let's use the tool `nm` with `grep` to uncompress it and see if our method exists in it:

```
cd src
nm -A libbitcoin_all.a | grep "Zagg"

```

> This command did not result in anything and hence it was clear that there was some fault with the `all` library which we had created. 

#### Step 2

Next, we tried to figure out all the individual libraries which `bitcoind` was using along with their dependencies, and used them with stellar.

**Finding all individual libraries:**

When we run the `make -n` command and try to build Bitcoin source code, notice that it essentially runs a `gcc` to compile the source code for `bitcoind`. We can also notice that `gcc` runs with linked libraries and dependencies. We copied them from the console into a notepad and carefully looked into it.  

Take a look at console of `make -n` command:

```
.
..
...
bitcoind;/bin/bash ../libtool --silent --tag=CXX --preserve-dup-deps  --mode=link g++ -std=c++11  -Wstack-protector -fstack-protector-all      -fPIE --param ggc-min-expand=1 --param ggc-min-heapsize=32768  -pthread  -Wl,-z,relro -Wl,-z,now -pie      -o bitcoind bitcoind-bitcoind.o  libbitcoin_server.a  libbitcoin_server.a libbitcoin_common.a univalue/libunivalue.la libbitcoin_util.a  libbitcoin_consensus.a crypto/libbitcoin_crypto_base.a crypto/libbitcoin_crypto_sse41.a crypto/libbitcoin_crypto_avx2.a crypto/libbitcoin_crypto_shani.a leveldb/libleveldb.a leveldb/libleveldb_sse42.a leveldb/libmemenv.a secp256k1/libsecp256k1.la libbitcoin_all.a -L/usr/lib/x86_64-linux-gnu -lboost_system -lboost_filesystem -lboost_thread -lboost_chrono  -lcrypto  -levent_pthreads -levent -levent  
...
..
.

```
- As you can see what all libraries are linked, like `libbitcoin_server.a`, `libbitcoin_common.a`, `libbitcoin_consensus.a` etc. and also notice some dependencies like `boost`, `univalue`, `crypto`, `secp256k1` etc. Which means that we need to include all these libraries with stellar as well.

**Using individual libraries and dependencies in Stellar**

We removed `all.a` library link from stellar first and then we added all the above libraries and dependencies. This is done in the `src/Makefile.am` file of the stellar code base. See the code snippet below:

```
stellar_core_LDADD = \
        .
        ..
        ...
    $(libbitcoin_server_LIBS) \
    $(libbitcoin_consensus_LIBS) \
    $(libbitcoin_common_LIBS) \
        $(libbitcoin_util_LIBS) \
    $(libbitcoin_wallet_a) \
    $(libbitcoin_univalue_LIBS) \
    $(libbitcoin_crypto_base_LIBS) \
    $(libbitcoin_crypto_sse41_LIBS) \
    $(libbitcoin_crypto_avx2_LIBS) \
    $(libbitcoin_crypto_shani_LIBS) \
    $(libbitcoin_leveldb_LIBS) \
    $(libbitcoin_leveldb_sse42_LIBS) \
    $(libbitcoin_memenv_LIBS) \
    $(libbitcoin_secp256k1_LIBS) \
    -lboost_system \
    -lboost_thread \
    -lboost_chrono \
    -lboost_filesystem \
    -lcrypto \
    -levent_pthreads \
    -levent 
```

There could be a better way to add those dependencies libraries, but let's overlook the decoration in order to fix the issue. We build the stellar project once again and, guess what, this time it worked! We could successfully be able to build the stellar code with Bitcoin as the library and was able to call Bitcoin's methods.

## Conclusion 

One thing I forgot to tell you is we used a [demo project](https://github.com/Vishwas1/cppBoilerplate), rather than Stellar directly, to fix the bug as we had more control over Makefiles and other source codes in there. I think it's always good to use a small project since for a beginner, the stellar project might seems huge and we will have less control over the code bases. But again it depends on our choice, how we want to proceed. I believe breaking a large problem in smaller chunks is always helpful. I hope this blog has given you enough insights on how to build and integrate c++ projects and how to approach the problem if you got one while doing so. Hope to see you next time. Cheers!

> Hey wait a minute, what went wrong with your approach of builing single library (`libbitcoin_all.a`)? You did not talk about it?

This is the question you possible have in your mind, right? I will leave this question for the reader to think and let me know your answers in the comment section below. Thanks!