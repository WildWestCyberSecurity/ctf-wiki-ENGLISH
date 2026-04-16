# False Disassembly

For some commonly used disassemblers, such as `objdump` or disassembler projects based on `objdump`, there are some disassembly flaws. There are ways to make the code disassembled by `objdump` not quite accurate.

## Jumping into the Middle of an Instruction

The simplest method is to use `jmp` to jump into the middle of an instruction for execution. This means the real code starts from "within" a certain instruction, but during disassembly, since it targets the complete instruction, it cannot list the assembly instruction code that is actually executed.

This might sound confusing and hard to understand, so let's look at an example. Given the following assembly code:

```
start:
	jmp label+1
label: 	
	DB 0x90
	mov eax, 0xf001
```

The first instruction at `label` is `DB 0x90`. Let's see the disassembly result from `objdump` for this code:

```
08048080 <start>:
  8048080: 	e9 01 00 00 00 	jmp 8048086 <label+0x1>
08048085 <label>:
  8048085: 	90 		nop
  8048086: 	b8 01 f0 00 00 	mov eax,0xf001
```

This looks fine — `DB 0x90` is correctly disassembled as `90 nop`.

But if we change the `nop` instruction to an instruction longer than 1 byte, objdump will not follow our jump and disassemble correctly, but instead linearly continue disassembling from top to bottom (linear sweep algorithm). For example, if I change `DB 0x90` to `DB 0xE9`, let's see the disassembly result from objdump again:

```
08048080 <start>:
  8048080: 	e9 01 00 00 00 	jmp 8048086 <label+0x1>
08048085 <label>:
  8048085: 	e9 b8 01 f0 00 	jmp 8f48242 <__bss_start+0xeff1b6>
```

Comparing with the previous disassembly result, you can clearly see what's happening. `DB 0xE9` is purely data and won't be executed, but the disassembly result treats it as an instruction, and subsequent results are changed accordingly.

objdump `ignores the code at the jmp destination address` and directly assembles the instruction after jmp. This way, our real code is nicely "hidden".

## Solution

How do we solve this problem? The most straightforward method appears to be manually replacing the useless `0xE9` with `0x90` using a hex editor. However, if the program performs file integrity checks and calculates checksum values, this method won't work.

So a better solution is to use IDA or similar disassemblers that perform control flow analysis. For the same problematic program, the disassembly result might look like:

```
  ---- section .text ----:
08048080 	E9 01 00 00 00 	jmp Label_08048086
			                                    ; (08048086)
			                                    ; (near + 0x1)
08048085 	DB E9

Label_08048086:
08048086	B8 01 F0 00 00	mov eax, 0xF001
			                                    ; xref ( 08048080 ) 
```

The disassembly result looks fine.

## Computing Jump Addresses at Runtime

This method can even defeat disassemblers that analyze control flow. Let's look at an example code to better understand:

```
; ----------------------------------------------------------------------------
    call earth+1
Return:
                    ; x instructions or random bytes here               x byte(s)
earth:              ; earth = Return + x
    xor eax, eax    ; align disassembly, using single byte opcode       1 byte
    pop eax         ; start of function: get return address ( Return )  1 byte
                    ; y instructions or random bytes here               y byte(s)
    add eax, x+2+y+2+1+1+z ; x+y+z+6                                    2 bytes
    push eax        ;                                                   1 byte
    ret             ;                                                   1 byte
                    ; z instructions or random bytes here               z byte(s)
; Code:
                    ; !! Code Continues Here !!
; ----------------------------------------------------------------------------
```

The program uses `call+pop` to get the return address saved on the stack when calling the function, which is essentially the `EIP` before the function call. Then junk data is inserted at the function return point. But in reality, during function execution, the return address has been modified to point to Code. Therefore, when the `earth` function returns, it jumps to `Code` to continue execution, rather than continuing at `Return`.

Let's look at a simple demo:

```
; ----------------------------------------------------------------------------
	call earth+1
earth: 	
    DB 0xE9 	        ; 1 <--- pushed return address,
		                ; E9 is opcode for jmp to disalign disas-
	; sembly
	pop eax 	        ; 1 hidden
	nop 	            ; 1
	add eax, 9 	        ; 2 hidden
	push eax 	        ; 1 hidden
	ret 	            ; 1 hidden
	DB 0xE9 	        ; 1 opcode for jmp to misalign disassembly
Code: 	                ; code continues here <--- pushed return address + 9
	nop
	nop
	nop
	ret
; ----------------------------------------------------------------------------
```

If using objdump for disassembly, just `call earth+1` will cause problems, as follows:

```
00000000 <earth-0x5>:
  0: 	e8 01 00 00 00 	call 6 <earth+0x1>
00000005 <earth>:
  5: 	e9 58 90 05 09 	jmp 9059062 <earth+0x905905d>
  a: 	00 00 		    add %al,(%eax)
  c: 	00 50 c3 		add %dl,0xffffffc3(%eax)
  f: 	e9 90 90 90 c3 	jmp c39090a4 <earth+0xc390909f>
```

Let's see the `ida` situation:

```
text:08000000 	; Segment permissions: Read/Execute
.text:08000000	 _text 	segment para public 'CODE' use32
.text:08000000 		assume cs:_text
.text:08000000 		;org 8000000h
.text:08000000 		assume 	es:nothing, ss:nothing, ds:_text,
.text:08000000 			fs:nothing, gs:nothing
.text:08000000 		dd 1E8h
.text:08000004 ; -------------------------------------------------------------
.text:08000004 		add cl, ch
.text:08000006 		pop eax
.text:08000007 		nop
.text:08000008 		add eax, 9
.text:0800000D 		push eax
.text:0800000E 		retn
.text:0800000E ; -------------------------------------------------------------
.text:0800000F 		dd 909090E9h
.text:08000013 ; -------------------------------------------------------------
.text:08000013 		retn
.text:08000013 _text 		ends
.text:08000013
.text:08000013
.text:08000013 		end
```

The last 3 `nop` instructions are all nicely hidden. Not only that, but the process of calculating `EIP` is also perfectly hidden. In fact, the entire disassembled code is completely different from the actual code.

How to solve this problem? In reality, there is no tool that can guarantee `100%` accurate disassembly. Perhaps when disassemblers achieve code emulation, they may be able to produce completely correct assembly.

In real-world situations, this is not a particularly big problem. Because with interactive disassemblers, you can specify where the code starts. And during debugging, you can clearly see the addresses where the program actually jumps to.

So at this point, in addition to static analysis, we also need dynamic debugging.




> Reference: [Beginners Guide to Basic Linux Anti Anti Debugging Techniques](http://www.stonedcoder.org/~kd/lib/14-61-1-PB.pdf)


