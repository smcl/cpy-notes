# CPython - notes

Based on lectures from [Philip Guo](https://www.youtube.com/watch?v=LhadeL7_EIU&list=PLwyG5wA5gIzgTFj5KgJJ15lxq5Cv6lo_0).

* [Lecture 1](https://www.youtube.com/watch?v=dJD9mLPCkuc&list=PLwyG5wA5gIzgTFj5KgJJ15lxq5Cv6lo_0&index=1)
* [Lecture 2](https://www.youtube.com/watch?v=dJD9mLPCkuc&list=PLwyG5wA5gIzgTFj5KgJJ15lxq5Cv6lo_0&index=2)
* [Lecture 3](https://www.youtube.com/watch?v=dJD9mLPCkuc&list=PLwyG5wA5gIzgTFj5KgJJ15lxq5Cv6lo_0&index=3)
* [Lecture 4](https://www.youtube.com/watch?v=dJD9mLPCkuc&list=PLwyG5wA5gIzgTFj5KgJJ15lxq5Cv6lo_0&index=4)
* [Lecture 5](https://www.youtube.com/watch?v=dJD9mLPCkuc&list=PLwyG5wA5gIzgTFj5KgJJ15lxq5Cv6lo_0&index=5)
* [Lecture 6](https://www.youtube.com/watch?v=dJD9mLPCkuc&list=PLwyG5wA5gIzgTFj5KgJJ15lxq5Cv6lo_0&index=6)

--------

A handful of folders of interest in root of source code dir:

### Include/
includes/interfaces.
important file: opcode.h

### Python/
source code for python interpreter itself
important file: ceval.c

### Lib/
library functions - written in python 

### Modules/
source code for python modules

--------

Some interesting stuff:

```
$ cat t.py
x = 1
y = 2
z = x + y
print(z)
```

we can get a neat disassembly by using the "dis" module (`python -m  <module>` will execute the main() method of <module>)

```
$ python -m dis t.py
  1           0 LOAD_CONST               0 (1)
              3 STORE_NAME               0 (x)

  2           6 LOAD_CONST               1 (2)
              9 STORE_NAME               1 (y)

  3          12 LOAD_NAME                0 (x)
             15 LOAD_NAME                1 (y)
             18 BINARY_ADD
             19 STORE_NAME               2 (z)

  4          22 LOAD_NAME                2 (z)
             25 PRINT_ITEM
             26 PRINT_NEWLINE
             27 LOAD_CONST               2 (None)
             30 RETURN_VALUE
```

What this does is:

1. push the constant "1" to the value stack
2. pop the top of the stack and store reference against name "x"
3. push constant "2" to the stack
4. pop top of the stack and store reference against name "y"
5. take reference to "x" and push to stack
6. take reference to "y" and push to stack
7. call binary_add opcode - which will take top two values from the stack and combine them (it's overloaded, so can do different things depending on the type) - will push result to stack
8. then call print_item, which will take top of stack and print to stdout
9. then print_newline, which will print a \n to stdout

Main execution of python interpreter is PyEval_EvalFrameEx() - there's an infinite loop

```
    for (;;) {                      // line  964
    // ...

            switch (opcode) {       // line 1109
    // ...
		case NOP:
			goto fast_next_opcode:
		case LOAD_FAST:
			x = GETLOCAL(oparg);
			if (x != NULL) {
    // ...
			
	    } /* switch */          // line 2823

    // ...
    }                               // line 3021
```
