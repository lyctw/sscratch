
This section delves into more advanced topics in C/C++, including pointers, memory management, and object-oriented programming.  Link to node1: [[0x1 - Introduction to C and C++]]

## Pointers in C/C++  

Pointers are a powerful feature in C and C++. They allow you to directly manipulate memory locations.  

### Example: Pointer Usage  

Here's a simple example of using pointers in C:  

```c  
#include <stdio.h>  

int main() {  
    int val = 10;  
    int *ptr = &val;  

    printf("Value: %d\n", *ptr);  
    printf("Address: %p\n", ptr);  

    return 0;  
}  
```


Bye :)