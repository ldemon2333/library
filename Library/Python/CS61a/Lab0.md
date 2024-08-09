## Appendix: Useful Python Command Line Options

Here are the most common ways to run Python on a file.

1. Using no command-line options will run the code in the file you provide and return you to the command line. If your file just contains function definitions, you'll see no output unless there is a syntax error.
    
    ```
    python3 lab00.py
    ```
    
2. **`-i`**: The `-i` option runs the code in the file you provide, then opens an interactive session (with a `>>>` prompt). You can then evaluate expressions, for example calling functions you defined. To exit, type `exit()`. You can also use the keyboard shortcut `Ctrl-D` on Linux/Mac machines or `Ctrl-Z Enter` on Windows.
    
    If you edit the Python file while running it interactively, you will need to exit and restart the interpreter in order for those changes to take effect.
    
    Here's how we can run `lab00.py` interactively:
    
    ```
    python3 -i lab00.py
    ```
    
3. **`-m doctest`**: Runs the doctests in a file, which are the examples in the docstrings of functions.
    
    Each test in the file consists of `>>>` followed by some Python code and the expected output.
    
    Here's how we can run the doctests in `lab00.py`:
    
    ```
     python3 -m doctest lab00.py
    ```
    
    When our code passes all of the doctests, no output is displayed. Otherwise, information about the tests that failed will be displayed.

[python] To run `doctest`, follow these steps:

1. **Write a function with examples in the docstring:**

```python
def add(a, b):
    """
    Adds two numbers and returns the result.

    >>> add(2, 3)
    5
    >>> add(-1, 1)
    0
    >>> add(0, 0)
    0
    """
    return a + b
```

2. **Save the script and run `doctest` from the command line:**

```sh
python -m doctest -v your_module.py
```

This command will test the examples in the docstrings and report any failures.

