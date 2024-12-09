# 14.1 Types of Memory
Two types of memory that are allocated. The first is called **stack** memory, and allocations and deallocations of it are managed *implicitly* by the compiler for you.

When you return from the function, the compiler deallocates the memory for you;

It is this need for long-lived memory that gets us to the second type of memory, called **heap** memory, where all allocations and deallocations are *explicitly* handled by you.

```c
void func(){
	int *x = (int *) malloc(sizeof(int));
}
```

First, you might notice that both stack and heap allocation occur on this line: first the compiler knows to make room for a pointer to an integer when it sees your declaration of said pointer (int \*x); subsequently, when the program calls `malloc()`, it requests space for an integer on the heap; the routine returns the address of such an integer (upon success, or NULL on failure), which is then stored on the stack for use by the program.

# 14.2 The `malloc()` Call
```c
#include <stdlib.h>

void *malloc(size_t size);
```

The single parameter `malloc()` takes is of type `size_t` which simply describes how many bytes you need. 

# 14.4 Common Errors
Correct memory management has been such a problem, in fact, that many newer languages have support for **automatic memory management**. In such languages, while you call something akin to malloc() to allocate memory (usually new or something similar to allocate a new object), you never have to call something to free space; rather, a **garbage collector** runs and figures out what memory you no longer have references to and frees it for you

## Forgetting to allocate memory
For example, the routine `strcpy(dst, src)` copies a string from a source pointer to a destination pointer. However, if you are not careful, you might do this:
![[Pasted image 20241126180204.png]]
When you run this code, it will likely lead to a **segmentation fault**, in this case, the proper code might instead look like this:
![[Pasted image 20241126180401.png]]
这里多分配一个字节是为了 end-of-string character.

Alternately, you could use strdup() and make your life easier.

## Not allocating enough memory
Buffer overflow.
![[Pasted image 20241126180935.png]]
Oddly enough

## Forgetting to initialize allocated memory

## Forgetting to free memory
Another common error is known as a memory leak.

In some cases, it may seem like not calling `free()` is reasonable. For example, your program is short-lived, and will soon exit; in this case, when the process dies, the OS will clean up all of its allocated pages and thus no memory leak will take place per se.

## Freeing memory before you are done with it
such a mistake is called a `dangling pointer`

## Freeing Memory Repeatedly
Double free. The result of doing so is undefined. 

## Calling `free()` Incorrectly

## Summary
purify and valgrind; both are excellent at helping you locate the source of your memory-related problems.

# 14.5 Underlying OS Support
One such system call is called `brk`, which is used to change the location of the program's break: the location of the end of the heap.

Finally, you can also obtain memory from the operating system via the `mmap()` call. By passing in the correct arguments, `mmap()` can create an anonymous memory region within your program --- a region which is not associated with any particular file but rather with **swap space**. This memory can then also be treated like a heap and managed as such.

# 14.6 Other Calls
There are a few other calls that the memory-allocation library support. `calloc()` allocates memory and also zeroes it before returning; The routine `realloc()` can also be useful, when you've allocated space for something (say, an array), and then need to add something to it.

A memory-bug detector called `valgrind`