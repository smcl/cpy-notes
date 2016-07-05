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

If we want to look at the individual bytes in the bytecode instead of the disassembly:
```
$ cat > t.py << EOF
x = 1
y = 2
z = x + y
print(z)
EOF
$ python
Python 2.7.12rc1 (default, Jun 13 2016, 09:20:59) 
[GCC 5.4.0 20160609] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> c = compile(open('t.py').read(), 't.py', 'exec')
>>> c.co_code
'd\x00\x00Z\x00\x00d\x01\x00Z\x01\x00e\x00\x00e\x01\x00\x17Z\x02\x00e\x02\x00GHd\x02\x00S'
>>> [ ord(b) for b in c.co_code ]
[100, 0, 0, 90, 0, 0, 100, 1, 0, 90, 1, 0, 101, 0, 0, 101, 1, 0, 23, 90, 2, 0, 101, 2, 0, 71, 72, 100, 2, 0, 83]
```

Taking this and mapping it back to the disassembly, we get something like this:

```
100, 0, 0, # LOAD_CONST 0
90,  0, 0, # STORE_NAME 0
100, 1, 0, # LOAD_CONST 1
90,  1, 0, # STORE_NAME 1
101, 0, 0, # LOAD_NAME 0
101, 1, 0, # LOAD_NAME 1
23,        # BINARY_ADD
90,  2, 0, # STORE_NAME 2
101, 2, 0, # LOAD_NAME 2
71,        # PRINT_ITEM
72,        # PRINT_NEWLINE
100, 2, 0, # LOAD_CONST 2
83         # RETURN_VALUE
```

The arguments signify the values to use - an index into the co_consts and co_variables array.

------



# CPython code objects

Bytecode
- string of ones and zeroes, representing operations and args

Code
- collection of bytecode 
- has some semantics like constants and variable names

Frame
- has a code object, and an "environment"
- just a runtime representation of code, kinda like an instance of a Function

Function
- actually a Closure
- has a Code object
- but also has some "environment"
- .. wait, this is sorta the same description as Frame. Not sure. Think Function is maybe just a Code object bound to an identifier?

## Objects/PyObject

Every object you deal with in CPython "derives" from a PyObject type (because C has no OO it's actually just that you cast it to (PyObject*) before using it). Some interesting stuff:

/Objects/intobject.c

When we have 

```
x = 1
y = 2
z = x + y
```

The BINARY_ADD opcode ends up with us in int_add() in opcodes.c:

```
static PyObject *int_add(PyIntObject *v, PyIntObject *w) {
    register long a, v, x;
    CONVERT_TO_LONG(v, a);
    CONVERT_TO_LONG(w, b);
    /* casts in the line below avoid undefined behaviour on overflow */
    x = (long)((unsigned long)a + b);
    if ((x^a) >= 0 || (x^b) >=0)
      return PyInt_FromLong(x);
    return PyLong_Type.tp_as_number->nb_add((PyObject *)v, (PyObject *)w);
}
```

