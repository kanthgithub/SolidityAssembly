# SolidityAssembly

- Solidity Inline Assembly references, best practices, Dos' & Donts' are listed here

## Read The Docs:

- https://solidity.readthedocs.io/en/v0.5.3/assembly.html#example
- https://solidity.readthedocs.io/en/v0.5.3/assembly.html


## Reddit:

- https://www.reddit.com/r/ethdev/comments/80y3oc/interviewing_for_intermediateexperienced_solidity/

## understanding Instructions:

- mload
  - https://ethereum.stackexchange.com/questions/9603/understanding-mload-assembly-function


## Datatype conversions:

- Address to Bytes

  - https://ethereum.stackexchange.com/questions/884/how-to-convert-an-address-to-bytes-in-solidity
  
  ```js
  function toBytes(address a) constant returns (bytes b){
   assembly {
        let m := mload(0x40)
        mstore(add(m, 20), xor(0x140000000000000000000000000000000000000000, a))
        mstore(0x40, add(m, 52))
        b := m
   }
}
```

```
This is designed to convert an address to a dynamic bytes type.
Addresses are 20 bytes long, and occupy the right-most 20 bytes of a 32-byte word:
```

```js
0x000000000000000000000000aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
```

```
The bytes type is two (or more) words: the number of bytes followed by the data.
The number of bytes in an address is 20 = 0x14, so it need to look like this in memory.
```
```js
0x0000000000000000000000000000000000000000000000000000000000000014
0xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa000000000000000000000000
```
```
The final thing to know is that Solidity stores the current top of memory in the location 0x40. 
So m is the top of memory, and is also where we are going to store the bytes version of a, namely b.

What we want to end up with is:
```

```js
m+0  : 0x0000000000000000000000000000000000000000000000000000000000000014
m+32 : 0xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa000000000000000000000000
```
```
So here are the steps:
```
1. let m := mload(0x40) - sets m to current top of memory
2. xor(0x140000000000000000000000000000000000000000, a) - puts 0x14 in front of the 20 bytes of address data and returns a 32-byte word padded with leading zeroes: 0x000000000000000000000014aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
3. add(m, 20) - when the result of the xor is written, it is shifted 20 bytes relative to the top of memory. 
This is both clever and dangerous; 
there's no guarantee that the memory at m was empty, and we are not competely over-writing it. Anyway, this puts the 0x14 where we want it at the end of the word and followed by the aaaas overflowing into the next word.
4. mstore(0x40, add(m, 52)) - finally we update the top of memory pointer;
we've added 52 bytes in total (32 + 20).
This would be better as add(m, 64) in my view in case anything elsewhere relies on memory being word-aligned,
but I may be over-cautious.
5. b := m - finally return the (pointer to the) result.

```
In short, this is very smart,
but I would definitely zero-out the two words at top of memory
before doing this to avoid any possible issues.
```
