Radare2 automatically parsing available imports and exports sections in the binary,
but moreover it can load additional debugging information, if presented.
Two main formats are supported: DWARF and PDB (for Windows binaries).
Note, that unlike many tools radare2 doesn't rely on Windows API to parse
PDB files, thus they can be loaded on any other supported platform - e.g.
Linux or OS X.

DWARF debuginfo loads automatically by default, because usually it's stored
right in the executable file. PDB is a bit of different beast - it is always
stored as a separate binary, thus the different logic of handling it.

At first, one of the common scenarios is to analyse the file from Windows distribution.
In this case all PDB files are available in the Microsoft server, which is by default
is in options. See all pdb options in radare2:
```
pdb.autoload = 0
pdb.extract = 1
pdb.server = https://msdl.microsoft.com/download/symbols
pdb.useragent = Microsoft-Symbol-Server/6.11.0001.402
```
Using the variable `pdb.server` you can change the address where radare2 will try to
download the PDB file by the GUID stored in executable header.
Usually there is no reason to change default `pdb.useragent`, but who knows where
it could be handy?

Because those PDB files are stored as "cab" archives on the server, `pdb.extract=1`
says to automatically extract them.

Note, that for automatical downloading to work you need "cabextract" tool, and
wget/curl installed.

Sometimes you don't need to do that from the radare2 itself, thus - two handy
rabin2 options:
```
 -P              show debug/pdb information
 -PP             download pdb file for binary
```
where `-PP` automatically downloads the pdb for the selected binary, using those
`pdb.*` config options. `-P` will dump the contents of the PDB file, which is useful
sometimes for a quick understanding of the symbols stored in it.

Apart from the basic scenario of just opening file, PDB information can be additionally
manipulated by the `id` commands:
```
[0x000051c0]> id?
|Usage: id Debug information
| Output mode:
| '*'              Output in radare commands
| id               Source lines
| idp [file.pdb]   Load pdb file information
| idpi [file.pdb]  Show pdb file information
| idpd             Download pdb file on remote server
```
Where `idpi` is basically the same as `rabin2 -P`.
Note, that `idp` can be also used not only in the static analysis mode, but also
in the debugging mode, even if connected via WinDbg.

For simplifying the loading PDBs, especially for the processes with many linked DLLs,
radare2 can autoload all required PDBs automatically - you need just set the
`e pdb.autoload=true` option. Then if you load some file in debugging mode
in Windows, using `r2 -d file.exe` or `r2 -d 2345` (attach to pid 2345), all
related PDB files will be loaded automatically.

DWARF information loading on the other hand is completely automated, you don't
need to run any commands/change any options:
```
r2 `which rabin2`
[0x00002437 8% 300 /usr/local/bin/rabin2]> pd $r
       ┌──< 0x00002437      752f           jne 0x2468                  ;[1]
       │⁝   0x00002439      48833d9f5b20.  cmp qword reloc.__cxa_finalize_224, 0
       │⁝   0x00002441      55             push rbp
       │⁝   0x00002442      4889e5         mov rbp, rsp
      ┌───< 0x00002445      740c           je 0x2453                   ;[2]
      ││⁝   0x00002447      488d3df25720.  lea rdi, obj.__dso_handle   ; 0x207c40 ; "@| "
      ││⁝   0x0000244e      e80dffffff     call 0x2360                 ;[3]
      └───> 0x00002453      e848ffffff     call sym.deregister_tm_clones ;[4]
       │⁝   0x00002458      c605915e2000.  mov byte [obj.completed.6991], 1 ; obj.__TMC_END__ ; [0x2082f0:1]=0
       │⁝   0x0000245f      5d             pop rbp
       │⁝   0x00002460      c3             ret
       │⁝   0x00002461      0f1f80000000.  nop dword [rax]
       └──> 0x00002468      f3c3           ret
        ⁝   0x0000246a      660f1f440000   nop word [rax + rax]
        ⁝   ;-- entry1.init:
        ⁝   ;-- frame_dummy:
        ⁝   0x00002470      55             push rbp
        ⁝   0x00002471      4889e5         mov rbp, rsp
        ⁝   0x00002474      5d             pop rbp
        └─< 0x00002475      e966ffffff     jmp sym.register_tm_clones  ;[5]
            ;-- blob_version:
            0x0000247a      55             push rbp                    ; ../blob/version.c:18
            0x0000247b      4889e5         mov rbp, rsp
            0x0000247e      4883ec10       sub rsp, 0x10
            0x00002482      48897df8       mov qword [rbp - 8], rdi
            0x00002486      b832000000     mov eax, 0x32               ; ../blob/version.c:24 ; '2'
            0x0000248b      84c0           test al, al                 ; ../blob/version.c:19
        ┌─< 0x0000248d      7409           je 0x2498                   ;[6]
        │   0x0000248f      488d05223c00.  lea rax, str.2.0.1_182_gf1aa3aa4d ; 0x60b8 ; "2.0.1-182-gf1aa3aa4d"
       ┌──< 0x00002496      eb07           jmp 0x249f                  ;[7]
       │└─> 0x00002498      488d052e3c00.  lea rax, 0x000060cd
       └──> 0x0000249f      488b75f8       mov rsi, qword [rbp - 8]
            0x000024a3      4989c0         mov r8, rax
            0x000024a6      b940000000     mov ecx, 0x40               ; section_end.ehdr
            0x000024ab      bac0400000     mov edx, 0x40c0
            0x000024b0      488d3d193c00.  lea rdi, str._s_2.1.0_git__d___linux_x86__d_git._s_n ; 0x60d0 ; "%s 2.1.0-git %d @ linux-x86-%d git.%s\n"
            0x000024b7      b800000000     mov eax, 0
            0x000024bc      e88ffeffff     call 0x2350                 ;[8]
            0x000024c1      b866000000     mov eax, 0x66               ; ../blob/version.c:25 ; 'f'
            0x000024c6      84c0           test al, al
        ┌─< 0x000024c8      740c           je 0x24d6                   ;[9]
        │   0x000024ca      488d3d273c00.  lea rdi, str.commit:_f1aa3aa4d2599c1ad60e3ecbe5f4d8261b282385_build:_2017_11_06__12:18:39 ; ../blob/version.c:26 ; 0x60f8 ; "commit: f1aa3aa4d2599c1ad60e3ecbe5f4d8261b282385 build: 2017-11-06__1
        │   0x000024d1      e8daf9ffff     call sym.imp.puts           ;[?]
        └─> 0x000024d6      b800000000     mov eax, 0                  ; ../blob/version.c:28
            0x000024db      c9             leave                       ; ../blob/version.c:29
            0x000024dc      c3             ret
            ;-- rabin_show_help:
            0x000024dd      55             push rbp                    ; .//rabin2.c:27
```
As you can see it loads function names and source line information.

