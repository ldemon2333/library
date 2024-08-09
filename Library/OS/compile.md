Compiling a C program involves several steps that typically include writing your C code, compiling it into an executable binary, and optionally linking it with other libraries. Here’s a step-by-step guide to compile a simple C program on a Unix-like system (such as Linux or macOS) using the command line:

### Step-by-Step Guide to Compile a C Program

1. **Write Your C Code**:
   Create a C source file using a text editor. For example, you can use `nano`, `vim`, `emacs`, or any other text editor of your choice.

   ```c
   // hello.c
   #include <stdio.h>
   
   int main() {
       printf("Hello, world!\n");
       return 0;
   }
   ```

   Save this file as `hello.c`.

2. **Open Terminal**:
   Open your terminal or command prompt.

3. **Navigate to the Directory**:
   Use the `cd` command to navigate to the directory where your `hello.c` file is located.

   ```sh
   cd /path/to/your/directory
   ```

4. **Compile the C Program**:
   Use the `gcc` (GNU Compiler Collection) command to compile the `hello.c` source file into an executable binary. You specify the output file name using the `-o` option followed by the desired name.

   ```sh
   gcc hello.c -o hello
   ```

   - `gcc`: The command to invoke the GNU C Compiler.
   - `hello.c`: The source file to compile.
   - `-o hello`: Specifies the output file name as `hello`.

5. **Run the Executable**:
   After successful compilation, you can run the executable file from the terminal.

   ```sh
   ./hello
   ```

   This command executes the `hello` program, which prints `Hello, world!` to the terminal.

### Optional Steps

- **Compiler Options**: You can include additional compiler options such as `-Wall` for enabling all warnings, `-g` for including debugging information, etc.
  
   ```sh
   gcc hello.c -o hello -Wall -g
   ```

- **Linking with Libraries**: If your program requires linking with external libraries (e.g., math library), you would specify them during compilation.

   ```sh
   gcc hello.c -o hello -lm
   ```

   Here, `-lm` links with the math library (`libm`).

### Summary

Compiling a C program involves writing your code, using `gcc` to compile it into an executable, and running the resulting executable. The process ensures that your C code is translated into machine-readable instructions that can be executed on your system. Adjustments can be made based on specific needs, such as adding compiler flags or linking with libraries, to enhance functionality or performance.