# CS450 Assignment 6 – Symbolic Links: Write-Up

## Overview

This assignment adds symbolic link support to xv6 (MIT x86 public version). The implementation
covers three areas: a new inode type, a new `symlink` system call, and transparent link resolution
inside `sys_open`.

---

## 1. New Inode Type (`stat.h`)

Added `#define T_SYMLINK 4` alongside the existing `T_DIR`, `T_FILE`, and `T_DEV` constants.
This type is stored in the inode's `type` field on disk, so symlinks persist across reboots.
A symlink inode stores its target path as a raw byte string in its data blocks (NUL-terminated,
truncated safely to 127 bytes + NUL).

---

## 2. Syscall Wiring

The `symlink` syscall was plumbed through all four required locations:

| File | Change |
|------|--------|
| `syscall.h` | Added `#define SYS_symlink 22` |
| `syscall.c` | Added `extern int sys_symlink(void)` and `[SYS_symlink] sys_symlink` in the dispatch table |
| `user.h`    | Added `int symlink(const char*, const char*)` prototype for user programs |
| `usys.S`    | Added `SYSCALL(symlink)` macro to generate the trap stub |

---

## 3. `sys_symlink` Implementation (`sysfile.c`)

`sys_symlink(target, linkpath)`:
1. Fetches both string arguments from user space via `argstr`.
2. Calls `create(linkpath, T_SYMLINK, 0, 0)` to allocate a new inode of type `T_SYMLINK`
   and link it into its parent directory — exactly like a regular file but with the symlink type.
3. Writes the target path string into the inode's data blocks using `writei`.
4. Returns 0 on success, -1 on any error.

Because `create` calls `ialloc` + `iupdate` + `dirlink` + `iupdate` all inside a `begin_op/end_op`
transaction, the symlink is durable — it survives a crash or reboot.

---

## 4. Symlink Resolution in `sys_open` (`sysfile.c`)

When `sys_open` reaches a non-O_CREATE path, after `namei` and `ilock`, it now checks whether
the resolved inode is a `T_SYMLINK`. If so, it enters a loop:

```c
for(depth = 0; ip->type == T_SYMLINK && depth < 10; depth++){
    read target from inode data blocks
    release old inode
    namei(target) → new inode
    ilock new inode
}
if(ip->type == T_SYMLINK)  // depth limit hit → cycle
    return -1;
```

Key design decisions:
- **Depth limit of 10**: mirrors the POSIX convention and the assignment spec. After 10 hops,
  `open` returns -1 without panicking, which is what `testsymlink` verifies.
- **Chained links**: each iteration releases the previous inode and resolves the next path from
  scratch via `namei`, so chains of arbitrary shape (A→B→C→file) work correctly.
- **No extra flag needed**: `open` always follows symlinks in this implementation (standard
  behavior). An `O_NOFOLLOW` flag was not required by the spec.

---

## 5. Test Program (`testsymlink.c`)

Three scenarios are verified:

| Test | What it checks |
|------|----------------|
| `symlink created` | `symlink("tgt","lnk")` returns 0 |
| `read through symlink` | `open("lnk", O_RDONLY)` returns the contents of `tgt` ("hello\n") |
| `loop detected` | A↔B cycle causes `open("a")` to fail (returns -1) |

Output when run in xv6:
PASS: symlink created
PASS: read through symlink
PASS: loop detected (open failed)
testsymlink done
$ 