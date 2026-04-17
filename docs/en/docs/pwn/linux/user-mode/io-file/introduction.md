# FILE Structure


## Introduction to FILE

FILE is a structure used to describe files in the standard I/O library of Linux systems, known as a file stream.
The FILE structure is created when a program executes functions like fopen, and it is allocated on the heap. We commonly define a pointer to a FILE structure to receive this return value.

The FILE structure is defined in libio.h, as shown below

```c
struct _IO_FILE {
  int _flags;		/* High-order word is _IO_MAGIC; rest is flags. */
#define _IO_file_flags _flags

  /* The following pointers correspond to the C++ streambuf protocol. */
  /* Note:  Tk uses the _IO_read_ptr and _IO_read_end fields directly. */
  char* _IO_read_ptr;	/* Current read pointer */
  char* _IO_read_end;	/* End of get area. */
  char* _IO_read_base;	/* Start of putback+get area. */
  char* _IO_write_base;	/* Start of put area. */
  char* _IO_write_ptr;	/* Current put pointer. */
  char* _IO_write_end;	/* End of put area. */
  char* _IO_buf_base;	/* Start of reserve area. */
  char* _IO_buf_end;	/* End of reserve area. */
  /* The following fields are used to support backing up and undo. */
  char *_IO_save_base; /* Pointer to start of non-current get area. */
  char *_IO_backup_base;  /* Pointer to first valid character of backup area */
  char *_IO_save_end; /* Pointer to end of non-current get area. */

  struct _IO_marker *_markers;

  struct _IO_FILE *_chain;

  int _fileno;
#if 0
  int _blksize;
#else
  int _flags2;
#endif
  _IO_off_t _old_offset; /* This used to be _offset but it's too small.  */

#define __HAVE_COLUMN /* temporary */
  /* 1+column number of pbase(); 0 is unknown. */
  unsigned short _cur_column;
  signed char _vtable_offset;
  char _shortbuf[1];

  /*  char* _save_gptr;  char* _save_egptr; */

  _IO_lock_t *_lock;
#ifdef _IO_USE_OLD_IO_FILE
};
struct _IO_FILE_complete
{
  struct _IO_FILE _file;
#endif
#if defined _G_IO_IO_FILE_VERSION && _G_IO_IO_FILE_VERSION == 0x20001
  _IO_off64_t _offset;
# if defined _LIBC || defined _GLIBCPP_USE_WCHAR_T
  /* Wide character stream stuff.  */
  struct _IO_codecvt *_codecvt;
  struct _IO_wide_data *_wide_data;
  struct _IO_FILE *_freeres_list;
  void *_freeres_buf;
# else
  void *__pad1;
  void *__pad2;
  void *__pad3;
  void *__pad4;

  size_t __pad5;
  int _mode;
  /* Make sure we don't get into trouble again.  */
  char _unused2[15 * sizeof (int) - 4 * sizeof (void *) - sizeof (size_t)];
#endif
};
```

FILE structures within a process are linked together through the _chain field to form a linked list. The head of this linked list is represented by the global variable _IO_list_all, through which we can traverse all FILE structures.

In the standard I/O library, three file streams are automatically opened when a program starts: stdin, stdout, and stderr. Therefore, in the initial state, _IO_list_all points to a linked list composed of these file streams. However, it should be noted that these three file streams reside in the data segment of libc.so, while file streams created using fopen are allocated on the heap.

We can find symbols such as stdin, stdout, and stderr in libc.so. These symbols are pointers to FILE structures, and the actual structure symbols are

```
_IO_2_1_stderr_
_IO_2_1_stdout_
_IO_2_1_stdin_
```


In fact, the _IO_FILE structure is wrapped inside another structure called _IO_FILE_plus, which contains an important pointer vtable that points to a series of function pointers.

In libc 2.23, the vtable offset is 0x94 for 32-bit and 0xd8 for 64-bit.

```
struct _IO_FILE_plus
{
	_IO_FILE    file;
	IO_jump_t   *vtable;
}
```

vtable is a pointer of type IO_jump_t, which stores a set of function pointers. As we will see later, these function pointers are called in various standard I/O functions.

```
void * funcs[] = {
   1 NULL, // "extra word"
   2 NULL, // DUMMY
   3 exit, // finish
   4 NULL, // overflow
   5 NULL, // underflow
   6 NULL, // uflow
   7 NULL, // pbackfail
   
   8 NULL, // xsputn  #printf
   9 NULL, // xsgetn
   10 NULL, // seekoff
   11 NULL, // seekpos
   12 NULL, // setbuf
   13 NULL, // sync
   14 NULL, // doallocate
   15 NULL, // read
   16 NULL, // write
   17 NULL, // seek
   18 pwn,  // close
   19 NULL, // stat
   20 NULL, // showmanyc
   21 NULL, // imbue
};
```



## fread

fread is a standard I/O library function used to read data from a file stream. Its function prototype is as follows

```
size_t fread ( void *buffer, size_t size, size_t count, FILE *stream) ;
```

* buffer: The buffer to store the read data.

* size: Specifies the length of each record.

* count: Specifies the number of records.

* stream: The target file stream.

* Return value: Returns the number of records read into the data buffer.

The code for fread is located in /libio/iofread.c, with the function name _IO_fread, but the actual functionality is implemented in the sub-function _IO_sgetn.

```
_IO_size_t
_IO_fread (buf, size, count, fp)
     void *buf;
     _IO_size_t size;
     _IO_size_t count;
     _IO_FILE *fp;
{
  ...
  bytes_read = _IO_sgetn (fp, (char *) buf, bytes_requested);
  ...
}
```

In the _IO_sgetn function, _IO_XSGETN is called, and _IO_XSGETN is a function pointer in _IO_FILE_plus.vtable. When calling this function, it first retrieves the pointer from vtable and then makes the call.

```
_IO_size_t
_IO_sgetn (fp, data, n)
     _IO_FILE *fp;
     void *data;
     _IO_size_t n;
{
  return _IO_XSGETN (fp, data, n);
}
```

By default, the function pointer points to the _IO_file_xsgetn function.

```
  if (fp->_IO_buf_base
	      && want < (size_t) (fp->_IO_buf_end - fp->_IO_buf_base))
	    {
	      if (__underflow (fp) == EOF)
		break;

	      continue;
	    }
```


## fwrite

fwrite is also a standard I/O library function used to write data to a file stream. Its function prototype is as follows

```
size_t fwrite(const void* buffer, size_t size, size_t count, FILE* stream);
```

* buffer: A pointer to the address of the data to be written (for fwrite).

* size: The number of bytes per data item to write.

* count: The number of data items of size bytes to write.

* stream: The target file pointer.

* Return value: The actual number of data items written (count).


The code for fwrite is located in /libio/iofwrite.c, with the function name _IO_fwrite.
In _IO_fwrite, the main functionality is achieved by calling _IO_XSPUTN to perform the write operation.

Based on the earlier introduction of _IO_FILE_plus, we know that _IO_XSPUTN resides in the vtable of _IO_FILE_plus. Calling this function requires first extracting the pointer from vtable and then jumping to make the call.

```
written = _IO_sputn (fp, (const char *) buf, request);
```

In the default function _IO_new_file_xsputn corresponding to _IO_XSPUTN, it calls _IO_OVERFLOW which is also located in the vtable.

```
 /* Next flush the (full) buffer. */
      if (_IO_OVERFLOW (f, EOF) == EOF)
```


The default function corresponding to _IO_OVERFLOW is _IO_new_file_overflow.

```
if (ch == EOF)
    return _IO_do_write (f, f->_IO_write_base,
			 f->_IO_write_ptr - f->_IO_write_base);
  if (f->_IO_write_ptr == f->_IO_buf_end ) /* Buffer is really full */
    if (_IO_do_flush (f) == EOF)
      return EOF;
```
Inside _IO_new_file_overflow, the system interface write function is ultimately called.



## fopen

fopen is used in the standard I/O library to open files. Its function prototype is as follows

```   
FILE *fopen(char *filename, *type);
```

* filename: The path to the target file.

* type: The type of open mode.

* Return value: Returns a file pointer.

Inside fopen, a FILE structure is created and some initialization operations are performed. Let's look at this process.

First, within the function __fopen_internal corresponding to fopen, the malloc function is called to allocate space for the FILE structure. From this we can learn that the FILE structure is stored on the heap.

```
*new_f = (struct locked_FILE *) malloc (sizeof (struct locked_FILE));
```

After that, the vtable for the created FILE is initialized, and _IO_file_init is called for further initialization.
```
_IO_JUMPS (&new_f->fp) = &_IO_file_jumps;
_IO_file_init (&new_f->fp);
```

In the initialization operation of the _IO_file_init function, _IO_link_in is called to link the newly allocated FILE into the FILE linked list starting from _IO_list_all.
```
void
_IO_link_in (fp)
     struct _IO_FILE_plus *fp;
{
    if ((fp->file._flags & _IO_LINKED) == 0)
    {
      fp->file._flags |= _IO_LINKED;
      fp->file._chain = (_IO_FILE *) _IO_list_all;
      _IO_list_all = fp;
      ++_IO_list_all_stamp;
    }
}
```

After that, the __fopen_internal function calls _IO_file_fopen to open the target file. _IO_file_fopen performs the open operation based on the open mode passed by the user. Ultimately, it calls the system interface open function, which we will not delve into further here.
```
if (_IO_file_fopen ((_IO_FILE *) new_f, filename, mode, is32) != NULL)
    return __fopen_maybe_mmap (&new_f->fp.file);
```

To summarize, the operations of fopen are:

* Allocate the FILE structure using malloc
* Set the vtable for the FILE structure
* Initialize the allocated FILE structure
* Link the initialized FILE structure into the FILE structure linked list
* Call the system call to open the file

## fclose

fclose is a function in the standard I/O library used to close an opened file. Its purpose is the opposite of fopen.

```
int fclose(FILE *stream)
```
Functionality: Closes a file stream. Using fclose flushes the remaining data in the buffer to the disk file, and releases the file pointer and associated buffers.



fclose first calls _IO_unlink_it to unlink the specified FILE from the _chain linked list.

```
if (fp->_IO_file_flags & _IO_IS_FILEBUF)
    _IO_un_link ((struct _IO_FILE_plus *) fp);
```

Then it calls the _IO_file_close_it function, which calls the system interface close to close the file.

```
if (fp->_IO_file_flags & _IO_IS_FILEBUF)
    status = _IO_file_close_it (fp);
```

Finally, it calls _IO_FINISH from the vtable, which corresponds to the _IO_file_finish function. This function calls free to release the previously allocated FILE structure.

```
_IO_FINISH (fp);
```


## printf/puts

printf and puts are commonly used output functions. When the argument to printf is a plain string ending with '\n', printf is optimized to the puts function with the newline character removed.

The function implementing puts in the source code is _IO_puts. The operation of this function is roughly the same as the fwrite flow. Internally, it also calls _IO_sputn from the vtable, which results in executing _IO_new_file_xsputn, and ultimately calls the system interface write function.

The call stack backtrace for printf is as follows, also implemented through _IO_file_xsputn:

```
vfprintf+11
_IO_file_xsputn
_IO_file_overflow
funlockfile
_IO_file_write
write
```

