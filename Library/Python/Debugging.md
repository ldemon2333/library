# Introduction

When an error occurs in a Python program, a traceback is displayed. For example:

```py
Traceback (most recent call last):
  File ".../ex.py", line 7, in <module>
    print(f(4))
          ^^^^
  File ".../ex.py", line 2, in f
    print(g(x + 1) + 2)
          ^^^^^^^^
  File ".../ex.py", line 5, in g
    return print(2) + 3
           ~~~~~~~~~^~~
TypeError: unsupported operand type(s) for +: 'NoneType' and 'int'
```

## Traceback Messages

The lines in the traceback are paired together. The **first** line in each pair has the following format:

```py
File "<file name>", line <number>, in <function>
```

The **second** line in the pair (it's indented farther in than the first) displays the actual line of code that makes the _next_ function call. This gives you a quick look at what expressions were involved.

The traceback is organized with the **most recent call last**, so look at the bottom.

## Error Messages

The very last line in the traceback message is the error statement. An _error statement_ has the following format:

```py
<error type>: <error message>
```

This line provides you with two pieces of information:

- **Error type**: the type of error that was caused (e.g. `SyntaxError`, `TypeError`). These are usually descriptive enough to help you narrow down your search for the cause of error.
- **Error message**: a more detailed description of exactly what caused the error. Different error types produce different error messages.

# Debugging Techniques
## Running doctests

Python has a great way to quickly write tests for your code. These are called doctests, and look like this:

```py
def foo(x):
    """A random function.

    >>> foo(4)
    4
    >>> foo(5)
    5
    """
```

The lines in the docstring that look like interpreter outputs are the **doctests**. To run them, go to your terminal and type:

```sh
python3 -m doctest file.py
```

This effectively loads your file into the Python interpreter, and checks to see if each doctest input (e.g. `foo(4)`) is the same as the specified output (e.g. `4`). If it isn't, a message will tell you which doctests you failed.

The command line tool has a `-v` option that stands for _verbose_.

```sh
python3 -m doctest file.py -v
```

In addition to telling you which doctests you failed, it will also tell you which doctests passed.

## Printing

Once the doctests tell you where the error is, you have to figure what went wrong. If the doctest gave you a traceback message, look at what [type of error](https://cs61a.org/articles/debugging/#error-types) it is to help narrow your search. Also check that you aren't making any [common mistakes](https://cs61a.org/articles/debugging/#common-bugs).

When you first learn how to program, it can be hard to spot bugs in your code. One common practice is to add `print` calls. For example, let's say the following function `foo` keeps returning the wrong thing:

```py
def foo(x):
    result = some_function(x)
    return result // 5
```

We can add a print call before the return to check what `some_function` is returning:

```py
def foo(x):
    result = some_function(x)
    print('DEBUG: result is', result)
    return other_function(result)
```


If it turns out `result` is not what we expect it to be, we would go look in `some_function` to see if it works properly. Otherwise, we might have to add a print call before the return to check `other_function`:

```py
def foo(x):
    result = some_function(x)
    print('DEBUG: result is', result)
    tmp = other_function(result)
    print('DEBUG: other_function returns', tmp)
    return tmp
```

Some advice:

- Don't just print out a variable -- add some sort of message to make it easier for you to read:
    
	```py
    print(x)   # harder to interpret
    print('DEBUG: x =', x)  # easier
    ```
    
- Use `print` calls to view the results of function calls (i.e. after function calls).
- Use `print` calls at the end of a `while` loop to view the state of the counter variables after each iteration:
    
    ```py
    i = 0
    while i < n:
        i += func(i)
        print('DEBUG: i is', i)
    ```
## Interactive Debugging

One way a lot of programmers like to investigate their code is by using Python interactively:

```py
python3 -i file.py
```

starts an interactive Python session after executing the contents of `file.py`.

## PythonTutor Debugging

Sometimes the best way to understand what is going on with a given piece of python code is to create an environment diagram.[PythonTutor](http://tutor.cs61a.org/) creates environment diagrams automatically.

# Error Types

The following are common error types that Python programmers run into.

## `SyntaxError`

- **Cause**: code syntax mistake
- **Example**:
    
    ```
      File "file name", line number
        def incorrect(f)
                        ^
    SyntaxError: invalid syntax
    ```
    
- **Solution**: the `^` symbol points to the code that contains invalid syntax. The error message doesn't tell you _what_ is wrong, but it does tell you _where_.
- **Notes**: Python will check for `SyntaxErrors` before executing any code. This is different from other errors, which are only raised during runtime.

## `IndentationError`

- **Cause**: improper indentation
- **Example**:
    
    ```
      File "file name", line number
        print('improper indentation')
    IndentationError: unindent does not match any outer indentation level
    ```
    
- **Solution**: The line that is improperly indented is displayed. Simply re-indent it.
- **Notes**: If you are inconsistent with tabs and spaces, Python will raise one of these. Make sure you use spaces! (It's just less of a headache in general in Python to use spaces and all cs61a content uses spaces).

## `TypeError`

- **Cause 1**:
    
    - Invalid operand types for primitive operators. You are probably trying to add/subract/multiply/divide incompatible types.
    - **Example**:
        
        ```
        TypeError: unsupported operand type(s) for +: 'function' and 'int'
        ```
        
- **Cause 2**:
    
    - Using non-function objects in function calls.
    - **Example**:
        
        ```
        >>> square = 3
        >>> square(3)
        Traceback (most recent call last):
          ...
        TypeError: 'int' object is not callable
        ```
        
- **Cause 3**:
    
    - Passing an incorrect number of arguments to a function.
    - **Example**:
        
        ```
        >>> add(3)
        Traceback (most recent call last):
          ...
        TypeError: add expected 2 arguments, got 1
        ```
        

## `NameError`

- **Cause**: variable not assigned to anything OR it doesn't exist. This includes function names.
- **Example**:
    
    ```
    File "file name", line number
      y = x + 3
    NameError: global name 'x' is not defined
    ```
    
- **Solution**: Make sure you are initializing the variable (i.e. assigning the variable to a value) before you use it.
- **Notes**: The reason the error message says "global name" is because Python will start searching for the variable from a function's local frame. If the variable is not found there, Python will keep searching the parent frames until it reaches the global frame. If it still can't find the variable, Python raises the error.

## `IndexError`

- **Cause**: trying to index a sequence (e.g. a tuple, list, string) with a number that exceeds the size of the sequence.
- **Example**:
    
    ```
    File "file name", line number
      x[100]
    IndexError: tuple index out of range
    ```
    
- **Solution**: Make sure the index is within the bounds of the sequence. If you're using a variable as an index (e.g. `seq[x]`, make sure the variable is assigned to a proper index.