# The most expensive Solidity function

Have you ever run into a function that just won’t fit into an Ethereum block because it costs too much gas? This is the story of how we tackled that very challenge!

## The problem

It all started when we built the Rarimo protocol. If you aren’t familiar with Rarimo, check out [this post](https://medium.com/@denys.riabtsev/national-passports-verification-with-zkp-part-1-b653f5e5c8d8) for more details. Rarimo lets users prove that they have a passport from a certain country without revealing any personal information.

The challenge was that there are no strict rules for passport cryptography. Each country can use its own method. Some use RSA or ECDSA signatures combined with various hash functions like SHA-256, SHA-384, SHA-512, and more. OpenPassport even has a [map](https://map.openpassport.app/) showing the algorithms used in passports worldwide. For example, Peru uses only ECDSA signatures over 384-bit curves, while Brazil uses 512-bit curves.

At first, it seemed simple to build a solution that supports all these algorithms since the only real difference was the size of the parameters used. But here’s the problem: the Ethereum Virtual Machine (EVM) only supports 256-bit unsigned integers! So, we had three choices:

1. **Use Zero-Knowledge Proofs:** Verify the signature with ZK-proofs. The downside was that users would need to download a huge "zkey", which would make the UX worse.
2. **Make a Precompile:** Create a precompile for signature verification. However, this would require a chain fork, which is not easy.
3. **Do Everything On-Chain:** Implement all the logic on-chain. Although this method wouldn’t work on Mainnet because of high gas costs, it was a good fit for us since we use our own Layer 2 solution with cheap gas. Our only challenge was to keep the function’s gas usage below 30 million per block.

## BigNumber library

Finally, we decided to go with the third approach: implementing ECDSA for both 384-bit and 512-bit curves directly on-chain. But we ran into another problem. It turned out that there isn’t an efficient "bigint" library available. Not because no one can write one, but because it’s nearly impossible to build one in a general way that works with any integer size while keeping the gas cost as low as possible.

For example, we tried to integrate our code with the [BigNumber library](https://github.com/firoorg/solidity-BigNumber). This library is really cool and gave us ideas for future improvements, but our verification algorithm still used too much gas because of two issues:

1. **Abstraction costs:** The library is written in a general way to handle any integer size. However, as is often the case, any abstractions make your code less optimized.
2. **Memory Expansion:** The EVM doesn't have a heap or allocators for efficient memory management. Once memory is allocated, it is never released, which also results in [memory expansion](https://www.evm.codes/about#memoryexpansion) overhead.

We spent a lot of time looking for a solution. We tried to optimize the library and even emulated a heap and allocators directly in Solidity to cut down the overhead. We even found some bugs and  [contributed](https://github.com/firoorg/solidity-BigNumber/pull/18)  to the original BigNumber library.

But nothing worked. Our implementation still required more than 400 billion gas for execution plus gas needed for memory expansion overhead. We couldn’t even run the whole algorithm in our testing environment because the machine ran out of memory.

## Uint512

Just when we were about to give up and try one of the other options, a sudden idea struck us. We realized that we didn't need a general BigNumber library at all - we only needed 512-bit integers! With everything we had learned from working with the BigNumber library, we believed it's possible to implement our own highly optimized library for 512-bit unsigned integers.

### Memory layout

How to implement a uint512 library most efficiently? The first challenge was deciding how to store a uint512 in memory. Since the integer size is fixed, we can always use 2 memory slots - each 256 bits - to hold a uint512. But what data type should be used? Structs? Bytes? Neither, because both require an extra memory slot. For example, a bytes type needs a slot for its length, and a struct needs a slot for a data pointer.

That's why we decided to use raw pointers directly on the stack. They are automatically removed when the function finishes, so only 2 memory slots are used for each uint512. Using less memory reduces the overhead from memory expansion, which is crucial for functions that perform a lot of work.

The diagram below shows the different storage structures:

![diagram](./layout.png)

Even more, starting with Solidity 0.8.8, [user-defined value types](https://soliditylang.org/blog/2021/09/27/user-defined-value-types/) are added, which makes the user experience much smoother. You no longer need to use the `memory` keyword, and you might even forget that the uint512 type isn’t built-in!

Here's an example:

```
// U512.sol
type uint512 is uint256; // synonym for pointer

// Contract.sol
uint512 a_ = U512.fromUint256(3);
uint512 b_ = U512.fromUint256(6);
uint512 m_ = U512.fromUint256(5);
uint512 r_ = U512.modadd(a_, b_, m_);
U512.eq(r_, U512.fromUint256(4)); // true
U512.toBytes(r_); // "0x00..04" - 64 bytes
```

This setup allows you to work with uint512 almost as if it were a native type.

### Basic operations

Now that we understand how to store our uint512 values, it should be clear how to implement basic operations. The idea is simple: perform the same operation on the matching 256-bit words of the two numbers, and then combine the results correctly.

For example, when adding two uint512 values, you add the lower words first and then the upper words, making sure to handle any overflow from the lower word addition. The diagram below shows how you can implement the add function.

![diagram](./add.png)

### Modexp magic 

But what about mod operations? Is it necessary to write a mod function in Solidity? That would be tough. The answer is no. We can use the [modexp precompile](https://www.evm.codes/precompiled?fork=cancun#0x05) available in the EVM! This built-in feature lets you make a static call to a predefined contract that calculates `b^e mod m` for any size of input.

This saves a lot of gas because, instead of coding the mod algorithm on-chain, the Ethereum node does the heavy work for you at a low cost. With this tool, it's easy to implement functions like modexp directly, as well as mod (by setting `e = 1`), and modadd along with modmul by first computing the result and then taking it modulo.

However, you must use the modexp precompile carefully. For example, we once accidentally passed the exponent size as 64 bytes, which ended up costing 10 times more gas, even though the exponent was just 1!

### Multiplication

Multiplication is the most expensive operation so far because its complexity grows quadratically, which leads to many overflows that are hard to manage. The BigNumber library uses a clever trick: instead of computing `a * b` directly, it calculates `((a + b)^2 - (a - b)^2) / 4`. This equation is mathematically equivalent to `a * b` and uses basic operations like add, sub, mul, and shr (instead of division by 4). Although it's an awesome idea, it requires several modexp calls along with many intermediate steps and extra memory allocations, making it inefficient.

It turned out that for small integers, a column-by-column multiplication (as shown in the diagram below) is more efficient:

![diagram](./mul.png)

But when multiplying 512-bit numbers stored as two 256-bit words, we must multiply the corresponding words and handle the carries accurately. This is not as straightforward as it seems. Multiplying two 256-bit numbers can generate a huge carry, and we only have Solidity’s built-in uint256 arithmetic to work with.

Fortunately, this is a known problem that can be solved efficiently using the EVM’s [modmul opcode](https://www.evm.codes/?fork=cancun#09). Remco Bloemen explains this approach in his article [Mathemagic finale: muldiv](https://xn--2-umb.com/21/muldiv/). We don’t need the division part of his method, but the small piece of code that calculates the most significant 256 bits of the product - "huge carry". It's only three lines of code, but it packs a lot of math:

```
// 512-bit multiply [prod1 prod0] = a * b
// Compute the product mod 2**256 and mod 2**256 - 1
// then use the Chinese Remainder Theorem to reconstruct
// the 512 bit result. The result is stored in two 256
// variables such that product = prod1 * 2**256 + prod0
uint256 prod0; // Least significant 256 bits of the product
uint256 prod1; // Most significant 256 bits of the product
assembly {
	let mm := mulmod(a, b, not(0))
	prod0 := mul(a, b)
	prod1 := sub(sub(mm, prod0), lt(mm, prod0))
}
```

This approach brings the gas cost of the multiplication function very close to that of the addition function: 353 gas (mul) vs 256 gas (add). Similarly, since only one modexp call is needed, the modulo versions become much more efficient 1176 gas (modmul) 760 gas (modadd).

### Reuse Memory

While our current library works well from an algorithmic perspective, we still haven't solved the memory expansion issue completely. One major problem with the BigNumber library was that it allocated memory for every modexp precompile call. Our ECDSA algorithm, however, makes thousands of such calls, and each call uses 384 bytes of memory! This extra memory quickly adds up - and the EVM never releases it once allocated!

But here's the good news: once a mod operation is complete, that memory is no longer needed. So why not help the EVM reuse that memory? Instead of allocating memory every time, we can allocate it once at the beginning of a heavy function and then pass the pointer to it to each mod operation. This way, no extra memory is allocated internally. For lighter functions, we even provide a version that allocates memory internally if you don't plan on doing many operations.

Another neat feature we added is assignment operations. When working with complex formulas, you don't always need to create new integer objects; you can simply assign new values to existing ones. This helps control memory usage and reduces overhead.

The code example below shows how it works: first, you create a call pointer once, which allocates memory for internal purposes. Then, you pass this pointer into each mod function so that no additional memory is allocated. You also use assignment operations so that no new integers are created. This approach makes it easier to see and control how much memory is used.

```  
call512 call_ = U512.initCall();  
uint512 a_ = U512.fromUint256(3);  
uint512 b_ = U512.fromUint256(6);  
uint512 m_ = U512.fromUint256(5);  
uint512 r_ = U512.modadd(call_, a_, b_, m_); // 4  
U512.modmulAssign(call_, r_, a_, m_); // assigns (r_ * a_) % m_ to r_
r_.eq(U512.fromUint256(2)); // true
r_.toBytes(); // "0x00..02"  
```

## Gas costs

By applying all the optimizations, we achieved some impressive results:

| uint512 operation | Avg gas  |
| ----------------- | -------- |
| add               | 269 gas  |
| sub               | 278 gas  |
| mul               | 353 gas  |
| mod               | 682 gas  |
| modinv            | 6083 gas |
| modadd            | 780 gas  |
| redadd            | 590 gas  |
| modmul            | 1176 gas |
| modsub            | 1017 gas |
| redsub            | 533 gas  |
| modexp            | 5981 gas |
| modexpU256        | 692 gas  |
| moddiv            | 7092 gas |
| and               | 251 gas  |
| or                | 251 gas  |
| xor               | 251 gas  |
| not               | 216 gas  |
| shl               | 272 gas  |
| shr               | 272 gas  |

For the ECDSA algorithm, the 384-bit curves consume 8.9 million gas and the 512-bit curves consume 13.6 million gas - a **30,000x improvement** compared to the initial version!