## mmap

` void ***mmap**(void *addr, size_t length, int prot, int flags, int fd, off_t offset);`

`mmap` **函数在处理大文件时特别有用，它可以提高文件的访问速度，并且使得文件的读写更加高效。此外，`mmap` 也可以用于创建匿名映射，即不与任何文件关联，仅仅是为了实现进程间通信或者分配一块连续的内存空间。**

- `addr` 参数指定映射区域的首地址，通常设置为0，表示由系统自动选择一个合适的地址。
- `length` 参数指定映射区域的长度，以字节为单位。
- `prot` 参数用于指定映射区域的访问权限，比如可读、可写、可执行等。  PROT_READ | PROT_WRITE
- `flags` 参数用于指定映射区域的属性，比如映射区域是共享的还是私有的。MAP_PRIVATE | MAP_ANONYMOUS
-  `fd` 参数是一个打开的文件描述符，它表示要映射到内存中的文件。
- `offset` 参数指定了文件中的偏移量，表示从文件的哪个位置开始映射数据到内存中。

## mprotect

```
int mprotect(void *addr, size_t len, int prot);
```

> 设置一段内存的读写属性

- `addr`: 指向内存区域起始地址的指针。
- `len`: 内存区域的长度，以字节为单位。
- `prot`: 新的保护属性，可以是以下之一的组合：
  - `PROT_NONE`: 无访问权限。
  - `PROT_READ`: 可读。
  - `PROT_WRITE`: 可写。
  - `PROT_EXEC`: 可执行。

函数返回值为0表示成功，-1表示失败，并设置`errno`以指示错误类型。

## madvise

`int madvise(void *addr, size_t length, int advice);`

`madvise`（Memory Advice）**是一个用于提供对虚拟内存区域的建议的系统调用**。它允许程序向内核提供一些关于内存使用模式的提示，以便内核能够更好地优化内存管理。这个系统调用通常用于优化内存使用，例如通过告知内核某些内存区域可能不再需要，或者可以通过异步操作来提前加载。

- `addr`: 指向内存区域起始地址的指针。
- `length`: 内存区域的长度，以字节为单位。
- `advice`: 提供给内核的建议，可以是以下之一：
  - `MADV_NORMAL`: 默认行为。
  - `MADV_RANDOM`: 表示对内存的随机访问。
  - `MADV_SEQUENTIAL`: 表示对内存的顺序访问。
  - `MADV_WILLNEED`: 表示程序将来会访问这个内存区域，建议内核提前加载。
  - `MADV_DONTNEED`: 表示程序将来可能不再访问这个内存区域，建议内核释放相关的资源。

函数返回值为0表示成功，-1表示失败，并设置`errno`以指示错误类型。

## calloc

`void *calloc(size_t num_elements, size_t element_size);`

`calloc`是 C 语言标准库中的一个函数，用于**动态分配内存空间并将其初始化为零**。与 `malloc` 函数类似，`calloc` 也用于在运行时分配内存。`calloc` 函数将分配 `num_elements * element_size` 个字节的内存空间，并将所有位都设置为零。因此，它适合用于创建数组或结构体，并且可以确保所有元素都初始化为零值。这是与 `malloc` 函数的一个重要区别，`malloc` 函数分配的内存中的初始值是不确定的，可能包含任意值。一般情况下，当需要分配一块内存空间来存储一组特定类型的元素，并且希望所有元素都初始化为零时，可以使用 `calloc` 函数。使用完分配的内存后，应该使用 `free` 函数来释放这块内存，以防止内存泄漏。

- `num_elements` 表示要分配的元素数量。
- `element_size` 表示每个元素的大小。

## signal(SIGINT, int_exit);

```
signal(SIGINT, int_exit);
signal(SIGTERM, int_exit);
signal(SIGABRT, int_exit);
```

- `SIGINT`：这个信号通常是由用户在终端按下 `Ctrl+C` 触发的。默认情况下，它会导致程序终止运行。

- `SIGTERM`：这个信号是在终止进程时发出的通知，通常用于请求进程正常终止。

- `SIGABRT`：这个信号通常是由程序调用 `abort` 函数引发的，用于异常终止程序的执行。

`signal` 函数允许我们为这些信号注册一个自定义的处理函数。在你的代码中，`int_exit` 函数被注册为这三种信号的处理函数。当程序收到这些信号时，将会调用 `int_exit` 函数来处理。

# [FD_CLOEXEC用法及原因](https://www.cnblogs.com/embedded-linux/p/6753617.html)

- python 使用os.system()执行命令多了会报open too manay file 的本质原因

# int clone (int (*__fn) (void *__arg), void *__child_stack,int __flags, void *__arg, ...) __THROW;

- `fn`：是一个指向函数的指针，用于指定新创建进程或线程的执行函数。该函数必须接受一个 `void*` 类型的参数，并返回一个 `int` 类型的值。
- `child_stack`：指向新创建进程或线程栈的指针。栈通常是在内存中预先分配好的，并在 `clone()` 调用时传递给该参数。
- `flags`：用于指定创建进程或线程的行为。它是一个按位或的掩码，可以使用一些标志位来控制创建的进程或线程的行为。常见的标志包括 `CLONE_VM`（共享虚拟内存空间）、`CLONE_FS`（共享文件系统信息）、`CLONE_FILES`（共享文件描述符表）、`CLONE_SIGHAND`（共享信号处理器）、`CLONE_THREAD`（创建线程而不是进程）等。
- `arg`：是一个指针，传递给 `fn` 执行函数的参数。
- `...`：是可选的参数，包括 `ptid`（父线程 ID）、`newtls`（用于设置新线程的线程局部存储）和 `ctid`（子线程 ID）等。

`clone()` 函数通过传递一个函数指针来指定新创建进程或线程的执行函数，这样就可以更加灵活地控制新进程或线程的行为。

```
flag
CLONE_VM：表示新进程与父进程共享虚拟内存空间。
CLONE_FS：表示新进程与父进程共享文件系统信息，如当前工作目录、根目录等。
CLONE_FILES：表示新进程与父进程共享文件描述符表。
CLONE_SIGHAND：表示新进程与父进程共享信号处理器。
CLONE_THREAD：表示创建线程而不是进程。
CLONE_PARENT：表示新进程的父进程是与调用者相同的父进程。
CLONE_CHILD_CLEARTID：表示新进程在退出时清除 ctid 所指向的内存中的内容。
CLONE_DETACHED：表示新线程在退出时不会进入僵死状态，而是直接退出。
```

