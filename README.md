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
0x000000000000000000000000aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa

24 characters (prefix):
 000000000000000000000000 

40 characters (address):
 aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa

2 characters  =  1 Byte => 24 Characters = 12 Bytes
2 characters  =  1 Byte => 40 Characters = 20 Bytes

total: 64 characters => 32 Bytes
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
### solidity-to-inline-assembly

```js
contract MyContract {
    address public beneficiary;

    function MyContract(address _beneficiary) public {
         beneficiary = _beneficiary;
    }

    function () public payable {
         beneficiary.transfer(msg.value);
    }
}
```

```js
function () payable {
    bytes4 sig = bytes4(keccak256("()")); // function signature

    assembly {
        let x := mload(0x40) // get empty storage location
        mstore ( x, sig ) // 4 bytes - place signature in empty storage

        let ret := call (gas, 
            beneficiary,
            msg.value, 
            x, // input
            0x04, // input size = 4 bytes
            x, // output stored at input location, save space
            0x0 // output size = 0 bytes
        )

        mstore(0x40, add(x,0x20)) // update free memory pointer
    }
}
```


https://ethereum.stackexchange.com/questions/47462/solidity-to-inline-assembly

### calling-the-function-of-another-contract-in-solidity

- https://medium.com/@blockchain101/calling-the-function-of-another-contract-in-solidity-f9edfa921f4c

```js
pragma solidity ^0.4.18;
contract Deployed {
    
    function setA(uint) public returns (uint) {}
    
    function a() public pure returns (uint) {}
    
}
contract Existing  {
    
    Deployed dc;
    
    function Existing(address _t) public {
        dc = Deployed(_t);
    }
 
    function getA() public view returns (uint result) {
        return dc.a();
    }
    
    function setA(uint _val) public returns (uint result) {
        dc.setA(_val);
        return _val;
    }
    
}

pragma solidity ^0.4.18;
contract ExistingWithoutABI  {
    
    address dc;
    
    function ExistingWithoutABI(address _t) public {
        dc = _t;
    }
    
    function setA_ASM(uint _val) public returns (uint answer) {
        
        bytes4 sig = bytes4(keccak256("setA(uint256)"));
        assembly {
            // move pointer to free memory spot
            let ptr := mload(0x40)
            // put function sig at memory spot
            mstore(ptr,sig)
            // append argument after function sig
            mstore(add(ptr,0x04), _val)

            let result := call(
              15000, // gas limit
              sload(dc_slot),  // to addr. append var to _slot to access storage variable
              0, // not transfer any ether
              ptr, // Inputs are stored at location ptr
              0x24, // Inputs are 36 bytes long
              ptr,  //Store output over input
              0x20) //Outputs are 32 bytes long
            
            if eq(result, 0) {
                revert(0, 0)
            }
            
            answer := mload(ptr) // Assign output to answer var
            mstore(0x40,add(ptr,0x24)) // Set storage pointer to new space
        }
    }
}
```

## Proxy-Delegate using assembly:

- https://fravoll.github.io/solidity-patterns/proxy_delegate.html

```
This generic example of a Proxy contract is inspired by this post and stores the current version of the delegate in its own storage. Because the design of the Delegate contract can take many forms, there is no explicit example given.

// This code has not been professionally audited, therefore I cannot make any promises about
// safety or correctness. Use at own risk.
```

```js
contract Proxy {

    address delegate;
    address owner = msg.sender;

    function upgradeDelegate(address newDelegateAddress) public {
        require(msg.sender == owner);
        delegate = newDelegateAddress;
    }

    function() external payable {
        assembly {
            let _target := sload(0)
            calldatacopy(0x0, 0x0, calldatasize)
            let result := delegatecall(gas, _target, 0x0, calldatasize, 0x0, 0)
            returndatacopy(0x0, 0x0, returndatasize)
            switch result case 0 {revert(0, 0)} default {return (0, returndatasize)}
        }
    }
}
```

```
The address variables in line 3 and 4 store the address of the delegate and the owner, respectively. The upgradeDelegate(..) function is the mechanism that allows a new version of the delegate being used, without the caller of the proxy having to worry about it. An authorized entity, in this case the owner (checked with a simple form of the Access Restriction pattern in line 7) is able to provide the address of a new delegate version, which replaces the old one (line 8).
```
```
The actual forwarding functionality is implemented in the function starting from line 11. The function does not have a name and is therefore the fallback function, which is being called for every unknown function identifier. Therefore, every function call to the proxy (besides the ones to upgradeDelegate(..)) will trigger the fallback function and execute the following inline assembly code: Line 13 loads the first variable in storage, in this case the address of the delegate, and stores it in the memory variable _target. Line 14 copies the function signature and any parameters into memory. In line 15 the delegatecall to the _target address is made, including the function data that has been stored in memory. A boolean containing the execution outcome is returned and stored in the result variable. Line 16 copies the actual return value into memory. The switch in line 17 checks whether the execution outcome was negative, in which case any state changes are reverted, or positive, in which case the result is returned to the caller of the proxy.
```

<b>Consequences</b>

```
There are several implications that should be considered when using the Proxy Delegate pattern for achieving upgradeability. With its implementation, complexity is increased drastically and especially developers new to smart contract development with Solidity, might find it difficult to understand the concepts of delegatecalls and inline assembly. This increases the chance of introducing bugs or other unintended behavior. Another point are the limitations on storage changes: fields cannot be deleted nor rearranged. While this is not an insurmountable problem, it is important to be aware of, in order to not accidentally break contract storage. An important negative consequence from a social perspecive is the potential loss in trust from users. With upgradeable contracts, immutability as one of the key benefits of blockchains, can be avoided. Users have to trust the responsible entities to not introduce any unwanted functionality with one of their upgrades. A solution to this caveat could be strategies that only allow for partial upgrades. Core features could be non-upgradeable, while other, less essential, features are implemented with the option for upgrades. If this approach is not applicable, a trust loss could also be mitigated by introducing a test period, during which upgrades can be carried out. After the expiration of the test period, the contract cannot be changed any longer.

Besides these negative consequences, the Proxy Delegate pattern is an efficient way to separate the upgrading mechanism from contract design. It allows for upgradeability, without breaking any dependencies.
```

<b>Known Uses</b>

```
Implementations of the Proxy Delegate pattern are more likely to be found in bigger DApps, containing a large number of contracts. One example for this is Augur, a prediction market that lets users bet on the outcome of future events. Another example is the EtherRouter contract of Colony, which is a platform for creating decentralized organizations. In both cases, Augur and Colony, the address of the upgradeable contract is not stored in the proxy itself, but in some kind of address resolver.
```
