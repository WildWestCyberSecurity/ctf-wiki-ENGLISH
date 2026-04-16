# DEX File

## Basic Introduction

Google designed a dedicated executable file format called DEX (Dalvik eXecutable File) for Java code in Android, suitable for mobile platforms with low memory and limited processor performance such as smartphones. Below, we will mainly introduce the format of DEX files.

## DEX File Format

### Data Type Definitions

Before introducing the specific structure of the DEX file, let's first look at some basic data types used in the DEX file.

| Name        | Description                         |
| --------- | -------------------------- |
| byte      | 8-bit signed integer                   |
| ubyte     | 8-bit unsigned integer                   |
| short     | 16-bit signed integer, little-endian byte order          |
| ushort    | 16-bit unsigned integer, little-endian byte order          |
| int       | 32-bit signed integer, little-endian byte order          |
| uint      | 32-bit unsigned integer, little-endian byte order          |
| long      | 64-bit signed integer, little-endian byte order          |
| ulong     | 64-bit unsigned integer, little-endian byte order          |
| sleb128   | Signed LEB128, variable length (see below)       |
| uleb128   | Unsigned LEB128, variable length (see below)       |
| uleb128p1 | Unsigned LEB128 plus `1`, variable length (see below) |

The reason for using variable-length data types is to minimize the space occupied by the executable file. For example, if the length of a string is 5, we actually only need one byte. However, we don't want to directly use `u1` to define the corresponding type, because that would limit all string lengths to the corresponding range.

Variable-length types are all based on the LEB128 (Little-Endian Base) type, which can represent 32-bit int numbers and selects an appropriate length based on the size of the number to be represented. As shown in the figure below, the highest bit of each byte indicates whether the next byte is used: 1 means used, 0 means not used. Therefore, each byte actually only has 7 effective bits to represent the corresponding number. If a LEB128 type variable uses 5 bytes and the highest bit of the fifth byte is 1, that indicates a problem.

![](./figure/leb128.png)

The function for reading unsigned LEB128 types in Dalvik is as follows:

```c++
DEX_INLINE int readUnsignedLeb128(const u1** pStream) {
    const u1* ptr = *pStream;
    int result = *(ptr++);      // Get the first byte
    if (result > 0x7f) {        // If the first byte is greater than 0x7f, it means the highest bit of the first byte is 1
        int cur = *(ptr++);     // Second byte
        result = (result & 0x7f) | ((cur & 0x7f) << 7); // First two bytes
        if (cur > 0x7f) {
            cur = *(ptr++);
            result |= (cur & 0x7f) << 14;
            if (cur > 0x7f) {
                cur = *(ptr++);
                result |= (cur & 0x7f) << 21;
                if (cur > 0x7f) {
                    /*
                     * Note: We don't check to see if cur is out of
                     * range here, meaning we tolerate garbage in the
                     * high four-order bits.
                     */
                    cur = *(ptr++);
                    result |= cur << 28;
                }
            }
        }
    }
    *pStream = ptr;
    return result;
}
```

For example, suppose we want to calculate the uleb128 value of c0 83 92 25, as follows:

- The highest bit of the first byte is 1, so there is a second byte. result1 = 0xc0 & 0x7f = 0x40
- Similarly, the result for the second byte is result2 = (0x83 & 0x7f) << 7 = 0x180
- The result for the third byte is result3 = (0x92 & 0x7f) << 14 = 0x48000
- The result for the fourth byte is result4 = (0x25) << 21 = 0x4a00000
- The value corresponding to this byte stream is result1 + result2 + result3 + result4 = 0x4a481c0

The function for reading signed LEB128 numbers in Dalvik is as follows:

```c++
 DEX_INLINE int readSignedLeb128(const u1** pStream) {
    const u1* ptr = *pStream;
    int result = *(ptr++);
    if (result <= 0x7f) {
        result = (result << 25) >> 25;   // Sign extension
    } else {
        int cur = *(ptr++);
        result = (result & 0x7f) | ((cur & 0x7f) << 7);
        if (cur <= 0x7f) {
            result = (result << 18) >> 18; // Sign extension
        } else {
            cur = *(ptr++);
            result |= (cur & 0x7f) << 14; // Sign extension
            if (cur <= 0x7f) {
                result = (result << 11) >> 11; // Sign extension
            } else {
                cur = *(ptr++);
                result |= (cur & 0x7f) << 21;
                if (cur <= 0x7f) {
                    result = (result << 4) >> 4;  // Sign extension
                } else {
                    /*
                     * Note: We don't check to see if cur is out of
                     * range here, meaning we tolerate garbage in the
                     * high four-order bits.
                     */
                    cur = *(ptr++);
                    result |= cur << 28;
                }
            }
        }
    }
    *pStream = ptr;
    return result;
}
```

For example, suppose we want to calculate the sleb128 value of d1 c2 b3 40, the calculation process is as follows:

- result1 = 0xd1 & 0x7f = 0x51
- result2 = (0xc2 & 0x7f) << 7 = 0x21000
- result3 = (0xb3 & 0x7f) << 14 = 0xcc000
- result4 = (0x40) << 21 = 0x8000000
- The final result (r1 + r2 + r3 + r4) << 4 >> 4 = 0xf80ce151


The uleb128p1 type is mainly used to represent unsigned numbers, and is applicable to the following scenarios:

- The number representation must be non-negative.
- When the number is 0xffffffff, adding 1 to it gives 0, in which case we only need 1 byte.
- **This requires further consideration.**

### DEX File Overview

The overall structure of a DEX file is as follows:

![](./figure/dex_structure.png)

It mainly consists of three parts:

- File header, which provides the basic attributes of the DEX file.
- Index section, which provides indexes to related data; the actual data is stored in the data section.
- Data section, which stores the actual strings and code.

### DEX File Header

The DEX file header mainly contains the magic field, adler32 checksum, SHA-1 hash value, the number of string_ids and their offset addresses, etc. It occupies a fixed 0x70 bytes, and the data structure is as follows:

```c++
struct DexHeader {
    u1  magic[8];           /* includes version number */
    u4  checksum;           /* adler32 checksum */
    u1  signature[kSHA1DigestLen]; /* SHA-1 hash */
    u4  fileSize;           /* length of entire file */
    u4  headerSize;         /* offset to start of next section */
    u4  endianTag;
    u4  linkSize;
    u4  linkOff;
    u4  mapOff;
    u4  stringIdsSize;
    u4  stringIdsOff;
    u4  typeIdsSize;
    u4  typeIdsOff;
    u4  protoIdsSize;
    u4  protoIdsOff;
    u4  fieldIdsSize;
    u4  fieldIdsOff;
    u4  methodIdsSize;
    u4  methodIdsOff;
    u4  classDefsSize;
    u4  classDefsOff;
    u4  dataSize;
    u4  dataOff;
};
```

The specific descriptions are as follows:

| Name              | Format                        | Description                                       |
| --------------- | ------------------------- | ---------------------------------------- |
| magic           | ubyte[8] = DEX_FILE_MAGIC | Identifies the DEX file, where DEX_FILE_MAGIC = "dex\n035\0"   |
| checksum        | uint                      | adler32 checksum of the remaining contents of the file except `magic` and this field, used to detect file corruption |
| signature       | ubyte[20]                 | SHA-1 signature (hash) of the contents of the file except `magic`, `checksum`, and this field, used to uniquely identify the file |
| file_size       | uint                      | Size of the entire file (including the header), in bytes                    |
| header_size     | uint = 0x70               | Size of the header, in bytes                           |
| endian_tag      | uint = ENDIAN_CONSTANT    | Byte order tag, big-endian or little-endian                          |
| link_size       | uint                      | If this file has not been statically linked, this value is `0`; otherwise it is the size of the link section        |
| link_off        | uint                      | If `link_size == 0`, this value is `0`; otherwise, it is the offset from the beginning of the file to the `link_data` section |
| map_off         | uint                      | This offset must be non-zero, indicating the offset from the beginning of the file to the `data` section         |
| string_ids_size | uint                      | Number of strings in the string identifiers list                          |
| string_ids_off  | uint                      | If `string_ids_size == 0` (which is admittedly a strange edge case), this value is `0`; otherwise it represents the offset from the beginning of the file to `string_ids` |
| type_ids_size   | uint                      | Number of elements in the type identifiers list, maximum 65535                  |
| type_ids_off    | uint                      | If `type_ids_size == 0` (which is admittedly a strange edge case), this value is `0`; otherwise it represents the offset from the beginning of the file to the start of the `type_ids` section |
| proto_ids_size  | uint                      | Number of elements in the prototype (method) identifiers list, maximum 65535              |
| proto_ids_off   | uint                      | If `proto_ids_size == 0` (which is admittedly a strange edge case), this value is `0`; otherwise it represents the offset from the beginning of the file to the start of the `proto_ids` section |
| field_ids_size  | uint                      | Number of elements in the field identifiers list                            |
| field_ids_off   | uint                      | If `field_ids_size == 0`, this value is `0`; otherwise it represents the offset from the beginning of the file to the start of the `field_ids` section |
| method_ids_size | uint                      | Number of elements in the method identifiers list                            |
| method_ids_off  | uint                      | If `method_ids_size == 0`, this value is `0`; otherwise it represents the offset from the beginning of the file to the start of the `method_ids` section |
| class_defs_size | uint                      | Number of elements in the class definitions list                              |
| class_defs_off  | uint                      | If `class_defs_size == 0` (which is admittedly a strange edge case), this value is `0`; otherwise it represents the offset from the beginning of the file to the start of the `class_defs` section |
| data_size       | uint                      | Size of the `data` section in bytes, must be an even multiple of sizeof(uint), indicating 8-byte alignment |
| data_off        | uint                      | Offset from the beginning of the file to the start of the `data` section                  |

### DEX Index Section

#### string id

The StringIds section contains `stringIdsSize` `DexStringId` structures, with the following structure:

```c++
struct DexStringId {
    u4 stringDataOff;   /* String data offset, i.e., the file offset of each StringData in the data section */
};
```

As we can see, what is stored in DexStringId is just the relative offset of each string. Additionally, each offset occupies 4 bytes, so the string section occupies a total of 4 * stringIdsSize bytes.

At the corresponding offset, strings are stored in MUTF-8 format. The beginning stores a LEB128 type variable as mentioned earlier, representing the length of the string, followed immediately by the string itself, and ending with \x00. The string length does not include \x00.

#### type id

The type_ids section indexes all types (classes, arrays, or primitive types) used in Java code. This list must be sorted by `string_id` index and must not contain duplicates.

```c++
struct DexTypeId {
    u4 descriptorIdx;    /* Index into the DexStringId list */
};
```

#### proto Id

The proto id field is mainly designed for Java method prototypes, and mainly contains the return type and parameter list of a method declaration, without involving the method name. It mainly contains the following three data structures:

```c++
struct DexProtoId {
    u4 shortyIdx;       /* Return type + parameter types, shorthand, index into the DexStringId list */
    u4 returnTypeIdx;   /* Return type, index into the DexTypeId list */
    u4 parametersOff;   /* Parameter types, offset to DexTypeList */
}

struct DexTypeList {
    u4 size;             /* Number of DexTypeItems, i.e., number of parameters */
    DexTypeItem list[1]; /* Points to the start of DexTypeItem */
};

struct DexTypeItem {
    u2 typeIdx;           /* Parameter type, index into the DexTypeId list, ultimately pointing to the string index */
};
```

#### field id

The field id section is mainly designed for the fields of each class in Java, and involves the following data structures:

```c++
struct DexFieldId {
    u2 classIdx;   /* Class type, index into the DexTypeId list */
    u2 typeIdx;    /* Field type, index into the DexTypeId list */
    u4 nameIdx;    /* Field name, index into the DexStringId list */
};
```

#### method id

The method id section is designed directly for methods in Java, and contains the class the method belongs to, the method prototype, and the method name.

```c++
struct DexMethodId {
    u2 classIdx;  /* Class type, index into the DexTypeId list */
    u2 protoIdx;  /* Declaration type, index into the DexProtoId list */
    u4 nameIdx;   /* Method name, index into the DexStringId list */
};
```



#### class def

classDefsSize indicates the size of the class def section, and classDefsOff indicates the offset of the class def section.

This section is designed for classes in Java, and contains the following data structures with the relevant information shown below:

```c++
// Basic class information
struct DexClassDef {
    u4 classIdx;    /* Class type, index into the DexTypeId list */
    u4 accessFlags; /* Access flags */
    u4 superclassIdx;  /* Superclass type, index into the DexTypeId list */
    u4 interfacesOff; /* Interfaces, offset to DexTypeList */
    u4 sourceFileIdx; /* Source file name, index into the DexStringId list */
    u4 annotationsOff; /* Annotations, points to DexAnnotationsDirectoryItem structure */
    u4 classDataOff;   /* Offset to DexClassData structure */
    u4 staticValuesOff;  /* Offset to DexEncodedArray structure */
};

// Overview of class fields and methods
struct DexClassData {
    DexClassDataHeader header; /* Specifies the number of fields and methods */
    DexField* staticFields;    /* Static fields, DexField structure */
    DexField* instanceFields;  /* Instance fields, DexField structure */
    DexMethod* directMethods;  /* Direct methods, DexMethod structure */
    DexMethod* virtualMethods; /* Virtual methods, DexMethod structure */

// Detailed description of the number of class fields and methods
struct DexClassDataHeader {
    u4 staticFieldsSize;  /* Number of static fields */
    u4 instanceFieldsSize; /* Number of instance fields */
    u4 directMethodsSize;  /* Number of direct methods */
    u4 virtualMethodsSize; /* Number of virtual methods */
};

// Field definition
struct DexField {
    u4 fieldIdx;    /* Index into DexFieldId */
    u4 accessFlags; /* Access flags */
};

// Method definition
struct DexMethod {
    u4 methodIdx;   /* Index into DexMethodId */
    u4 accessFlags; /* Access flags */
    u4 codeOff;     /* Offset to DexCode structure */
};

// Code overview
struct DexCode {
    u2 registersSize;   /* Number of registers used */
    u2 insSize;         /* Number of parameters */
    u2 outsSize;        /* Number of registers used by other methods when called, allocated on the caller's stack and pushed (speculated) */
    u2 triesSize;       /* Number of Try/Catch blocks */
    u4 debugInfoOff;    /* Offset to debug information */
    u4 insnsSize;       /* Number of instructions, in 2-byte units */
    u2 insns[1];        /* Instruction set */
};
```

#### Summary

As we can see, the indexing in the index section is quite complex, but also quite clever. Here is an example given by the Dalvik designers in their presentation at [Google Developer Day 2008 China](https://sites.google.com/site/developerdaychina/).

![](./figure/dex_structure_designer.png)

### DEX Data Section

This is where all the various data mentioned above is stored.

### DEX Map Section

The mapOff field in DexHeader provides the offset of the DexMapList structure in the DEX file. After the Dalvik virtual machine parses the contents of the DEX file, it maps the contents to the DexMapList data structure. It can be said that this structure describes the overall picture of the corresponding DEX file. The specific code is as follows:

```c++
struct DexMapList {
    u4 size;               /* Number of DexMapItems, for convenient parsing */
    DexMapItem list[1];    /* Points to DexMapItem */
};

struct DexMapItem {
    u2 type;      /* Type starting with kDexType */
    u2 unused;    /* Unused, for byte alignment */
    u4 size;      /* Specifies the number of the corresponding type */
    u4 offset;    /* Specifies the file offset of the corresponding type's data */
};

/* The type field is an enumeration constant; the specific type can be easily determined from the type name. */
/* map item type codes */
enum {
    kDexTypeHeaderItem               = 0x0000,
    kDexTypeStringIdItem             = 0x0001,
    kDexTypeTypeIdItem               = 0x0002,
    kDexTypeProtoIdItem              = 0x0003,
    kDexTypeFieldIdItem              = 0x0004,
    kDexTypeMethodIdItem             = 0x0005,
    kDexTypeClassDefItem             = 0x0006,
    kDexTypeMapList                  = 0x1000,
    kDexTypeTypeList                 = 0x1001,
    kDexTypeAnnotationSetRefList     = 0x1002,
    kDexTypeAnnotationSetItem        = 0x1003,
    kDexTypeClassDataItem            = 0x2000,
    kDexTypeCodeItem                 = 0x2001,
    kDexTypeStringDataItem           = 0x2002,
    kDexTypeDebugInfoItem            = 0x2003,
    kDexTypeAnnotationItem           = 0x2004,
    kDexTypeEncodedArrayItem         = 0x2005,
    kDexTypeAnnotationsDirectoryItem = 0x2006,
};
```

## DEX Example

You can find an APK yourself, then use the 010 Editor template to parse it and see the corresponding results.

## References

- Android Software Security and Reverse Engineering
