---
layout: post
title: "C++ smart pointers speedrun"
---

Raw pointers can be useful when you want to manipulate memory directly, but using them comes with consequences if improperly managed, e.g. if you forget to free a pointer, free a pointer more than once, or try to use a pointer after freeing it.

```c++
void func() {
    int *i = new int(1);
    int *j = new int(2);
    delete j;
    
    cout << *j << endl; // undefined: use after free
    delete j;           // undefined: double free
}
// memory leak for i
```

Smart pointers are essentially containers around pointers that have additional memory management capabilities.
C++11 comes with three types of smart pointers: unique pointers, shared pointers, and weak pointers.

## Unique pointers
A unique pointer is a type of smart pointer that ensures a single owner.
It makes sure any memory allocated is reclaimed after the pointer goes out of scope.

```c++
void func() {
    // create unique pointer that points to some heap memory
    unique_ptr<int> i = unique_ptr<int>(new int(1));
    // unique_ptr<int> i(new int(1));           // alternate syntax
    // unique_ptr<int> i = make_unique<int>(1); // alternate syntax
}
// i is freed here automatically
```

To ensure a single owner, unique pointers cannot be assigned or copied.
```c++
unique_ptr<int> i = make_unique<int>(2);
unique_ptr<int> j = i;                    // error: cannot copy

unique_ptr<int> j = make_unique<int>(3);
j = i;                                    // error: cannot assign
```
The copy constructor (used to create a new object from an existing object) and the copy assignment operator (used to replace an existing object with another object) are both intentionally deleted.

You can only _move_ a unique pointer, which transfers its ownership.
```c++
unique_ptr<int> i = make_unique<int>(3);
unique_ptr<int> j = move(i);

// j now owns the memory and i is set to nullptr
```

You can also move a unique pointer by returning it.
```c++
unique_ptr<int> return_ptr() {
    unique_ptr<int> res = make_unique<int>(3);
    return res;
}

int main() {
    unique_ptr<int> i = return_ptr();
    // ownership is transferred from res to i
    return 0;
}
```
The memory is first owned by `res`, then is transferred to `i` via the function call.
Now, when `i` goes out of scope, the memory is freed.

A common struggle with returning raw pointers from functions is that it isn't clear who is responsible for freeing the memory - the caller or the callee?
Usually programmers will try to clarify this in variable names, comments, or docs.
But with smart pointers, the transfer of ownership can be captured in the type.

If you want to delete the memory prematurely (i.e. before the pointer goes out of scope) you can do so with `reset()`:
```c++
unique_ptr<int> i = make_unique<int>(3);
i.reset();
// i's memory is cleared before it goes out of scope
```

## Shared pointers
So far we have only talked about exclusive ownership of pointers.
If you want multiple owners of the same pointer, you can use a `shared_ptr`.

```c++
shared_ptr<int> i = make_shared<int>(4); // ref count = 1
shared_ptr<int> j = i;                   // ref count = 2
```
Unlike a unique pointer, a shared pointer does not free its memory once it is out of scope, since there can still be active users.
Instead, memory is freed when there are no users remaining, i.e. its reference count reaches zero. 

```c++
void create_owners(shared_ptr<int> ptr) {
    shared_ptr<int> owner2 = ptr; // 2: ref count = 2
    shared_ptr<int> owner3 = ptr; // 3: ref count = 3
}
// 4: out of scope, so owner2 and owner3 are freed; ref count = 1

int main() {
    shared_ptr<int> owner1 = make_shared<int>(4); // 1: ref count initialized to 1
    create_owners(owner1);
    cout << *owner1 << endl; // 5: still accessible
    return 0;
}
// 6: owner1 out of scope, ref count = 0 and memory is freed
```

## Circular references
A problem with reference counting memory management is that it cannot deal with cycles.
If two objects contain pointers to each other, neither can be freed due to the other's presence.

```c++
class B; // forward declaration

class A {
    public:
        A() {
            cout << "calling A()\n";
        }
        ~A() {
            cout << "calling ~A()\n";
        }
        void create_ptr(shared_ptr<B> b) {
            ptr_to_B = b;
        }
    private:
        shared_ptr<B> ptr_to_B;
};

class B {
    public:
        B() {
            cout << "calling B()\n";
        }
        ~B() {
            cout << "calling ~B()\n";
        }
        void create_ptr(shared_ptr<A> a) {
            ptr_to_A = a;
        }
    private:
        shared_ptr<A> ptr_to_A;
};

int main() {
    shared_ptr<A> a = make_shared<A>();
    shared_ptr<B> b = make_shared<B>();
    a->create_ptr(b);
    b->create_ptr(a);
    return 0;
}
// output:
// calling A
// calling B
```
In the above example, both `a` and `b` have a pointer to each other, so neither's destructor is called.

## Weak pointers
A weak pointer can be used to break circular references.
A weak pointer is just like a shared pointer, except it does not contribute to the reference count.
In other words, the presence of weak pointers is not enough to keep the memory around.

If we change `A`'s pointer to `B` to be a weak pointer instead:
```
@@ -1 +1 @@
-shared_ptr<B> ptr_to_B;
+weak_ptr<B> ptr_to_B;
```
We now see the destructors called:
```c++
// output:
// calling A()
// calling B()
// calling ~B()
// calling ~A()
```
When the main function goes out of scope, `b` is freed despite `a` having a pointer to it, since the pointer is a weak pointer.
`a` is then freed since both its owners are out of scope.
