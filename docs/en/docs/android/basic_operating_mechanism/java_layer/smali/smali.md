# Smali

## Introduction

When executing code in the Android Java layer, it is actually the Dalvik (ART) virtual machine (implemented in C or C++ code) that parses Dalvik bytecode to simulate the program's execution process.

Naturally, Dalvik bytecode is obscure and difficult to understand, so researchers have provided a mnemonic representation for Dalvik bytecode: smali syntax. Using tools (such as apktool), we can convert existing dex files into several smali files (**generally speaking, one smali file corresponds to one class**), and then read them. For different tools, the converted smali code is generally different, after all, this syntax is not an official standard. Here we introduce the more commonly used syntax. It is worth noting that in smali syntax, registers are used throughout, but during interpretation and execution, many of them are mapped to the stack.

**It seems like an example would be appropriate here!!!!!**

## Basic Structure

The basic information of a Smali file is as follows:

- Basic class information
    - The first three lines describe the information of the class converted to this Smali file
    - If the class implements interfaces, the corresponding interface information
- If the class uses annotations, the corresponding annotation information
- Field descriptions
- Method descriptions

Interestingly, Smali code essentially restores the meaning of Java code. It mainly has the following two types of statements:

- Declaration statements are used to declare the top-down classes, methods, variable types in Java, as well as the number of registers to be used in each method, and other information.
- Execution statements execute each line of Java code, including method calls, field read/write operations, exception catching, and other operations.

Overall, Smali code has fairly good readability.

## Declaration Statements

In smali code, declaration statements generally start with `.`.

### Registers

Currently, the registers used by Dalvik are all 32-bit. For 64-bit type variables, such as double types, it uses two adjacent 32-bit registers to represent them.

Furthermore, we know that Dalvik supports up to 65536 registers (numbered from 0 to 65535), but ARM architecture CPUs only have 37 registers. So how does Dalvik handle this? In fact, each Dalvik virtual machine maintains a call stack, which is used to support the mutual mapping between virtual registers and real registers.

#### Register Declaration

When executing a specific method, Dalvik uses the `.registers` directive to determine the number of registers needed by the function. The virtual machine allocates stack space of the corresponding size for the method based on the number of requested registers. When Dalvik operates on these registers, it is actually operating on the stack space.

#### Register Naming Rules

The registers requested by a method are allocated to the function method's parameters and local variables. In smali, there are generally two naming conventions:

- v naming convention
- p naming convention

Assuming a method requests m+n registers, where local variables occupy m registers and parameters occupy n registers, the naming for different naming conventions is as follows:

|  Attribute  |       v Naming Convention        |     p Naming Convention     |
| :---------: | :-----------------------------: | :-------------------------: |
| Local Variables |   $v_0,v_1,...,v_{m-1}$   | $v_0,v_1,...,v_{m-1}$ |
| Function Parameters | $v_m,v_{m+1},...,v_{m+n}$ | $p_0,p_1,...,p_{n-1}$ |

 Generally, we prefer the p naming convention because it has better readability, allowing us to conveniently know which type a register belongs to.

And this is essentially the common register naming convention in smali syntax. Registers starting with p are all parameter registers, registers starting with v are all local variable registers, and the sum of both is the number of registers requested by the method.

### Variable Types

In Dalvik bytecode, variables are mainly divided into two types:

| Type | Members |
| ---- | ---------------------------------------- |
| Primitive Types | boolean, byte, short, char, int, long, float, double, void (only used for return value types) |
| Reference Types | objects, arrays |

However, in smali we don't actually need to put the full description of a variable's type in its entirety. We only need to be able to identify it, so what can we do? We can abbreviate it. The abbreviation method in Dalvik is as follows:

| Java Type | Type Descriptor |
| :-----: | :---: |
| boolean |   Z   |
|  byte   |   B   |
|  short  |   S   |
|  char   |   C   |
|   int   |   I   |
|  long   |   J   |
|  float  |   F   |
| double  |   D   |
|  void   |   V   |
| Object Type |   L   |
| Array Type  |   [   |

Among these, the object type can represent all classes in Java code. For example, if a class is referenced by its full name package.name.ObjectName in Java code, then in Dalvik, its descriptor is `Lpackage/name/ObjectName;`, where:

- L is the object type mentioned above.
- The `.` in the full name is replaced with `/`.
- It is followed by a `;`.

For example, `java.lang.String` has the corresponding form `Ljava/lang/String;`

> Note: The so-called full name is its complete name, not just an abbreviation. For example, String is actually java.lang.String.

The array type can represent all arrays in Java. Its general form consists of two parts from front to back:

- **Array dimension** number of [, but the maximum dimension of an array is 255.
- Data element type, where the type naturally cannot be [.

For example, the int array `int []` is represented in smali as `[I `.

For example, the array type `String[][]` is represented in smali as `[[Ljava/lang/String;`.

### Fields

In Java classes, there are generally member variables, also known as properties or fields. Fields in Java are divided into:

- Instance fields, instance properties
- Static fields, class properties, shared by all class instances.

#### Instance Fields

Declaration is as follows:

```text
#instance fields
.field <access modifier> [non-access modifier] <field name>:<field type>
```

Where the access modifiers can be:

- public
- private
- protected

Non-access modifiers can be (**clarify their usage!!!**):

- final
- volidate
- transient

For example:

```smali
# instance fields
.field private str1:Ljava/lang/String;
```

This declaration is actually:

```java
private java.lang.String str1;
```

#### Static Fields

Generally represented as follows:

```smali
#static fields
.field <access modifier> static [modifier] <field name>:<field type>
```

We won't introduce the corresponding content here, and directly give an example:

```
# static fields
.field public static str2:Ljava/lang/String;
```

The actual declaration is:

```java
public static java.lang.String str2;
```

### Methods

In smali code, methods are generally presented in the following form:

```text
# Describes the method type
.method <access modifier> [modifier] <method prototype>
      <.locals>
      [.parameter]
      [.prologue]
      [.line]
      <code logic>
      [.line]
      <code logic>
.end
```

The first line describes the type of method in comment form, generally added by the decompilation tool, and is divided into two types:

- Direct method
- Virtual method

Access modifiers may have the following forms, corresponding one-to-one with those in Java:

- public
- private
- protected

Modifiers mainly have the following range of values:

- static, indicating that the method is a static method

The method prototype is generally in the form `methodName(parameter type descriptors)return type descriptor`. Unlike Java methods, in smali's method prototype there are no names for the corresponding parameters; the names of corresponding parameters may be specified in .parameter.

.locals specifies the local variables used by the method.

The number of .parameter entries matches the number of parameters the method uses, with each statement declaring one parameter. If the method is a non-static method, we use p0 to represent this, i.e., the current object; otherwise, parameters start normally from p0.

.prologue specifies the start of the program. Code that has been obfuscated may not have this statement.

.line specifies the line number of the corresponding code in the original Java file. If the program has been obfuscated, this line generally won't be present.

**Give an example,,,,find an appropriate example!!!!!!**

### Classes

#### Basic Class Information

As follows:

```text
.class <access modifier> [non-access modifier] <class name>
.super <parent class name>
.source <source file name>
```

Where `<>` content must exist, and `[]` content is optional. Access modifiers are the so-called `public`, `protected`, `private`. Non-access modifiers refer to `final`, `abstract`. For example:

```smali
.class public final Lcom/a/b/c;
.super Ljava/lang/Object;
.source "Demo.java"
```

We can see that our class's access modifier is `public`, the non-access modifier is `final`, the class name is `com.a.b.c`, it inherits from the parent class `java.lang.Object`, and the corresponding source file is `Demo.java`.

#### Interfaces

If a class implements an interface, it is done through `.implements`, as follows:

```
#interfaces
.implements <interface name>
```

Here's an example. Generally, smali will annotate it with a comment to indicate it is an interface.

```smali
# interfaces
.implements Landroid/view/View$OnClickListener;
```

#### Categories of Classes

Java allows defining another class within a class, and even allows multiple levels of nesting. We call a class within a class an inner class. Inner classes mainly include:

- Member inner classes
- Static nested classes
- Method inner classes
- Anonymous inner classes

In smali, each class corresponds to a smali file.

#### Class References

In smali code, we use this to represent the reference to the parent class. For inner classes within the parent class, we reference them based on their nesting level. The format is `this$[level]`. For example:

```java
public class MainActivity extends Activity {   //this$0
   public class firstinner  //this$1
   {
      public class secondinner //this$2
      {
         public class thirdinner //this$3
         {

         }
      }
   }
}
```

For example, `thirdinner` references `firstinner` using `this$1`. Moreover, fields like `this$x` are all defined as `synthetic` type, indicating that these fields are automatically generated by the compiler and do not exist in the source code.

Additionally, in smali, each class corresponds to a smali file. The smali file names for these classes are:

```
MainActivity.smali
MainActivity$firstinner.smali
MainActivity$firstinner$secondinner.smali
MainActivity$firstinner$thirdinner.smali
```

### Annotations

The format of annotations is as follows:

```smali
#annotations
.annotation [annotation attribute] <annotation scope>
    [annotation field=value]
    ...
.end
```

Where, if the annotation scope is a class, the annotation will appear directly in the smali file. If the annotation scope is a method or field, it will be included within the definition of the corresponding method or field.

## Execution Statements

This section partially references http://blog.csdn.net/wizardforcel/article/details/54730253.

### Dalvik Instruction Format

Before introducing the instructions in smali syntax, let's first look at the basic format of Dalvik instructions.

The format of instructions in Dalvik mainly includes two aspects: bit description and format ID. Currently, almost all instructions in Dalvik are shown in the figure below, where the first column gives the format of the instruction described by bits, the second column is the format ID, the third column represents the corresponding syntax, and the fourth column provides an explanation.

![](./figure/Dalvik-Executable-instruction-formats.png)

#### Bit Description

In the bit description, each type of instruction in Dalvik is generally composed of the following elements:

- One op, an 8-bit opcode
- Several characters, each representing 4 bits
- Several `|`, used as separators for easier reading.
- Several $\varnothing$, also 4 characters each, indicating that this part of bits is 0.

Additionally, in the format shown above, instructions consist of one or more space-separated 16-bit words, where each word can contain the elements described above.

For example, the instruction `B|A|op CCCC` contains 2 words, 32 bits in total. The lower 8 bits of the first word are the opcode, the middle 4 bits are A, and the upper 4 bits are B. The second word is a standalone 16-bit value.

#### Format ID

However, as shown in the table:

![](./figure/Dalvik-Instruction-sample.png)

Such an instruction format can still represent different instruction meanings depending on the ID.

Generally, a format ID consists of several characters, typically containing 3 characters:

- The first digit represents the number of words

- The second:

    - If a digit, it represents the maximum number of registers the instruction contains (this is because some instructions can contain a variable number of registers)
    - If r, it indicates that a range of registers is used.

- The third character represents the type of additional data used by the instruction. As shown in the table below:

  | Mnemonic | Bit Sizes | Meaning                                  |
  | -------- | --------- | ---------------------------------------- |
  | b        | 8         | immediate signed byte                    |
  | c        | 16, 32    | constant pool index                      |
  | f        | 16        | interface constants (only used in statically linked formats) |
  | h        | 16        | immediate signed hat (high-order bits of a 32- or 64-bit value; low-order bits are all `0`) |
  | i        | 32        | immediate signed int, or 32-bit float    |
  | l        | 64        | immediate signed long, or 64-bit double  |
  | m        | 16        | method constants (only used in statically linked formats) |
  | n        | 4         | immediate signed nibble                  |
  | s        | 16        | immediate signed short                   |
  | t        | 8, 16, 32 | branch target                            |
  | x        | 0         | no additional data                       |

- If a fourth character exists:

  - s indicates static linking is used
  - i indicates the instruction should be processed inline.

#### Syntax

The basic requirements are as follows:

- Instructions start with the opcode op, followed directly by one or more parameters, separated by commas.
- The parameters of the instruction start from the first part of the instruction, with the op in the lower 8 bits. The upper 8 bits can be an 8-bit parameter, two 4-bit parameters, or empty. If the instruction exceeds 16 bits, the subsequent parts serve as parameters in order.
- The parameter `Vx` represents a register, such as v0, v1, etc. The reason for using v instead of r is to avoid naming conflicts with registers in the machine architecture that implements the virtual machine.
- The parameter `#+X` represents a constant numeric value.
- The parameter `+X` represents an address offset relative to the instruction.
- The parameter `kind@X` represents a constant pool index value, where kind represents the constant pool type, which can be one of the following four types:
    - string, string constant pool index
    - type, type constant pool index
    - field, field constant pool index
    - meth, method constant pool index

Taking the instruction `op vAA, type@BBBB` as an example, the instruction uses 1 register vAA and a 32-bit type constant pool index.

### Instruction Characteristics

Dalvik instructions roughly follow common architecture and C-style calling conventions in their calling specification, as follows:

- Parameter order is Dest-then-source.

- Suffixes are used to indicate the operation type, thereby eliminating ambiguity:

    - Normal 32-bit operations are not marked.
    - Normal 64-bit operations use `-wide` as a suffix.
    - Type-specific opcodes use their type (or simple abbreviation) as a suffix. These types include: `-boolean`, `-byte`, `-char`, `-short`, `-int`, `-long`, `-float`, `-double`, `-object`, `-string`, `-class`, and `-void`.

- Opcode suffixes are used to distinguish identical operations with different instruction styles or options. These suffixes are separated from the main name by `/`, with the primary purpose of establishing a one-to-one mapping with static constants in the code that generates and parses executables, to reduce the possibility of reader confusion.


  For example, in the instruction `move-wide/from16 vAA, vBBBB`:

  - `move` is the base opcode, indicating this is a basic operation for moving register values.
  - `wide` is the name suffix, indicating the instruction operates on 64-bit data.
  - `from16` is the opcode suffix, indicating the source is a 16-bit register reference variable.
  - `vAA` is the destination register, with a value range of `v0` - `v255`.
  - `vBBBB` is the source register, with a value range of `v0` - `v65535`.

### Specific Instructions

Here, we introduce each instruction's meaning in detail and categorize them as much as possible.

#### Nop Instruction

The nop instruction performs no operation and is generally used for code alignment.

#### Data Definition Instructions

| op&id  | Syntax                                   | Parameters                               | Description                              |
| ------ | ---------------------------------------- | ---------------------------------------- | ---------------------------------------- |
| 2 11n  | const/4 vA, #+B                          | `A:` destination register (4 bits)           `B:` signed integer (4 bits) | Moves the given value (sign-extended to 32 bits) into the specified register. |
| 13 21s | const/16 vAA, #+BBBB                     | `A:` destination register (8 bits)           `B:` signed integer (16 bits) | Moves the given value (sign-extended to 32 bits) into the specified register. |
| 14 31i | const vAA, #+BBBBBBBB                    | `A:` destination register (8 bits)           `B:` arbitrary 32-bit constant | Moves the given value into the specified register. |
| 15 21h | const/high16 vAA, #+BBBB0000             | `A:` destination register (8 bits)           `B:` signed integer (16 bits) | Moves the given value (right-zero-extended to 32 bits) into the specified register. |
| 16 21s | const-wide/16 vAA, #+BBBB                | `A:` destination register (8 bits)           `B:` signed integer (16 bits) | Moves the given value (sign-extended to 64 bits) into the specified register pair. |
| 17 31i | const-wide/32 vAA, #+BBBBBBBB            | `A:` destination register (8 bits)            `B:` signed integer (32 bits) | Moves the given value (sign-extended to 64 bits) into the specified register pair. |
| 18 51l | const-wide vAA, #+BBBBBBBBBBBBBBBB       | `A:` destination register (8 bits)           `B:` arbitrary double-width (64-bit) constant | Moves the given value into the specified register pair. |
| 19 21h | const-wide/high16 vAA, #+BBBB000000000000 | `A:` destination register (8 bits)           `B:` signed integer (16 bits) | Moves the given value (right-zero-extended to 64 bits) into the specified register pair. |
| 1a 21c | const-string vAA, string@BBBB            | `A:` destination register (8 bits)           `B:` string index | Assigns the given string reference to the specified register. |
| 1b 31c | const-string/jumbo vAA, string@BBBBBBBB  | `A:` destination register (8 bits)            `B:` string index | Assigns the given string reference (larger) to the specified register. |
| 1c 21c | const-class vAA, type@BBBB               | `A:` destination register (8 bits)           `B:` type index | Assigns the given class reference to the specified register. If the specified type is a primitive type, a reference to the degenerate class of the primitive type will be stored. |

For example, if the Java code is as follows:

```java
boolean z = true;
z = false;
byte b = 1;
short s = 2;
int i = 3;
long l = 4;
float f = 0.1f;
double d = 0.2;
String str = "test";
Class c = Object.class;
```

Then the compiled code is as follows:

```smali
const/4 v10, 0x1
const/4 v10, 0x0
const/4 v0, 0x1
const/4 v8, 0x2
const/4 v5, 0x3
const-wide/16 v6, 0x4
const v4, 0x3dcccccd    # 0.1f
const-wide v2, 0x3fc999999999999aL    # 0.2
const-string v9, "test"
const-class v1, Ljava/lang/Object;
```

As we can see, different syntax is used depending on the data type size. Additionally, we can see that the literal value of the float is 0x3dcccccd, which is actually 0.1. For information about how floating-point numbers are represented in computers, please search online. Furthermore, generally speaking, smali will automatically convert the string's ID to its actual string for us.

#### Data Movement

Data movement instructions mainly move data from one register or memory location to another.

| op&id  | Syntax                          | Parameters                            | Description                     |
| ------ | ----------------------------- | :---------------------------------- | ------------------------------- |
| 01 12x | move vA, vB                   | `A:` destination register (4 bits) `B:` source register (4 bits) | vA=vB |
| 02 22x | move/from16 vAA, vBBBB        | `A:` destination register (8 bits) `B:` source register (16 bits) | vAA=vBBBB |
| 03 32x | move/16 vAAAA, vBBBB          | `A:` destination register (16 bits) `B:` source register (16 bits) | vAAAA=VBBBB |
| 04 12x | move-wide vA, vB              | `A:` destination register pair (4 bits) `B:` source register pair (4 bits) | vA, v(A+1)=vB, V(B+1) |
| 05 22x | move-wide/from16 vAA, vBBBB   | `A:` destination register pair (8 bits) `B:` source register pair (16 bit) | vAA, v(AA+1)=vBBBB, V(BBBB+1) |
| 06 32x | move-wide/16 vAAAA, vBBBB     | `A:` destination register pair (16 bits) `B:` source register pair (16 bit) | vAAAA, v(AAAA+1)=vBBBB, V(BBBB+1) |
| 07 12x | move-object vA, vB            | `A:` destination register (4 bits) `B:` source register (4 bits) | Object reference assignment, vA=vB |
| 08 22x | move-object/from16 vAA, vBBBB | `A:` destination register (8 bits) `B:` source register (16 bits) | Object reference assignment, vAA=vBBBB |
| 09 32x | move-object/16 vAAAA, vBBBB   | `A:` destination register (16 bits) `B:` source register (16 bits) | Object reference assignment, vAAAA=vBBBB |
| 0a 11x | move-result vAA               | `A:` destination register (8 bits)  | Moves the function call return value into the vAA register. |
| 0b 11x | move-result-wide vAA          | `A:` destination register pair (8 bits) | Moves the function call return value into the vAA register. |
| 0c 11x | move-result-object vAA        | `A:` destination register (8 bits)  | Moves the function call return object reference into the vAA register. |
| 0d 11x | move-exception vAA            | `A:` destination register (8 bits)  | Saves the caught exception into the given register. |

Among these, the `move` series of instructions and `move-result` are used to handle primitive types that are 32 bits or less.

The `move-wide` series of instructions and `move-result-wide` are used to handle 64-bit types, including `long` and `double` types.

The `move-object` series of instructions and `move-result-object` are used to handle object references.

Additionally, the suffixes (`/from16`, `/16`) only affect the number of bits in the bytecode and the range of registers, and do not affect the instruction logic.

#### Data Conversion Instructions

Data conversion instructions mainly convert one data type to another. The currently available instructions are as follows:

| **Instruction** | **Description** |
| --------------- | ----------------- |
| neg-int         | Negate an integer |
| not-int         | Bitwise NOT of an integer |
| neg-long        | Negate a long integer |
| not-long        | Bitwise NOT of a long integer |
| neg-float       | Negate a single-precision float |
| neg-double      | Negate a double-precision float |
| int-to-long     | Convert integer to long integer |
| int-to-float    | Convert integer to single-precision float |
| int-to-dobule   | Convert integer to double-precision float |
| long-to-int     | Convert long integer to integer |
| long-to-float   | Convert long integer to single-precision float |
| long-to-double  | Convert long integer to double-precision float |
| float-to-int    | Convert single-precision float to integer |
| float-to-long   | Convert single-precision float to long integer |
| float-to-double | Convert single-precision float to double-precision float |
| double-to-int   | Convert double-precision float to integer |
| double-to-long  | Convert double-precision float to long integer |
| double-to-float | Convert double-precision float to single-precision float |
| int-to-byte     | Convert integer to byte |
| int-to-char     | Convert integer to char |
| int-to-short    | Convert integer to short |

For example, `int-to-short v0,v1` means force-converting the value of register v1 to short type and placing it into v0.

#### Math Operation Instructions

Math operation instructions include arithmetic operation instructions and logical operation instructions. Among them, arithmetic operation instructions include addition, subtraction, multiplication, division, modulo, shift, and other operations. Logical operation instructions mainly perform AND, OR, NOT, XOR, and other operations between values.

Data operation instructions have the following four categories, where the operator is binop.

| **Instruction** | **Description** |
| -------------------------- | ------------------------------ |
| binop vAA, vBB, vCC        | Performs operation on vBB register and vCC register, result saved to vAA register |
| binop/2addr vA, vB         | Performs operation on vA register and vB register, result saved to vA register |
| binop/lit16 vA, vB, #+CCCC | Performs operation on vB register and constant CCCC, result saved to vA register |
| binop/lit8 vAA, vBB, #+CC  | Performs operation on vBB register and constant CC, result saved to vAA register |

The latter 3 categories of instructions have the additional suffixes 2addr, lit16, and lit8 compared to the first category. However, for instructions with the same base bytecode, the operations performed are similar. So here we mainly introduce the first category of instructions. In addition, depending on the data type, a data type suffix is added after the base bytecode, such as `-int` or `-long`, indicating the data type being operated on is integer or long integer respectively. The operation types of the first category of instructions are as follows:

| Operation Type | **Description** |
| --------- | ------------------ |
| add-type  | vBB + vCC          |
| sub-type  | vBB - vCC          |
| mul-type  | vBB * vCC          |
| div-type  | vBB / vCC          |
| rem-type  | vBB % vCC          |
| and-type  | vBB & vCC          |
| or-type   | vBB \| vCC         |
| xor-type  | vBB ^ vCC          |
| shl-type  | vBB << vCC, signed left shift |
| shr-type  | vBB >> vCC, signed right shift |
| ushr-type | vBB >>> vCC, unsigned right shift |

Where the -type after the base bytecode can be -int, -long, -float, -double.

For example, the Java source code is:

```java
int a = 5, b = 2;
a += b;
a -= b;
a *= b;
a /= b;
a %= b;
a &= b;
a |= b;
a ^= b;
a <<= b;
a >>= b;
a >>>= b;
```

The corresponding smali is:

```smali
const/4 v0, 0x5
const/4 v1, 0x2
add-int/2addr v0, v1
sub-int/2addr v0, v1
mul-int/2addr v0, v1
div-int/2addr v0, v1
rem-int/2addr v0, v1
and-int/2addr v0, v1
or-int/2addr v0, v1
xor-int/2addr v0, v1
shl-int/2addr v0, v1
shr-int/2addr v0, v1
ushr-int/2addr v0, v1
```

#### Array Operation Instructions

Array operation instructions implement operations such as getting array length, creating new arrays, array assignment, and getting/setting array elements.

| **Instruction** | **Description** |
| ---------------------------------------- | ---------------------------------------- |
| array-length vA, vB                      | Gets the length of the array in the given vB register and assigns it to the vA register. The array length refers to the number of elements in the array. |
| new-array vA, vB, type@CCCC              | Constructs an array of size vB with element type type@CCCC, and assigns the reference to the vA register |
| filled-new-array {vC, vD, vE, vF, vG},type@BBBB | Constructs an array of size vA with element type type@BBBB and fills the array contents. The vA register is implicitly used; besides specifying the array size, it also specifies the number of parameters. vC~vG is the sequence of parameter registers used |
| filled-new-array/range {vCCCC  ..vNNNN}, type@BBBB | The instruction function is the same as filled-new-array {vC, vD, vE, vF, vG},type@BBBB, except that parameter registers use the range suffix to specify the value range. vC is the first parameter register, N = A + C - 1 |
| fill-array-data vAA, +BBBBBBBB           | Fills the array with specified data. The vAA register is the array reference, which must be an array of primitive types. A data table follows immediately after the instruction |
| new-array/jumbo vAAAA, vBBBB,type@CCCCCCCC | The instruction function is the same as new-array vA,vB,type@CCCC, but the register values and instruction index ranges are larger (new instruction added in Android 4.0) |
| filled-new-array/jumbo {vCCCC  ..vNNNN},type@BBBBBBBB | The instruction function is the same as filled-new-array/range {vCCCC  ..vNNNN},type@BBBB, except the index value range is larger (new instruction added in Android 4.0) |
| arrayop vAA, vBB, vCC                    | Gets and sets array elements specified by the vBB register. The vCC register specifies the array element index, and the vAA register is used to store the value of the array element being read or set. Reading elements uses aget-type instructions, and setting elements uses aput-type instructions. Different instruction suffixes follow the instruction depending on the type stored in the array. The instruction list is as follows: aget, aget-wide, aget-object, aget-boolean, aget-byte, aget-char, aget-short, aput, aput-wide, aput-object, aput-boolean, aput-byte, aput-char, aput-short. |

We can define an array as follows:

```java
int[] arr = new int[10];
```

The corresponding smali is:

```smali
const/4 v1, 0xa
new-array v0, v1, I
```

If we directly initialize the array at definition time, as follows:

```smali
int[] arr = {1, 2, 3, 4, 5};
```

The corresponding smali is:

```smali
const/4 v1, 0x1
const/4 v2, 0x2
const/4 v3, 0x3
const/4 v4, 0x4
const/4 v5, 0x5
filled-new-array {v1, v2, v3, v4, v5}, I
move-result v0
```

When the registers are consecutive, it can also be written as the following code:

```smali
const/4 v1, 0x1
const/4 v2, 0x2
const/4 v3, 0x3
const/4 v4, 0x4
const/4 v5, 0x5
filled-new-array-range {v1..v5}, I
move-result v0
```

#### Instance Operation Instructions

Instance operation instructions mainly implement functions such as instance type conversion, checking, and creation.

| **Instruction** | **Description** |
| ---------------------------------------- | ---------------------------------------- |
| check-cast vAA, type@BBBB                | Converts the object reference in the vAA register to type type@BBBB. If it fails, a ClassCastException is thrown. If type B specifies a primitive type, for a non-primitive type A, it will always fail at runtime |
| instance-of vA, vB, type@CCCC            | Determines whether the object reference in the vB register can be converted to the specified type. If yes, the vA register is assigned 1; otherwise, the vA register is assigned 0. |
| new-instance vAA, type@BBBB              | Constructs a new instance of the specified type and assigns the object reference to the vAA register. The type specified by type cannot be an array class |
| check-cast/jumbo vAAAA, type@BBBBBBBB    | Same function as check-cast vAA, type@BBBB, but with larger register values and instruction index ranges (new instruction added in Android 4.0) |
| instance-of/jumbo vAAAA, vBBBB, type@CCCCCCCC | Same function as instance-of vA, vB, type@CCCC, but with larger register values and instruction index ranges (new instruction added in Android 4.0) |
| new-instance/jumbo vAAAA, type@BBBBBBBB  | Same function as new-instance vAA, type@BBBB, but with larger register values and instruction index ranges (new instruction added in Android 4.0) |

For example, we define an instance:

```java
Object obj = new Object();
```

The corresponding smali code is:

```smali
new-instance v0, Ljava/lang/Object;
invoke-direct-empty {v0}, Ljava/lang/Object;-><init>()V
```

As another example, we can perform the following type check:

```java
String s = "test";
boolean b = s instanceof String;
```

The corresponding smali code is:

```smali
const-string v0, "test"
instance-of v1, v0, Ljava/lang/String;
```

If we perform a type cast:

```java
String s = "test";
Object o = (Object)s;
```

The corresponding smali code is:

```smali
const-string v0, "test"
check-cast v0, Ljava/lang/Object;
move-object v1, v0
```

#### Field Operation Instructions

Field operation instructions mainly perform read and write operations on instance fields. Read operations are marked with get, i.e., vx=vy.field. Write operations are marked with put, i.e., vy.field=vx.

For Java classes, there are mainly two types of fields: instance fields and static fields. Instance fields are marked with i prefixed to the operation instruction, such as iget, iput. Static fields are marked with s prefixed to the operation instruction, such as sput, sget.

Additionally, operations on fields of different sizes are distinguished by adding suffixes after the instruction. For example, the iget-byte instruction means reading the value of a byte-type instance field, and the iput-short instruction means the type of the instance field being set is short.

Instance field operation instructions include:

iget, iget-wide, iget-object, iget-boolean, iget-byte, iget-char, iget-short,

iput, iput-wide, iput-object, iput-boolean, iput-byte, iput-char, iput-short.

Static field operation instructions include:

sget, sget-wide, sget-object, sget-boolean, sget-byte, sget-char, sget-short,

sput, sput-wide, sput-object, sput-boolean, sput-byte, sput-char, sput-short.

If we write the following code:

```java
int[] arr = new int[2];
int b = arr[0];
arr[1] = b;
```

The corresponding smali is:

```smali
const/4 v0, 0x2
new-array v1, v0, I
const/4 v0, 0x0
aget-int v2, v1, v0
const/4 v0, 0x1
aput-int v2, v1, v0
```

If we want to get the static int field staticField of class com.example.test, the smali is:

```smali
sget v0, Lcom/example/Test;->staticField:I
```

#### Comparison Instructions

Comparison instructions implement operations that compare the values of two registers (floating-point or long integer types).

There are currently 5 comparison instructions, occupying opcodes from 2d to 31. The mnemonic structure for all 5 comparison instructions is:

`cmpkind vAA, vBB, vCC`

Where `A` is the destination register (8 bits) that stores the result, `B` is the first source register or register pair; `C` is the second source register or register pair.

These 5 comparison instructions are as follows:

| op   | Instruction           | Comment              |
| ---- | --------------------- | -------------------- |
| 2d   | cmpl-float (lt bias)  | Compare two single-precision floats |
| 2e   | cmpg-float (gt bias)  | Compare two single-precision floats |
| 2f   | cmpl-double (lt bias) | Compare two double-precision floats |
| 30   | cmpg-double (gt bias) | Compare two double-precision floats |
| 31   | cmp-long              | Compare two long integers |

The difference between lt bias and gt bias only matters when comparing with NaN values. When NaN values are not involved, the execution logic of all five instructions is:

**If the vBB register is greater than the vCC register, the result is 1; if equal, the result is 0; if less than, the result is -1.**

If one of the compared values is `NaN`, then "gt bias" instructions return `1`, and "lt bias" instructions return `-1`. Why are two types of comparison instructions needed specifically for `NaN`? This is to exclude illegal values like `NaN` from the true branch.

For example, when checking whether a floating-point number satisfies the condition `x < y`, it is recommended to use the `cmpg-float` instruction. Because if the result is `-1`, it indicates the test result is true, and any other value indicates the test result is false. The false cases include "the current comparison is valid but x >= y", or "one of the values is `NaN`".

#### Jump Instructions

Jump instructions implement the operation of jumping from the current address to a specified offset. There are three types of jump instructions in the Dalvik instruction set:

- goto, unconditional jump
- switch, branch jump
- if, conditional jump

##### goto Instructions

As follows:

| Instruction       | Meaning                    |
| ----------------- | ----------------------- |
| goto +AA          | Unconditionally jump to the specified offset. Offset AA cannot be 0 |
| goto/16 +AAAA     | Unconditionally jump to the specified offset. Offset AAAA cannot be 0 |
| goto/32 +AAAAAAAA | Unconditionally jump to the specified offset |

##### if Instructions

The if instructions are mainly divided into two types: if-test and if-testz. `if-test vA,vB,+CCCC` compares vA with vB, and if the comparison result is satisfied, jumps to the offset specified by CCCC (relative to the current offset). The offset CCCC cannot be 0. The if-test type instructions are as follows:

| Instruction            | Description    |
| -------------------- | ------------ |
| `if-eq vA,vB,target` | Jump if vA=vB  |
| `if-ne vA,vB,target` | Jump if vA!=vB |
| `if-lt vA,vB,target` | Jump if vA<vB  |
| `if-gt vA,vB,target` | Jump if vA>vB  |
| `if-ge vA,vB,target` | Jump if vA>=vB |
| `if-le vA,vB,target` | Jump if vA<=vB |

The if-testz type instructions are as follows:

| Instruction       | Description   |
| ----------------- | ----------- |
| if-eqz vAA,target | Jump if vA=0  |
| if-nez vAA,target | Jump if vA!=0 |
| if-ltz vAA,target | Jump if vA<0  |
| if-gtz vAA,target | Jump if vA>0  |
| if-lez vAA,target | Jump if vA<=0 |
| if-gtz vAA,target | Jump if vA>=0 |

For example, the Java code is as follows:

```java
int a = 10
if(a > 0)
    a = 1;
else
    a = 0;
```

The smali code is:

```smali
const/4 v0, 0xa
if-lez v0, :cond_0 # if block start
const/4 v0, 0x1
goto :cond_1       # if block end
:cond_0            # else block start
const/4 v0, 0x0
:cond_1            # else block end
```

In the case of only an if statement:

```java
int a = 10;
if(a > 0)
    a = 1;
```

The smali code is:

```smali
const/4 v0, 0xa
if-lez v0, :cond_0 # if block start
const/4 v0, 0x1
:cond_0            # if block end
```

##### switch Instructions

As follows:

| Instruction                 | Meaning                                  |
| --------------------------- | ---------------------------------------- |
| packed-switch vAA,+BBBBBBBB | The vAA register is the value to be judged in the switch branch. BBBBBBBB points to an offset table in packed-switch-payload format, where the values in the table are regularly incrementing. |
| sparse-switch vAA,+BBBBBBBB | The vAA register is the value to be judged in the switch branch. BBBBBBBB points to an offset table in sparse-switch-payload format, where the values in the table are irregular offsets. |

For the first type of incrementing switch:

```java
int a = 10;
switch (a){
    case 0:
        a = 1;
        break;
    case 1:
        a = 5;
        break;
    case 2:
        a = 10;
        break;
    case 3:
        a = 20;
        break;
}
```

The corresponding smali is:

```smali
const/16 v0, 0xa

packed-switch v0, :pswitch_data_0 # switch start

:pswitch_0                        # case 0
const/4 v0, 0x1
goto :goto_0

:pswitch_1                        # case 1
const/4 v0, 0x5
goto :goto_0

:pswitch_2                        # case 2
const/16 v0, 0xa
goto :goto_0

:pswitch_3                        # case 3
const/16 v0, 0x14
goto :goto_0

:goto_0                           # switch end
return-void

:pswitch_data_0                   # jump table start
.packed-switch 0x0                # starting from 0
    :pswitch_0
    :pswitch_1
    :pswitch_2
    :pswitch_3
.end packed-switch                # jump table end
```

For a non-incrementing switch, the code is as follows:

```smali
int a = 10;
switch (a){
    case 0:
        a = 1;
        break;
    case 10:
        a = 5;
        break;
    case 20:
        a = 10;
        break;
    case 30:
        a = 20;
        break;
}
```

The corresponding smali is:

```smali
const/16 v0, 0xa

sparse-switch v0, :sswitch_data_0 # switch start

:sswitch_0                        # case 0
const/4 v0, 0x1
goto :goto_0

:sswitch_1                        # case 10
const/4 v0, 0x5

goto :goto_0

:sswitch_2                        # case 20
const/16 v0, 0xa
goto :goto_0

:sswitch_3                        # case 15
const/16 v0, 0x14
goto :goto_0

:goto_0                           # switch end
return-void

.line 55
:sswitch_data_0                   # jump table start
.sparse-switch
    0x0 -> :sswitch_0
    0xa -> :sswitch_1
    0x14 -> :sswitch_2
    0x1e -> :sswitch_3
.end sparse-switch                # jump table end
```



#### Lock Instructions

Lock instructions are used in multi-threaded programs. They include the following two instructions:

| **Instruction** | **Description** |
| ----------------- | --------- |
| monitor-enter vAA | Acquires the lock for the specified object |
| monitor-exit vAA  | Releases the lock for the specified object |

#### Method Call Instructions

Method call instructions implement the operation of calling an instance's methods. The base is invoke, and suffixes are added based on the category of the called method, such as virtual methods, parent class methods, etc. Finally, range is optionally used to specify the register range. Generally, they are divided into two categories:

- invoke-kind {vC, vD, vE, vF, vG},meth@BBBB

- invoke-kind/range {vCCCC  .. vNNNN},meth@BBBB


  Overall, the general instructions are as follows:

| **Instruction** | **Description** |
| ---------------------------------------- | --------- |
| invoke-virtual or invoke-virtual/range    | Calls an instance's virtual method |
| invoke-super or invoke-super/range        | Calls an instance's parent class method |
| invoke-direct or invoke-direct/range      | Calls an instance's direct method |
| invoke-static or invoke-static/range      | Calls an instance's static method |
| invoke-interface or invoke-interface/range | Calls an instance's interface method |

In Dalvik, direct methods refer to all instance constructors and `private` instance methods of a class. `protected` or `public` methods are called virtual methods.

#### Exception Instructions

The throw vAA instruction is used to throw an exception of the type specified in the vAA register.

##### try catch

First, let's look at try catch, as follows:

```java
int a = 10;
try {
    callSomeMethod();
} catch (Exception e) {
    a = 0;
}
callAnotherMethod();
```

The corresponding smali is:

```smali
const/16 v0, 0xa

:try_start_0            # try block start
invoke-direct {p0}, Lnet/flygon/myapplication/SubActivity;->callSomeMethod()V
:try_end_0              # try block end

.catch Ljava/lang/Exception; {:try_start_0 .. :try_end_0} :catch_0

:goto_0
invoke-direct {p0}, Lnet/flygon/myapplication/SubActivity;->callAnotherMethod()V
return-void

:catch_0                # catch block start
move-exception v1
const/4 v0, 0x0
goto :goto_0            # catch block end
```

As we can see, if an exception exists between `:try_start_0` and `:try_end_0`, it will search downward for a `.catch` (or `.catch-all`) statement. When the condition is met, it jumps to the label position, which is `:catch_0` here. After it finishes, there is a `goto` to jump back.

##### try-finally

The Java code is as follows:

```java
int a = 10;
try {
    callSomeMethod();
} finally {
    a = 0;
}
callAnotherMethod();
```

The corresponding smali code is:

```smali
const/16 v0, 0xa

:try_start_0            # try block start
invoke-direct {p0}, Lnet/flygon/myapplication/SubActivity;->callSomeMethod()V
:try_end_0              # try block end

.catchall {:try_start_0 .. :try_end_0} :catchall_0

const/4 v0, 0x0         # a copy placed outside
invoke-direct {p0}, Lnet/flygon/myapplication/SubActivity;->callAnotherMethod()V
return-void

:catchall_0             # finally block start
move-exception v1
const/4 v0, 0x0
throw v1                # finally block end
```

As we can see, since the logic in `finally` is executed regardless of whether an exception occurs, there are two parts in the code.

##### try-catch-finally

When we use both catch and finally together, as follows:

```java
int a = 10;
try {
    callSomeMethod();
} catch (Exception e) {
    a = 1;
}
finally {
    a = 0;
}
callAnotherMethod();
```

The corresponding smali code is:

```smali
const/16 v0, 0xa

:try_start_0            # try block start
invoke-direct {p0}, Lnet/flygon/myapplication/SubActivity;->callSomeMethod()V
:try_end_0              # try block end

.catch Ljava/lang/Exception; {:try_start_0 .. :try_end_0} :catch_0
.catchall {:try_start_0 .. :try_end_0} :catchall_0

const/4 v0, 0x0         # a copy placed outside

:goto_0
invoke-direct {p0}, Lnet/flygon/myapplication/SubActivity;->callAnotherMethod()V
return-void

:catch_0                # catch block start
move-exception v1
const/4 v0, 0x1
const/4 v0, 0x0         # a copy placed inside the catch block
goto :goto_0            # catch block end

:catchall_0             # finally block start
move-exception v2
const/4 v0, 0x0
throw v2                # finally block end
```

#### Return Instructions

In Java, we use Return to return the execution result of a method. Similarly, in Dalvik, we also need return instructions to return method execution results.

| Instruction       | Description    |
| ----------------- | -------------- |
| return-void       | Returns nothing |
| return vAA        | Returns a 32-bit non-object type value |
| return-wide vAA   | Returns a 64-bit non-object type value |
| return-object vAA | Returns an object type reference |

## java2smali

**!!From Java code to smali code!!**

This example is from <u>http://blog.csdn.net/dd864140130/article/details/52076515</u>.

The Java code is as follows:

```java
public class MainActivity extends Activity implements View.OnClickListener {

    private String TAG = "MainActivity";
    private static final float pi = (float) 3.14;

    public volatile boolean running = false;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }

    @Override
    public void onClick(View view) {
        int result = add(4, 5);
        System.out.println(result);

        result = sub(9, 3);

        if (result > 4) {
            log(result);
        }
    }

    public int add(int x, int y) {
        return x + y;
    }

    public synchronized int sub(int x, int y) {
        return x + y;
    }

    public static void log(int result) {
        Log.d("MainActivity", "the result:" + result);
    }


}
```

The corresponding smali code is as follows:

```smali
# File header description
.class public Lcom/social_touch/demo/MainActivity;
.super Landroid/app/Activity;# Specifies the parent class of MainActivity
.source "MainActivity.java"# Source file name

# Indicates that View.OnClickListener interface is implemented
# interfaces
.implements Landroid/view/View$OnClickListener;

# Defines the static float field pi
# static fields
.field private static final pi:F = 3.14f

# Defines the String type field TAG
# instance fields
.field private TAG:Ljava/lang/String;

# Defines the boolean type field running
.field public volatile running:Z

# Constructor method. If you're wondering where this method comes from, go learn JVM fundamentals
# direct methods
.method public constructor <init>()V
    .locals 1# Indicates the function uses one local variable

    .prologue# Indicates the method code officially begins
    .line 8# Corresponds to line 8 in the Java source file
    # Calls the init() method in Activity
    invoke-direct {p0}, Landroid/app/Activity;-><init>()V

    .line 10
    const-string v0, "MainActivity"

    iput-object v0, p0, Lcom/social_touch/demo/MainActivity;->TAG:Ljava/lang/String;

    .line 13
    const/4 v0, 0x0

    iput-boolean v0, p0, Lcom/social_touch/demo/MainActivity;->running:Z

    return-void
.end method

# Static method log()
.method public static log(I)V
    .locals 3
    .parameter "result"# Represents the result parameter

    .prologue
    .line 42
    # Assigns "MainActivity" to register v0
    const-string v0, "MainActivity"
    # Creates a StringBuilder object and assigns its reference to register v1
    new-instance v1, Ljava/lang/StringBuilder;

    # Calls the constructor of StringBuilder
    invoke-direct {v1}, Ljava/lang/StringBuilder;-><init>()V

    # Assigns "the result:" to register v2
    const-string v2, "the result:"

    # In {v1,v2}, register v1 stores the reference to the StringBuilder object.
    # Calls the append(String str) method of StringBuilder, register v2 is the parameter register.
    invoke-virtual {v1, v2}, Ljava/lang/StringBuilder;->append(Ljava/lang/String;)Ljava/lang/StringBuilder;

    # Gets the result of the previous method execution. At this point v1 stores the result after append() execution.
    # The reason it still returns v1 is that the append() method returns its own reference
    move-result-object v1

    # Continues calling the append() method. p0 represents the first parameter register, i.e., the result parameter mentioned above
    invoke-virtual {v1, p0}, Ljava/lang/StringBuilder;->append(I)Ljava/lang/StringBuilder;

    # Same as above
    move-result-object v1

    # Calls the toString() method of the StringBuilder object
    invoke-virtual {v1}, Ljava/lang/StringBuilder;->toString()Ljava/lang/String;

    # Gets the result of the previous method execution. The toString() method returns a new String object, so v1 now stores the reference to the String object
    move-result-object v1

    # Calls the static method e() of the Log class. Since e() is a static method, {v0,v1} are parameter registers
    invoke-static {v0, v1}, Landroid/util/Log;->d(Ljava/lang/String;Ljava/lang/String;)I

    .line 43
    # Calls the return instruction. No value is returned here
    return-void
.end method


# virtual methods
.method public add(II)I
    .locals 1
    .parameter "x"# First parameter
    .parameter "y"# Second parameter

    .prologue
    .line 34

    # Calls the add-int instruction to sum and assigns the result to register v0
    add-int v0, p1, p2

    # Returns the value in register v0
    return v0
.end method


.method public onClick(Landroid/view/View;)V
    .locals 4
    .parameter "view" # Parameter view

    .prologue
    const/4 v3, 0x4 # Assigns 4 to register v3

    .line 23# Line 23 in the Java source file
    const/4 v1, 0x5# Assigns 5 to register v1

    # Calls the add() method
    invoke-virtual {p0, v3, v1}, Lcom/social_touch/demo/MainActivity;->add(II)I

    # Gets the execution result of the add method from register v0
    move-result v0

    .line 24# Line 24 in the Java source file
    .local v0, result:I

    # Assigns the PrintStream object reference out to register v1
    sget-object v1, Ljava/lang/System;->out:Ljava/io/PrintStream;

    # Executes the println() method of the out object
    invoke-virtual {v1, v0}, Ljava/io/PrintStream;->println(I)V

    .line 26

    const/16 v1, 0x9# Assigns 9 to register v1
    const/4 v2, 0x3# Assigns 3 to register v2

    # Calls the sub() method. In {p0,v1,v2}, p0 refers to this, i.e., the current object; v1 and v2 are parameters
    invoke-virtual {p0, v1, v2}, Lcom/social_touch/demo/MainActivity;->sub(II)I
    # Gets the execution result of the sub() method from register v0
    move-result v0

    .line 28
    if-le v0, v3, :cond_0# If the value in register v0 is less than or equal to the value in register v3, jump to cond_0 to continue execution

    .line 29

    # Calls the static method log()
    invoke-static {v0}, Lcom/social_touch/demo/MainActivity;->log(I)V

    .line 31
    :cond_0
    return-void
.end method

.method protected onCreate(Landroid/os/Bundle;)V
    .locals 1
    .parameter "savedInstanceState" # Parameter savedInstanceState

    .prologue
    .line 17

    # Calls the parent class method onCreate()
    invoke-super {p0, p1}, Landroid/app/Activity;->onCreate(Landroid/os/Bundle;)V

    .line 18

    const v0, 0x7f04001a# Assigns 0x7f04001a to register v0

    # Calls the setContentView() method
    invoke-virtual {p0, v0}, Lcom/social_touch/demo/MainActivity;->setContentView(I)V

    .line 19
    return-void
.end method

# declared-synchronized indicates this method is a synchronized method
.method public declared-synchronized sub(II)I
    .locals 1
    .parameter "x"
    .parameter "y"

    .prologue
    .line 38

    monitor-enter p0# Acquires the lock object p0 for this method
     add-int v0, p1, p2
    # Releases the lock object
    monitor-exit p0

    return v0
.end method
```

## Compiling - smali2dex

Given a smali file, we can compile the smali file into a dex file using the following method:

```shell
java -jar smali.jar assemble  src.smali -o src.dex
```

Where smali.jar is from <u>https://bitbucket.org/JesusFreke/smali/overview</u>.

## Running smali

After compiling the smali file into a dex file, we can further execute it.

First, use adb to push the dex file to the phone:

```shell
adb push main.dex /sdcard/
```

 Then use the following command to execute:

```shell
adb shell dalvikvm -cp /sdcard/main.dex main
```

 Where:

-   Here we use the dalvikvm command.
-   -cp refers to the classpath path, which is /sdcard/main.dex here.
-   main refers to the class name.

## References

- Android Software Security and Reverse Engineering
- http://blog.csdn.net/wizardforcel/article/details/54730253
- http://blog.csdn.net/dd864140130/article/details/52076515
