# File Steram Oriented Programming

file 구조체에 대한 이해 필요

특히, \_IO\_FILE 구조체, \_IO\_FILE_plus 구조체, \_IO\_jump\_t 구조체

**중요포인트**

FILE \*fp의 값을 overwrite하여 fake file 구조체를 가리키게 하고 fake vtable을 만들어 이를 실행하는 방식

------

**상세설명**
```
struct _IO_FILE {
  int _flags;           /* High-order word is _IO_MAGIC; rest is flags. */
#define _IO_file_flags _flags

  /* The following pointers correspond to the C++ streambuf protocol. */
  /* Note:  Tk uses the _IO_read_ptr and _IO_read_end fields directly. */
  char* _IO_read_ptr;   /* Current read pointer */
  char* _IO_read_end;   /* End of get area. */
  char* _IO_read_base;  /* Start of putback+get area. */
  char* _IO_write_base; /* Start of put area. */
  char* _IO_write_ptr;  /* Current put pointer. */
  char* _IO_write_end;  /* End of put area. */
  char* _IO_buf_base;   /* Start of reserve area. */
  char* _IO_buf_end;    /* End of reserve area. */
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

struct _IO_FILE_plus
{
  _IO_FILE file;
  const struct _IO_jump_t *vtable;
};

struct _IO_jump_t
{
    JUMP_FIELD(size_t, __dummy);
    JUMP_FIELD(size_t, __dummy2);
    JUMP_FIELD(_IO_finish_t, __finish);
    JUMP_FIELD(_IO_overflow_t, __overflow);
    JUMP_FIELD(_IO_underflow_t, __underflow);
    JUMP_FIELD(_IO_underflow_t, __uflow);
    JUMP_FIELD(_IO_pbackfail_t, __pbackfail);
    /* showmany */
    JUMP_FIELD(_IO_xsputn_t, __xsputn);
    JUMP_FIELD(_IO_xsgetn_t, __xsgetn);
    JUMP_FIELD(_IO_seekoff_t, __seekoff);
    JUMP_FIELD(_IO_seekpos_t, __seekpos);
    JUMP_FIELD(_IO_setbuf_t, __setbuf);
    JUMP_FIELD(_IO_sync_t, __sync);
    JUMP_FIELD(_IO_doallocate_t, __doallocate);
    JUMP_FIELD(_IO_read_t, __read);
    JUMP_FIELD(_IO_write_t, __write);
    JUMP_FIELD(_IO_seek_t, __seek);
    JUMP_FIELD(_IO_close_t, __close);
    JUMP_FIELD(_IO_stat_t, __stat);
    JUMP_FIELD(_IO_showmanyc_t, __showmanyc);
    JUMP_FIELD(_IO_imbue_t, __imbue);
#if 0
    get_column;
    set_column;
#endif
};
```

\_IO_close를 원샷이나 system으로 덮은 뒤 fclose(fp)를 실행하면 익스될지도..

\_IO\_list\_all이라는게 있다. 이것은,

`extern struct _IO_FILE_plus *_IO_list_all;` 이렇게 선언되어 있다.

여기에서 마지막 vtable 부분을 fake vtable로 덮어써버리면 원하는 함수를 실행시킬 수 있다.

ex)House of Orange공격의 경우, `_int_malloc()`함수의 에러 메시지 출력 과정에서 `_IO_overflow`가 호출되는 원리를 이용한다.

ex2) pwnable.tw의 seethefile의 경우 `_IO_close` 또는 `_IO_finish` 함수 포인터를 덮어쓰면 된다.

**중요**

조건이 있다.

- io구조체의 flag부분은 -1 (0xffffffff)로 채워줄 것
- close나 finish함수의 인자값은 fp가 들어가니까 인자 유의할 것
- fake fp구조체의 `_cur_column` (fp + 0x48 (32-bit) or fp + 0x88 (64-bit)) 부분은 0을 가리키는 주소로 세팅해줘야 한다.
- fp + 0x94 (32-bit) or fp + 0xd8 (64-bit) 부분이 vtable이고
- vtable + (4 * 18) 부분에 `_IO_close` 함수가 있다.