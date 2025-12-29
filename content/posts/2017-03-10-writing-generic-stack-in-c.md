---
title: Writing Generic stack in C
date: '2017-03-10 08:20:00'
author: Jiten Thakkar
tags:
- C
- generic
- stack
- pointers
- malloc
- realloc
comments: true
---

Object oriented languages like C++, Java etc usually support generic programming (templates in C++ and generics in Java). So it becomes easy to write a generic library in those languages. For a language like C, it is slightly complicated to write such library because it doesn’t have concept of classes and doesn’t support generic programming right off the bat. But not to worry. It’s doable. It just requires some knowledge of heap memory and magical powers of void pointers. So let’s design a generic stack.

First, let’s define a structure which would have the data of the stack and all the book keeping information like reference to last element and total elements.

    Stack {
      data  ==> field that holds all the data
      topElement ==> field that point to the top member
      totalCapacity ==> total capacity of the stack, useful to check is the stack is full or not
    }

The data field here needs to support any type of data (basic and user defined) for which we want to make a stack. Here the void pointers come into picture. Before we go into how void pointer would be used, let’s understand how memory access in C works from the point of view of types.

When you declare a pointer like `int *a` in C, it just says that the data pointed to by the pointer is of type integer. This information is used by the compiler when the code accesses the data in that memory location. It tells the compiler how many bytes of data needs to be read while accessing the memory pointed to by a pointer of type int. Suppose size of an integer is 4 bytes, then it will read 4 bytes of data while accessing the pointer. This tells us that regardless of the type of data that is stored at the destination, the number of bytes that are read relies on the type of the pointer that points to that location. So if we typecast the same pointer that was pointing to `a` to `double*`, then it would now read 8 bytes of data when accessed(assuming that double is of 8 bytes in the system). When  you access data using a `void` pointer, the compiler gives an error because it doesn’t know how much data needs to be read because there is no type. But if you typecast the same `void` pointer to an integer then it would treat the underlying data as an integer and if you typecast it to a `double` then it would treat the data as `double`.

Well, that’s where the trick lies for writing a generic data structure in C. Disguise the data as void when storing them and when you want to read it, just read the necessary number of bytes based on the type of the data that is being stored. Based on the above explaination, we can now define the data structure for stack as follow:

    typedef struct {
      void* data;
      int top;
      int totalElements;
      int memberSize;
    } Stack;

Here, first three members are as explained in the structure above. The `data` field is declared as `void*` so that it can point to any type of data. The fourth member `memberSize` has the size in number of bytes of one member of the type for which we are writing the Stack. As explained, we will use this field about member size while accessing the data stored in `data` field. So to conclude, the Stack which is represented as a structure consists of `memberSize` to know the size of an element store in the data, `totalElements` to know the total capacity of the Stack and `top` which tells the index of the member that is on the top of the stack.

Now that we have figured out the structure of the generic Stack, let’s write the functionalities for the stack. We will write function create stack, push, top, pop and destroy.

First createStack. This one is straight forward. The function will take initial size of the Stack and size of the member element. We will use these two to calculate total memory required for the stack and allocate it. We will initialize top to -1 which means the stack is empty. So, the function would be something like this:

    Stack* createStack(int memberSize, int totalElements) {
      Stack *s = malloc(sizeof(Stack));
      s->top = -1;
      s->memberSize = memberSize;
      s->totalElements = totalElements;
      s->data = malloc(totalElements*memberSize);
      return s;
    }

We are allocating total of `totalElements*memberSize` bytes of data for the stack using [malloc](http://www.cplusplus.com/reference/cstdlib/malloc/) function. This means if we are creating a stack  of size 10 for type `int`, then assuming the size of `int` is 4 bytes, we will allocate 40 bytes of data for the stack. You can think of field `data` as an array of 40 elements of one byte each. Now, depending on how we access this data would define how to treat these 40 bytes. If we typecast the `data` field to `int*` then the compiler would treat it as an array of 10 integers. So if we do something like `int* x = &(int*)s->data[1]` then pointer `x` would point to the 5th byte of the data array(since arrays in C start with index 0 so accessing `[1]`  means accessing second element). Whereas if we type cast it to `char*` (and assuming size of `char` is 1 byte), the compiler would treat it as an array of 40 elements. And `char* x = &(char*)s->data[1]` would point to second byte of the data array. It’s just a game of pointer arithmetics. Pretty fucked up huh! They don’t call C type unsafe for no reason.

Next, stackPush: this is important one. Here we will use all the pointer magic to save data being pushed to the stack. Let’s walk through the code below:

    int stackPush(Stack *s,  void *element) {
      //check is the stack is full
      if (s->top == s->totalElements - 1) {
        //if full, return 1 which would signal that the operation failed
        return 1;
      }
      s->top++;
      //calculate starting location for the new element
      void* target = (char*)s->data+(s->top*s->memberSize);
      memcpy(target, element, s->memberSize);
      return 0;
    }

Here, the `data` parameter contains the data that needs to be pushed to the stack. First we check if the stack if full. If it’s full then we just return 1 which would tell the caller that the call to push failed. (Later we will fix this by dynamically expanding the capacity of the stack). Now, we need to copy the data in the `element` parameter to the array in the Stack. For that we calculate the start index of the next member by first typecasting it to `char*` and then adding total bytes taken by members already in the stack. By typecasting `data` to `char*`, we are treating the data as an array of elements of size 1. This way the arithmetic to calculate target index using `memberSize` works out well. Lastly we copy `memberSize` bytes of the data into the `data` array using [memcpy](http://www.cplusplus.com/reference/cstring/memcpy).

We are returning 1 whenever the stack becomes full. If you have used a generic container library like Stack in any other language, you would have noticed that you don’t have to worry about the capacity of the container. So, what if we just expand the capacity of the stack dynamically when we run out of memory? That would be cool. And doing that is also very easy thanks to [realloc](http://www.cplusplus.com/reference/cstdlib/realloc/). This function takes pointer to the memory that was allocated dynamically and new size and expands the size of the allocated memory. Under the hood, it calls `malloc` to allocate memory of new size and calls [memcpy](http://www.cplusplus.com/reference/cstring/memcpy) to copy data from old location to the new location and returns the pointer to the new location (hey, may be you can write your own very simple `realloc`). Let’s write function expandStack:

    int expandStack(Stack* s) {
      //double total capacity of the stack
      s->data = realloc(s->data, s->totalElements * 2);
      s->totalElements *= 2;
      return 0;
    }

    int stackPush(Stack *s,  void *data) {
      //check is the stack is full
      if (s->top == s->totalElements - 1) {
        //if full, call expand function to expand the size of the stack
        expandStack(s);
      }
      s->top++;
      //calculate starting location for the new element
      void* target = (char*)s->data+(s->top*s->memberSize);
      memcpy(target, data, s->memberSize);
      return 0;
    }

We will just double the size of the stack whenever we hit the capacity. And we have our stack with unlimited capacity(unlimited in theory of course).

Next, stackPop: This function is same as stackPush just in reverse direction. Here we first check if the stack is empty. If it is then we return 1 which would tell the caller that the operation failed because the stack is empty. Else we will calculate the start index of the top element and copy that data to target pointer being passed as parameter. Here is the function:

    int stackPop(Stack *s,  void *target) {
      if (s->top == -1) {
        return 1;
      }
      void* source = (char*)s->data+(s->top*s->memberSize);
      s->top--;
      memcpy(target, source, s->memberSize);
      return 0;
    }

Next, stackTop: exactly same as stackPop function. The only difference is that we won’t reduce the `top` index value because we are just reading the top member. Here is the function:

    int stackTop(Stack *s,  void *target) {
      if (s->top == -1) {
        return 1;
      }
      void* source = (char*)s->data+(s->top*s->memberSize);
      memcpy(target, source, s->memberSize);
      return 0;
    }


Next stackDestroy: this one is also straight forward. Free all the dynamically allocated memory and you are done. Here is the function:

    int stackDestroy(Stack *s) {
      free(s->data);
      free(s);
      return 0;
    }

Cool, we just made our a generic stack with some basic functionality. You can write other generic containers like [queue](http://www.cplusplus.com/reference/queue/queue/), [vectors](http://www.cplusplus.com/reference/vector/vector/) etc. using same technique. You can checkout full code [here](https://github.com/jiten-thakkar/GenericStack) which has additional functionality of getMax using function pointer.


