# `_MakeBlendField` Overview

> Warning:  
> Many parts of `zombs_wasm.wasm` are heavily obfuscated and remain a mystery, so this part of the documentation is (largely) incomplete and only provides an outline of the algorithm. You'll have to figure out the actual code and its behavior during operation by yourself.

> Note:  
> The version of `zombs_wasm.wasm` this overview analyzes is from late 2024, which may not be 100% consistent with the current version.

Zombs.io employs an original method to validate players via [Proof of Work](https://en.wikipedia.org/wiki/Proof_of_work) (PoW) challenges that the client has to solve in order to enter a server and maintain a connection. It is a measure to occupy RAM and CPU resources of the client and, therefore, increase multiboxing costs.  
The main algorithm of the PoW process is written in C++ and compiled into `zombs_wasm.wasm` with Emscripten. Reversing is very difficult due to the low-level nature of the [WebAssembly](https://en.wikipedia.org/wiki/WebAssembly) (WASM) language it uses.  
The core function `zombs_wasm.wasm` exposes is called `_MakeBlendField`. From now on, for convenience, `_MakeBlendField` will simply be referred to as MBF.

![mbf](./mbf.jpg)

When the client receives an opcode 5(`PRE_ENTER_WORLD`)/10(`BLEND`) packet, it has to decode the packet with `game.network.codec` and send back the required data in opcode 4(`ENTER_WORLD`)/10 and 6(`ENTER_WORLD_2`, opcode 5 only) packets. If the client fails to submit data that passes validation within the given time, the client will not be able to enter the server/disconnects from the server.

## Challenge

The opcode 5/10 packet the server sends contains a 132-byte message, which can be represented by a `Uint8Array`, such as this one shown below:
```
[139,165,21,124,237,21,234,88,128,192,29,175,28,15,118,27,23,73,18,225,109,252,205,195,45,213,46,51,246,67,218,224,19,28,197,74,97,235,236,131,205,163,61,41,148,171,164,228,104,163,187,238,91,157,27,116,176,245,245,185,89,203,170,255,85,195,243,88,38,230,46,252,7,25,12,191,170,12,159,145,147,193,3,135,145,178,215,57,46,252,35,65,226,130,39,146,41,4,18,27,4,34,201,232,195,133,20,68,19,42,42,74,168,255,4,181,85,189,86,121,144,210,90,82,224,238,5,8,107,176,123,245]
```
This message can further be split into 3 parts: a 32-bit integer (4 bytes) and two 64-byte segments. The first byte of the first 64-byte segment is used to determine the logic branch used for the challenge (`byte % <number of branches>`, 0-indexed), where each branch adopts a unique set of constants in various parts of the algorithm.  
After deobfuscating the integer with a specific formula, two important parameters of the challenge can be inferred: the last byte of the integer (`integer & 255`) is the difficulty of the challenge, which determines the rarity of the solution and the time it takes to find such solution; the first 2 bytes (`integer >>> 16`) is the size of the [PRNG](https://en.wikipedia.org/wiki/Pseudorandom_number_generator)-generated pool (in megabytes), which will later be filled using [Xorshift](https://en.wikipedia.org/wiki/Xorshift)-generated numbers and be used to retrieve a random byte value in later steps of the algorithm while actually occupying memory with its sheer size. For comparison, the difficulty is generally 17 for opcode 5 and 9-13 for opcode 10, while the pool size is usually 128 megabytes.

> Note:  
> As of early 2025, after a few hours into the connection, opcode 10 packets may reach difficulty 19 or higher, while pool size is increased to 512 megabytes (theoretical limit is near 2048 megabytes), significantly increasing the cost of maintaining longer connections.

`blend` and `field`, which are both important components in the MBF process, can be obtained from the two 64-byte segments. They will be further elaborated in the upcoming sections.

## Hash Functions

The hash functions of MBF's algorithm are derived from the [SHA-1](https://en.wikipedia.org/wiki/SHA-1) hash algorithm, which produces a 20-byte (5 32-bit integers) hash value (digest) from any given input. The hash value changes drastically even with only a slight difference in the input, making creating a solution with the given result without guessing nearly impossible.  
These hash functions resemble that of the original SHA-1 function, but with different sets of constants and partially modified logic for each branch.

## `blend`, `field`, `blend_field` and `mask`

`blend` and `field` (unofficially named after `_MakeBlendField`) are two important components in the MBF process. `blend` is a hash value obtained via reversing the first 64-byte segment and hashing the reversed segment, while `field` is the second 64-byte segment itself. They together produce `blend_field` and `mask`, two crucial elements that participate in the actual PoW loop.
`blend_field` is a 64-byte array produced from applying arithmetic to each byte of `field` with certain bytes in `blend` and a set of magic numbers (153, 111, 72 and 221), and eventually reversing the processed array. It is one of the segments that make up `payload` in the PoW loop, which will be explained later. The first 16 bytes of this array are sent by the client in the opcode 6 packet within a 100-byte [ByteBuffer](https://github.com/protobufjs/bytebuffer.js/) default buffer, like this one:
```
[161,229,245,1,88,44,180,171,255,85,253,126,113,49,212,239,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]
```
`mask` is another hash value that is acquired from appending a 32-bit integer obtained from applying a specific conversion formula to `chall_count` (a 32-bit integer representing the number of PoW challenges the websocket has solved, including the current one) at the end of the array, and then hashing the resulting 24-byte array. It is used to "mask" `random_buffer` to create the PoW result sent by the client in the opcode 5/10 packet, which will also be explained later.

## PoW Loop

In order to solve the PoW challenge, the client has to repeatedly change and hash the `payload` until one whose hash value matches the given criteria is found. The payload is an array that can be split into 3 parts: the server's IPv4 in the form of a string (without the null terminator `\0`), `blend_field` and a 64-byte `random_buffer`.  
The PRNG used in the loop is the standard C++ [MT19937](https://en.wikipedia.org/wiki/Mersenne_Twister) generator paired with [`std::uniform_int_distribution`](https://cplusplus.com/reference/random/uniform_int_distribution/). The seed of the PRNG is obtained from calling `performance.now()` in JavaScript through Emscripten features.

The steps are described below (every array is a 0-indexed byte array):

1. Randomly choose 2 bytes in the random pool and swap them.
2. Pick another byte in the pool randomly and insert it into a random position of `random_buffer`.
3. Set `random_buffer[10/11/12/13]` to `random_buffer[0/40/51/4] + random_buffer[23/25/50/45] + uid[0/1/2/3]` (`uid` is the player's uid expressed in a 32-bit integer) and `random_buffer[14/15/16/17]` to `random_buffer[41/22/35/39] ^ blend_field[0/1/2/3]`.
4. Hash the payload to generate a digest and validate it to confirm if it matches the criteria. The criteria are given by the following statement:
> If the difficulty is `N`, then for every integer `i` in `[0, N)`, there is `(digest[i >> 3] << (i & 7)) & 128 == 0`.
5. If the digest meets the criteria, "mask" `random_buffer` by setting `random_buffer[i]` to `random_buffer[i] ^ mask[i % 20]` for every integer `i` in `[0, 64)`; otherwise repeat step 1. The masked array is the result sent by the client in the opcode 5/10 packet, such as this one shown below:
```
[212,153,0,149,244,56,73,26,178,35,28,213,168,24,168,171,98,149,44,184,17,8,186,240,133,97,240,60,19,240,248,28,116,101,177,105,164,25,88,23,107,205,198,51,106,126,26,249,169,143,150,119,251,252,183,90,184,78,110,34,190,129,82,87]
```

## Bugs

When the zombs.io website client tries to connect to a server whose IPv4 address is shorter than one that the client tried before, it throws the error shown below because the `_free()` function isn't exported in `zombs_wasm.wasm`. Fix this by adding func $6 in `zombs_wasm.wasm` (which is the `free()` function in C++) to the exports. For custom clients instantiating `zombs_wasm.wasm` each time they connect, this can be ignored.

![error_1](./error_1.png)

## Trivia

- When the client tries to decode an opcode 5/10 packet using an outdated `zombs_wasm.wasm` file, the client may freeze or throw an error because using the wrong deobfuscation formula might yield an unreasonably high difficulty or random pool size, and thus trigger an indefinite loop or a memory overflow.  
- The term "0mb Wasm", often used by the zombs.io community, refers to custom MBF algorithms that significantly reduce RAM usage by limiting the size of the random pool. This can be achieved through either analyzing the logic and rebuilding the whole algorithm or directly manipulating `zombs_wasm.wasm` by editing its text format (`.wat`). However, the mathematical nature of PoW challenges causes a significant reduction in CPU usage and computing time to be highly unlikely. (Otherwise ez cryptocurrency mining :P)

## Credits

Article written by AstralCat   
Preliminary research and base algorithm by ABCxff  
In-depth reversal and algorithm revision by AstralCat
