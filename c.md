

# C Language

## Introduction
This wiki aims to teach the basics of the C Language, using a practical approach. After a short introduction to each topic, the reader may practice by attempting the proposed exercises. We assume the reader is comfortable using Linux and the bash shell. For a refresher, check out [`linux-basics.md`](/linux-basics.md)

## Installation
There are many C compilers out there, but this wiki will use the GNU GCC compiler, which can be installed on Linux with the following:
```bash
$ sudo apt install gcc
```
Run `gcc --version` to be sure everything went smoothly:
```
$ gcc --version
gcc (Ubuntu 9.3.0-17ubuntu1~20.04) 9.3.0
Copyright (C) 2019 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```
You should now be able to compile and execute any C source code:

```bash
$ gcc prog.c
$ ./a.out
```
You can also use arguments to specify a name for the executable.
```bash
$ gcc prog.c -o prog
$ ./prog
```
[Here](https://gcc.gnu.org/onlinedocs/gcc/Invoking-GCC.html) is a more comprehensive list of gcc arguments.

## Printf()
Most of your programs will start with the following line:
```c
# include <stdio.h>
```
This gives you access to the `printf` function (among other utils), which will let you print formatted strings to stdout, or standard out(put). You can use the printf function like so:

```c
#include <stdio.h>

int main() {
	printf("Hello, World!\n");
	return 0;
}
```
We will go through and explain each line.

```c
int main() {
```
Here, we define the `main` function, your code must start here. `main` is a very important function, and on standard systems, it is where code execution starts. There are only [a few cases]((https://stackoverflow.com/questions/3379190/avoiding-the-main-entry-point-in-a-c-program)) when main isn't the starting routine for a C program.

```c
printf("Hello, World!\n");
```

This is the printf function - which stands for print formatted - and if you give it a string (a set of characters enclosed within double quotes) it will print that string to stdout.

### Printing strings and other types
Programming languages use types to interpret data. Manipulation of the same binary may lead to different results depending on how we interpret it: as an integer, real number, character, user-defined type, ELF file, etc. Most languages have native types to handle integers, (`int` in C) real numbers (`double`), characters (`char`). You might be surprised to learn that C has no native `string` type. This is because a string in C is treated as a _location in memory_ big enough to contain all the characters in the string consecutively, and one special character called the null-terminator, which marks the end of the string. The following is a visualization of how a string may look in memory:
```
+---+---+---+---+---+---+---+---+---+---+---+---+----+----+
| H | e | l | l | o |   | W | o | r | l | d | ! | \n | \0 |
+---+---+---+---+---+---+---+---+---+---+---+---+----+----+
```

Say you want to estimate the perimeter of a circle with radius r = 1. If we replace the above print statement with the following:
```c
printf("The perimeter of a circle with radius one is roughly: " + (2 * 3.14159));
```
We get an error yelling about invalid operands to binary `+`.  This is because we're attempting to add a `float` value to a string, or location in memory. This is not defined in C, so we get a compile-time error. But how do we print numbers then? Well, `printf` has _format specifiers_, that will "stand in" for a value in a string, and `printf` will handle printing the complete string to stdout.

Below are the most common format specifiers we will use:
```

| Format specifier| Data type           |
| --------        |:-------------:      |
| %d              | integer             |
| %u              | unsigned integer    |
| %s           	  | string*             |
| %f              | float               |
| %lf             | double (long float) |
```

[Here](https://www.cplusplus.com/reference/cstdio/printf/) is a more comprehensive list of format specifiers.

And here is the fixed code:
```c
printf("The perimeter of a circle with radius one is roughly: %lf", 2 * 3.14159);
```
The value `2 * 3.14159` is evaluated and the result substituted in for the format specifier `%lf`, that stands for the double type. 

### Exercises

1. Use a print statement to estimate the surface area of a sphere with a radius of 10
2. Why is the output of `printf("%u", -1);` so different from the number passed to printf? [Hint](https://stackoverflow.com/questions/16056758/c-c-unsigned-integer-overflow)
3. Surprisingly,`printf("The number ten is: " + 10); ` is legal C, and runs! But something strange happens to the output. Why might this be? 
Hint: Try labeling each character in the string with its index. Remember a string is a location in memory and the null terminator marks the end of a string. What is the minimum amount of information `printf` needs in order to know what sequence of characters to print to stdout? 

### Further reading
* [Man page for printf](https://linux.die.net/man/3/printf)

## Variables and input

We can store data of a certain type by creating a variable of that type and giving it a name:
```c
int myint = 12;
```
Here we define a variable called `myint` that stores `int` data - or integer values, and we assign it a value of `12` using the assignment operator, `=`. Now, any time that we write the identifier `myint` in an expression or on the left hand side of a statement, it will be replaced by the current value it is assigned to. Note that `=` does not test for equality but rather assigns the value of the right hand side to the left. For equality testing, use the `==` operator. It will return a value of 1 if its operands are equal, and 0 otherwise.

Wait... a value of 1/0? Why not true/false?

Because bare-bones C is a thin wrapper around the assembly that your code is compiled to and translated into machine code that the computer understands and executes, and assembly has no data types: it's up to the programmer to make sure that the code agrees on what is what. This philosophy was inherited by C, so there is no native `bool` type<sup>[1]</sup> for true and false values, instead a non-zero value is interpreted as true, and a value of 0 to be false.
[1]: in C99 the type _Bool was added to C, and the header file <stdbool.h> to the standard library that is a tiny wrapper that defines values "true" and "false" to be 1 and 0, respectively.

Note however that due to the limited representation of floating-point values that testing for strict equality on two floating point values is begging for bugs. Check out [this stackoverflow post](https://stackoverflow.com/a/4682941/11821550) for more info.

Assigning a data type of less storage to one of more is fine and will simply promote the data type to the larger.
Assigning a data type of larger storage to a smaller one incurs a loss of precision. For example:
```c
	int x = 3.5;
```
is valid, but the .5 will be lost as `x` holds only the value 3.

[Here is a list of data types in C and their minimum guaranteed range.](https://en.wikipedia.org/wiki/C_data_types#Main_types)

### Scanf: The "inverse" of printf()

Printing to stdout is useful, but we're still missing input. That's what the `scanf` function will do for us. It uses the same format specifiers as `printf`, but will take input from stdin - standard input - and store it in memory. `scanf` will require you to tell it where in memory to put the data it reads. To do this, we can use the `&` operator, which will return the address of a variable:

```c
int x;
float y;
double z;
char c;
scanf("%d %f %lf %c", &x, &y, &z, &c);
```
will read in an integer, float, double and char value from stdin and store it in their respective variables.

`printf` and `scanf` will return an integer value depicting the number of bytes printed to stdout and the number of input items successfully matched, respectively.
### Exercises
1. Write a program to calculate the perimeter of any circle.
### Further Reading
* [Man page for scanf](https://linux.die.net/man/3/scanf)
