# Function Selector and Argument Encoding

> For details, see the [official documentation](https://docs.soliditylang.org/en/v0.8.1/abi-spec.html#)

> Referenced from my blog [Function Selector and Argument Encoding](https://hitcxy.com/2021/argument-encoding/)

In the Ethereum ecosystem, ABI (Application Binary Interface) is a standard way to interact with contracts from outside the blockchain and for contract-to-contract interactions. Data is encoded according to its type as described in this specification.

## Function Selector

### Principle
The first 4 bytes of the Keccak (SHA-3) hash of a function signature specify the function to be called, in the form of bytes4(keccak256('balanceOf(address)')) == 0x70a08231. Here, 0x70a08231 is the Function Selector for balanceOf(address).

- The basic prototype is the function name plus the parameter type list enclosed in parentheses, with parameter types separated by a single comma and no spaces.
- For uint types, they must be converted to uint256 for calculation. For example, ownerOf(uint256) has Function Selector = bytes4(keccak256('ownerOf(uint256)')) == 0x6352211e.
- If function parameters contain structs, the struct is expanded into individual parameters, but these parameters are enclosed in `()`. See the examples below for details.

### Examples

```solidity
pragma solidity >=0.4.16 <0.9.0;
pragma experimental ABIEncoderV2;

contract Demo {
    struct Test {
        string name;
        string policies;
        uint num;
    }
    
    uint public x;
    function test1(bytes3) public {x = 1;}
    function test2(bytes3[2] memory) public  { x = 1; }
    function test3(uint32 x, bool y) public  { x = 1; }
    function test4(uint, uint32[] memory, bytes10, bytes memory) public { x = 1; }
    function test5(uint, Test memory test) public { x = 1; }
    function test6(uint, Test[] memory tests) public { x = 1; }
    function test7(uint[][] memory,string[] memory) public { x = 1; }
}

/* Function Selectors
{
    "0d2032f1": "test1(bytes3)",
    "2b231dad": "test2(bytes3[2])",
    "92e92919": "test3(uint32,bool)",
    "4d189ce2": "test4(uint256,uint32[],bytes10,bytes)",
    "4ca373dc": "test5(uint256,(string,string,uint256))",
    "ccc5bdd2": "test6(uint256,(string,string,uint256)[])",
    "cc80bc65": "test7(uint256[][],string[])",
    "0c55699c": "x()"
}
*/
```

## Function Selector and Argument Encoding

### Principle

* For dynamic types such as dynamic arrays, structs, and variable-length bytes, the encoding stores their `offset`, `length`, and `data`.
    - First, store parameters in order: for fixed-length data types, store their `data` directly; for variable-length data types, store their `offset` first.
    - Traverse variable-length data in order: first store the `offset`. For the first variable-length data, store its `offset = 0x20 * number` (`number` is the number of function parameters). For the next variable-length data, its `offset = offset_of_prev + 0x20 + 0x20 * number` (the first `0x20` is the size occupied by storing the length of the previous variable-length data, and `number` is the number of elements in the previous variable-length data).
    - Traverse variable-length data in order: after storing the `offset`, iterate over each variable-length data and store their `length` and `data` respectively.
    - (`ps:` For types like structs, when storing, treat the struct's internal elements as parameters of a new function. In this case, for the first variable-length data in the struct, its `offset = 0x20 * num`, where `num` is the number of struct elements.)

### Examples

For the 7 functions in the contract example above, the final encoding of each function call is as follows:

- test1("0x112233")

```
0x0d2032f1                                                             // function selector
0 - 0x1122330000000000000000000000000000000000000000000000000000000000 // data of first parameter
```

- test2(["0x112233","0x445566"])

```
0x2b231dad                                                             // function selector
0 - 0x1122330000000000000000000000000000000000000000000000000000000000 // first data of first parameter
1 - 0x4455660000000000000000000000000000000000000000000000000000000000 // second data of first parameter
```

- test3(0x123,1)

```
0x92e92919                                                             // function selector
0 - 0x0000000000000000000000000000000000000000000000000000000000000123 // data of first parameter
1 - 0x0000000000000000000000000000000000000000000000000000000000000001 // data of second parameter
```

- test4(0x123,["0x11221122","0x33443344"],"0x31323334353637383930","0x3132333435")

```
0x4d189ce2                                                             // function selector
0 - 0x0000000000000000000000000000000000000000000000000000000000000123 // data of first parameter
1 - 0x0000000000000000000000000000000000000000000000000000000000000080 // offset of second parameter
2 - 0x3132333435363738393000000000000000000000000000000000000000000000 // data of third parameter
3 - 0x00000000000000000000000000000000000000000000000000000000000000e0 // offset of forth parameter
4 - 0x0000000000000000000000000000000000000000000000000000000000000002 // length of second parameter
5 - 0x0000000000000000000000000000000000000000000000000000000011221122 // first data of second parameter
6 - 0x0000000000000000000000000000000000000000000000000000000033443344 // second data of second parameter
7 - 0x0000000000000000000000000000000000000000000000000000000000000005 // length of forth parameter
8 - 0x3132333435000000000000000000000000000000000000000000000000000000 // data of forth parameter

/* Some explanations
data of first parameter: uint is a fixed-length type, directly store its data
offset of second parameter: uint32[] is a dynamic array, first store its offset=0x20*4 (4 represents the number of function parameters) 
data of third parameter: bytes10 is a fixed-length type, directly store its data
offset of forth parameter: bytes is a variable-length type, first store its offset=0x80+0x20*3=0xe0 (0x80 is the offset of the previous variable-length type, 3 is the number of slots occupied by storing the length and two elements of the previous variable-length type)
length of second parameter: After storing data or offset, begin storing the length and data of variable-length data. This is the length of the second parameter.
first data of second parameter: The first data of the second parameter
second data of second parameter: The second data of the second parameter
length of forth parameter: The above completes storing the second variable-length data. This is the length of the next variable-length data.
data of forth parameter: The data of the fourth parameter
*/
```

- test5(0x123,["cxy","pika",123])

```
0x4ca373dc                                                             // function selector
0 - 0x0000000000000000000000000000000000000000000000000000000000000123 // data of first parameter
1 - 0x0000000000000000000000000000000000000000000000000000000000000040 // offset of second parameter
2 - 0x0000000000000000000000000000000000000000000000000000000000000060 // first data offset of second parameter
3 - 0x00000000000000000000000000000000000000000000000000000000000000a0 // second data offset of second parameter
4 - 0x000000000000000000000000000000000000000000000000000000000000007b // third data of second parameter
5 - 0x0000000000000000000000000000000000000000000000000000000000000003 // first data length of second parameter
6 - 0x6378790000000000000000000000000000000000000000000000000000000000 // first data of second parameter
7 - 0x0000000000000000000000000000000000000000000000000000000000000004 // second data length of second parameter
8 - 0x70696b6100000000000000000000000000000000000000000000000000000000 // second data of second parameter

/* Some explanations
data of first parameter: uint is a fixed-length type, directly store its data
offset of second parameter: struct, first store its offset=0x20*2 (2 represents the number of function parameters) 
first data offset of second parameter: Struct internal elements can be treated as function parameters. There are three elements. Since the first element is a string type, first store its offset=0x20*3=0x60
second data offset of second parameter: The second element of the struct is a string type, first store its offset=0x60+0x20+0x20=0xa0 (the first 0x20 is the size occupied by storing the first string's length, the second 0x20 is the size occupied by storing the first string's data)
third data of second parameter: The third element of the struct is a uint fixed-length type, directly store its data
first data length of second parameter: Store the length of the first element of the struct
first data of second parameter: Store the data of the first element of the struct
second data length of second parameter: Store the length of the second element of the struct
second data of second parameter: Store the data of the second element of the struct
*/
```

- test6(0x123,[["cxy1","pika1",123], ["cxy2","pika2",456]])

```
Since this is a struct array, it needs to be decomposed from inside out. Internally, there are two structs. Let's look at their encoding separately.

For the ["cxy1","pika1",123] struct, its encoding is as follows (directly encoded as function parameters):
0 - 0x0000000000000000000000000000000000000000000000000000000000000060 // offset of "cxy1"
1 - 0x00000000000000000000000000000000000000000000000000000000000000a0 // offset of "pika1"
2 - 0x000000000000000000000000000000000000000000000000000000000000007b // encoding of 123
3 - 0x0000000000000000000000000000000000000000000000000000000000000004 // length of "cxy1"
4 - 0x6378793100000000000000000000000000000000000000000000000000000000 // encoding of "cxy1"
5 - 0x0000000000000000000000000000000000000000000000000000000000000005 // length of "pika1"
6 - 0x70696b6131000000000000000000000000000000000000000000000000000000 // encoding of "pika1"

For the ["cxy2","pika2",456] struct, its encoding is as follows (directly encoded as function parameters):
0 - 0x0000000000000000000000000000000000000000000000000000000000000060 // offset of "cxy2"
1 - 0x00000000000000000000000000000000000000000000000000000000000000a0 // offset of "pika2"
2 - 0x00000000000000000000000000000000000000000000000000000000000001c8 // encoding of 456
3 - 0x0000000000000000000000000000000000000000000000000000000000000004 // length of "cxy2"
4 - 0x6378793200000000000000000000000000000000000000000000000000000000 // encoding of "cxy2"
5 - 0x0000000000000000000000000000000000000000000000000000000000000005 // length of "pika2"
6 - 0x70696b6132000000000000000000000000000000000000000000000000000000 // encoding of "pika2"

Since these are structs, we also need the offset of ["cxy1","pika1",123] and the offset of ["cxy2","pika2",456], as follows:
0 - a                                                                  // offset of ["cxy1","pika1",123]
1 - b                                                                  // offset of ["cxy2","pika2",456]
2 - 0x0000000000000000000000000000000000000000000000000000000000000060 // offset of "cxy1"
3 - 0x00000000000000000000000000000000000000000000000000000000000000a0 // offset of "pika1"
4 - 0x000000000000000000000000000000000000000000000000000000000000007b // encoding of 123
5 - 0x0000000000000000000000000000000000000000000000000000000000000004 // length of "cxy1"
6 - 0x6378793100000000000000000000000000000000000000000000000000000000 // encoding of "cxy1"
7 - 0x0000000000000000000000000000000000000000000000000000000000000005 // length of "pika1"
8 - 0x70696b6131000000000000000000000000000000000000000000000000000000 // encoding of "pika1"
9 - 0x0000000000000000000000000000000000000000000000000000000000000060 // offset of "cxy2"
10- 0x00000000000000000000000000000000000000000000000000000000000000a0 // offset of "pika2"
11- 0x00000000000000000000000000000000000000000000000000000000000001c8 // encoding of 456
12- 0x0000000000000000000000000000000000000000000000000000000000000004 // length of "cxy2"
13- 0x6378793200000000000000000000000000000000000000000000000000000000 // encoding of "cxy2"
14- 0x0000000000000000000000000000000000000000000000000000000000000005 // length of "pika2"
15- 0x70696b6132000000000000000000000000000000000000000000000000000000 // encoding of "pika2"
a points to offset of "cxy1", so a=0x20*2=0x40
b points to offset of "cxy2", so b=0x20*9=0x120

Since this is a struct array, with the struct wrapped inside an array, it should be encoded using the dynamic array encoding method, as follows:
0 - c                                                                  // offset of [["cxy1","pika1",123],["cxy2","pika2",456]]
1 - 0x0000000000000000000000000000000000000000000000000000000000000002 // count of second parameter
2 - 0x0000000000000000000000000000000000000000000000000000000000000040 // offset of ["cxy1","pika1","1"]
3 - 0x0000000000000000000000000000000000000000000000000000000000000120 // offset of ["cxy2","pika2","1"]
4 - 0x0000000000000000000000000000000000000000000000000000000000000060 // offset of "cxy1"
5 - 0x00000000000000000000000000000000000000000000000000000000000000a0 // offset of "pika1"
6 - 0x000000000000000000000000000000000000000000000000000000000000007b // encoding of 123
7 - 0x0000000000000000000000000000000000000000000000000000000000000004 // length of "cxy1"
8 - 0x6378793100000000000000000000000000000000000000000000000000000000 // encoding of "cxy1"
9 - 0x0000000000000000000000000000000000000000000000000000000000000005 // length of "pika1"
10- 0x70696b6131000000000000000000000000000000000000000000000000000000 // encoding of "pika1"
11- 0x0000000000000000000000000000000000000000000000000000000000000060 // offset of "cxy2"
12- 0x00000000000000000000000000000000000000000000000000000000000000a0 // offset of "pika2"
13- 0x00000000000000000000000000000000000000000000000000000000000001c8 // encoding of 456
14- 0x0000000000000000000000000000000000000000000000000000000000000004 // length of "cxy2"
15- 0x6378793200000000000000000000000000000000000000000000000000000000 // encoding of "cxy2"
16- 0x0000000000000000000000000000000000000000000000000000000000000005 // length of "pika2"
17- 0x70696b6132000000000000000000000000000000000000000000000000000000 // encoding of "pika2"
c is the second parameter of the function and is a dynamic type, so offset c = 0x20*2 = 0x40

So the total encoding is as follows:
0xccc5bdd2                                                             // function selector
0 - 0x0000000000000000000000000000000000000000000000000000000000000123 // encoding of 0x123
1 - 0x0000000000000000000000000000000000000000000000000000000000000040 // offset of second parameter
2 - 0x0000000000000000000000000000000000000000000000000000000000000002 // count of second parameter
3 - 0x0000000000000000000000000000000000000000000000000000000000000040 // offset of ["cxy1","pika1","1"]
4 - 0x0000000000000000000000000000000000000000000000000000000000000120 // offset of ["cxy2","pika2","1"]
5 - 0x0000000000000000000000000000000000000000000000000000000000000060 // offset of "cxy1"
6 - 0x00000000000000000000000000000000000000000000000000000000000000a0 // offset of "pika1"
7 - 0x000000000000000000000000000000000000000000000000000000000000007b // encoding of 123
8 - 0x0000000000000000000000000000000000000000000000000000000000000004 // length of "cxy1"
9 - 0x6378793100000000000000000000000000000000000000000000000000000000 // encoding of "cxy1"
10- 0x0000000000000000000000000000000000000000000000000000000000000005 // length of "pika1"
11- 0x70696b6131000000000000000000000000000000000000000000000000000000 // encoding of "pika1"
12- 0x0000000000000000000000000000000000000000000000000000000000000060 // offset of "cxy2"
13- 0x00000000000000000000000000000000000000000000000000000000000000a0 // offset of "pika2"
14- 0x00000000000000000000000000000000000000000000000000000000000001c8 // encoding of 456
15- 0x0000000000000000000000000000000000000000000000000000000000000004 // length of "cxy2"
16- 0x6378793200000000000000000000000000000000000000000000000000000000 // encoding of "cxy2"
17- 0x0000000000000000000000000000000000000000000000000000000000000005 // length of "pika2"
18- 0x70696b6132000000000000000000000000000000000000000000000000000000 // encoding of "pika2"
```

- test7([[1,2],[3]],["one","two","three"])

```
Similarly, decompose from inside out. First, the [1, 2] and [3] dynamic arrays within [[1,2],[3]]:
0 - a                                                                  // offset of [1,2]
1 - b                                                                  // offset of [3]
2 - 0x0000000000000000000000000000000000000000000000000000000000000002 // count for [1,2]
3 - 0x0000000000000000000000000000000000000000000000000000000000000001 // encoding of 1
4 - 0x0000000000000000000000000000000000000000000000000000000000000002 // encoding of 2
5 - 0x0000000000000000000000000000000000000000000000000000000000000001 // count for [3]
6 - 0x0000000000000000000000000000000000000000000000000000000000000003 // encoding of 3
a points to the start of [1,2], so a=0x20*2=0x40
b points to the start of [3], so b=0x20*5=0xa0

Then the encoding of the [[1,2],[3]] dynamic array itself:
0 - c                                                                  // offset of [[1,2],[3]]
1 - 0x0000000000000000000000000000000000000000000000000000000000000002 // count for [[1,2],[3]]
2 - 0x0000000000000000000000000000000000000000000000000000000000000040 // offset of [1,2]
3 - 0x00000000000000000000000000000000000000000000000000000000000000a0 // offset of [3]
4 - 0x0000000000000000000000000000000000000000000000000000000000000002 // count for [1,2]
5 - 0x0000000000000000000000000000000000000000000000000000000000000001 // encoding of 1
6 - 0x0000000000000000000000000000000000000000000000000000000000000002 // encoding of 2
7 - 0x0000000000000000000000000000000000000000000000000000000000000001 // count for [3]
8 - 0x0000000000000000000000000000000000000000000000000000000000000003 // encoding of 3
c points to the start of [[1,2],[3]], so a=0x20*2=0x40

Next is the encoding of each string in the ["one","two","three"] dynamic array:
0 - d                                                                  // offset for "one"
1 - e                                                                  // offset for "two"
2 - f                                                                  // offset for "three"
3 - 0x0000000000000000000000000000000000000000000000000000000000000003 // count for "one"
4 - 0x6f6e650000000000000000000000000000000000000000000000000000000000 // encoding of "one"
5 - 0x0000000000000000000000000000000000000000000000000000000000000003 // count for "two"
6 - 0x74776f0000000000000000000000000000000000000000000000000000000000 // encoding of "two"
7 - 0x0000000000000000000000000000000000000000000000000000000000000005 // count for "three"
8 - 0x7468726565000000000000000000000000000000000000000000000000000000 // encoding of "three"
d points to the start of "one", so d=0x20*3=0x60
e points to the start of "two", so e=0x20*5=0xa0
f points to the start of "three", so f=0x20*7=0xe0

Then the encoding of the ["one","two","three"] dynamic array itself:
0 - g                                                                  // offset of ["one","two","three"]
1 - 0x0000000000000000000000000000000000000000000000000000000000000003 // count for ["one","two","three"]
2 - 0x0000000000000000000000000000000000000000000000000000000000000060 // offset for "one"
3 - 0x00000000000000000000000000000000000000000000000000000000000000a0 // offset for "two"
4 - 0x00000000000000000000000000000000000000000000000000000000000000e0 // offset for "three"
5 - 0x0000000000000000000000000000000000000000000000000000000000000003 // count for "one"
6 - 0x6f6e650000000000000000000000000000000000000000000000000000000000 // encoding of "one"
7 - 0x0000000000000000000000000000000000000000000000000000000000000003 // count for "two"
8 - 0x74776f0000000000000000000000000000000000000000000000000000000000 // encoding of "two"
9 - 0x0000000000000000000000000000000000000000000000000000000000000005 // count for "three"
10- 0x7468726565000000000000000000000000000000000000000000000000000000 // encoding of "three"
We won't calculate g yet, as it involves the overall encoding of the function parameters.

The above has completed the analysis of [[1,2],[3]] and ["one","two","three"]. Finally, they are encoded as a whole:
0 - 0x0000000000000000000000000000000000000000000000000000000000000040 // offset of [[1,2],[3]]
1 - g                                                                  // offset of ["one","two","three"]
2 - 0x0000000000000000000000000000000000000000000000000000000000000002 // count for [[1,2],[3]]
3 - 0x0000000000000000000000000000000000000000000000000000000000000040 // offset of [1,2]
4 - 0x00000000000000000000000000000000000000000000000000000000000000a0 // offset of [3]
5 - 0x0000000000000000000000000000000000000000000000000000000000000002 // count for [1,2]
6 - 0x0000000000000000000000000000000000000000000000000000000000000001 // encoding of 1
7 - 0x0000000000000000000000000000000000000000000000000000000000000002 // encoding of 2
8 - 0x0000000000000000000000000000000000000000000000000000000000000001 // count for [3]
9 - 0x0000000000000000000000000000000000000000000000000000000000000003 // encoding of 3
10- 0x0000000000000000000000000000000000000000000000000000000000000003 // count for ["one","two","three"]
11- 0x0000000000000000000000000000000000000000000000000000000000000060 // offset for "one"
12- 0x00000000000000000000000000000000000000000000000000000000000000a0 // offset for "two"
13- 0x00000000000000000000000000000000000000000000000000000000000000e0 // offset for "three"
14- 0x0000000000000000000000000000000000000000000000000000000000000003 // count for "one"
15- 0x6f6e650000000000000000000000000000000000000000000000000000000000 // encoding of "one"
16- 0x0000000000000000000000000000000000000000000000000000000000000003 // count for "two"
17- 0x74776f0000000000000000000000000000000000000000000000000000000000 // encoding of "two"
18- 0x0000000000000000000000000000000000000000000000000000000000000005 // count for "three"
19- 0x7468726565000000000000000000000000000000000000000000000000000000 // encoding of "three"
g points to the start of the string array, so g=0x20*10=140

So the total selector + encoding is as follows:
0xcc80bc65                                                             // function selector
0 - 0x0000000000000000000000000000000000000000000000000000000000000040 // offset of [[1,2],[3]]
1 - 0x0000000000000000000000000000000000000000000000000000000000000140 // offset of ["one","two","three"]
2 - 0x0000000000000000000000000000000000000000000000000000000000000002 // count for [[1,2],[3]]
3 - 0x0000000000000000000000000000000000000000000000000000000000000040 // offset of [1,2]
4 - 0x00000000000000000000000000000000000000000000000000000000000000a0 // offset of [3]
5 - 0x0000000000000000000000000000000000000000000000000000000000000002 // count for [1,2]
6 - 0x0000000000000000000000000000000000000000000000000000000000000001 // encoding of 1
7 - 0x0000000000000000000000000000000000000000000000000000000000000002 // encoding of 2
8 - 0x0000000000000000000000000000000000000000000000000000000000000001 // count for [3]
9 - 0x0000000000000000000000000000000000000000000000000000000000000003 // encoding of 3
10- 0x0000000000000000000000000000000000000000000000000000000000000003 // count for ["one","two","three"]
11- 0x0000000000000000000000000000000000000000000000000000000000000060 // offset for "one"
12- 0x00000000000000000000000000000000000000000000000000000000000000a0 // offset for "two"
13- 0x00000000000000000000000000000000000000000000000000000000000000e0 // offset for "three"
14- 0x0000000000000000000000000000000000000000000000000000000000000003 // count for "one"
15- 0x6f6e650000000000000000000000000000000000000000000000000000000000 // encoding of "one"
16- 0x0000000000000000000000000000000000000000000000000000000000000003 // count for "two"
17- 0x74776f0000000000000000000000000000000000000000000000000000000000 // encoding of "two"
18- 0x0000000000000000000000000000000000000000000000000000000000000005 // count for "three"
19- 0x7468726565000000000000000000000000000000000000000000000000000000 // encoding of "three"
```

## Practice Challenges

### balsn 2020
- Challenge Name: Election

!!! note
    Note: Challenge attachments and related content can be found in the [ctf-challenges/blockchain](https://github.com/ctf-wiki/ctf-challenges/tree/master/blockchain) repository.
