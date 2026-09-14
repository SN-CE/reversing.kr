In the Disasm in cutter, we go to `main`. At `0x08048563` we see an xref to `str.Wrong`. So we backtrace this path. We see that to arrive at `0x08048563`, the `jne`(or `jnz`) at `0x0804854d` must execute. The `jne` at `0x0804854d` executes if the `cmp eax, 1` right above it, at `0x0804854a`, results in a non-zero value and the zero flag is not set. So this is the check where the binary checks if the password entered matches the password programmed into it. But since the `cmp` instruction is checking equality of `eax` with `1`, the actual check for the password must be done in a function. And that function must be setting `eax` to 0 or 1 before returning. 

***

But to be sure that this `cmp eax, 1` is where the check is happening, we trace the other path from here. What happens if eax **is** 1. The snippet is like so:

```asm
0x0804854a      cmp     eax, 1     ; 1
0x0804854d      jne     0x804855b
0x0804854f      call    fcn.080484f7 ; fcn.080484f7
0x08048554      mov     eax, 0
0x08048559      jmp     0x804857c
-
-
-
0x0804857c      leave
0x0804857d      ret
```

So we see that if the `jne` jump does not happen, it calls a function, then after executing that function, it moves `0` into `eax` and jumps directly to `0x0804857c`, wherein the function exits. So the `cmp eax, 1` is quite possibly the check we need. But to be even more sure, we check the function `fcn.080484f7`.
And when we do, we see this line:

```asm
0x08048505      mov     dword [ptr], str.Correct
```

That is an xref to `str.Correct`. Thus we can be certain that the `cmp eax, 1` check is the correct one.

***

So now we see two function calls right before `cmp eax, 1`. One of them must be the function that does the arithmetic of checking the password:

```asm
0x08048540      call    fcn.08048434 ; fcn.08048434
0x08048545      call    fcn.08048451 ; fcn.08048451
0x0804854a      cmp     eax, 1     ; 1
0x0804854d      jne     0x804855b
```

We check `fcn.08048434` first. We are faced with this line:

```asm
0x0804844a      call    __isoc99_scanf
```

So this is the function that takes in input. We can leave this alone. But before we do, we must keep one address in mind: `0x804a020`. We see this in: 

```asm
0x0804843a      mov     eax, data.08048650 ; 0x8048650
0x0804843f      mov     dword [var_18h], 0x804a020 ; data.0804a020  ;  [0x804a020:4]=0
0x08048447      mov     dword [esp], eax ; const char *format
0x0804844a      call    __isoc99_scanf ; sym.imp.__isoc99_scanf ; int scanf(const char *format)
```

`scanf` takes two arguments. The first argument is the format string, and the second argument is the address where to store input. `eax` stores `data.08048650` and then the value in `eax` is moved to the top of the stack: `mov dword [esp], eax`. The argument at the top of the stack is the first argument. So `data.08048650` is the format string. Thus we can conclude that `0x804a020` is the address of the input string.

***

Now we check `fcn.08048451`. There we find multiple `cmp` and `xor` functions. This is where the arithmetic happens. Now, right here, the simplest way to crack this binary is to patch it. On the very last line of the function, we see: `0x080484f0 mov eax, 1` So the required condition, is that `eax` must be 1. Multiple times in this function, we see this pattern: 

```asm
0x080484a1      mov     eax, 0
0x080484a6      jmp     0x80484f5
```

at `0x80484f5` we find: 

```asm
0x080484f5      pop     ebp
0x080484f6      ret
```

Which means that the binary exits after moving `0` into `eax`. What we can do is, patch all the `mov eax, 0` into `mov eax, 1`. This makes sure that no matter the input string, the binary will always show `Correct!`.

The other way to do it, and find the actual flag, is to go through the instructions and see what it's comparing each input character to. This is done below:

```asm
fcn.08048451();
0x08048451      push    ebp
0x08048452      mov     ebp, esp
0x08048454      movzx   eax, byte [data.0804a021] ; [0x804a021:1]=0
0x0804845b      cmp     al, 0x31   ; 49
0x0804845d      je      0x8048469
0x0804845f      mov     eax, 0
0x08048464      jmp     0x80484f5
0x08048469      movzx   eax, byte [data.0804a020] ; [0x804a020:1]=0
```

At, `0x08048454` we see a single byte, located at `data.0804a021` is moved into `eax`. The location `0x804a021` is 1 `byte` after `0x804a020`.`0x804a020` is where scanf stored the input. Thus, `0x804a021` is the second character of the input string. This byte is moved into `eax` after zero extending it. Then it's compared to `0x31` which is `1` in `ASCII`. `al` is the last byte of `eax`(accumulator low). If they are equal(meaning the second character of the input string is 1) the binary proceeds to start mutating the input string at `0x08048469`. If not, then it does `mov eax, 0`, which is the failure condition and jumps to `0x80484f5` where the function exits. This is the very first check in the function. And it gives us one of the characters required. The second character of the flag is **1**.

***

Now, the mutation:

```asm
0x08048469      movzx   eax, byte [data.0804a020] ; [0x804a020:1]=0
0x08048470      xor     eax, 0x34  ; 52
0x08048473      mov     byte [data.0804a020], al ; [0x804a020:1]=0
0x08048478      movzx   eax, byte [data.0804a022] ; [0x804a022:1]=0
0x0804847f      xor     eax, 0x32  ; 50
0x08048482      mov     byte [data.0804a022], al ; [0x804a022:1]=0
0x08048487      movzx   eax, byte [data.0804a023] ; [0x804a023:1]=0
0x0804848e      xor     eax, 0xffffff88 ; 4294967176
0x08048491      mov     byte [data.0804a023], al ; [0x804a023:1]=0
0x08048496      movzx   eax, byte [data.0804a024] ; [0x804a024:1]=0
0x0804849d      cmp     al, 0x58   ; 88
0x0804849f      je      0x80484a8
0x080484a1      mov     eax, 0
0x080484a6      jmp     0x80484f5
```

We find that 3 characters are mutated. The characters held at `0x804a020`(the first character), `0x804a022`(the third character), and `0x804a023`(the fourth character). The fifth character at `0x0804a024` is moved into `eax` and is compared with `0x58`. Which is the letter `X` in `ASCII`. And we find the same pattern of `mov eax, 0` and `jmp 0x80484f5`, if the fifth character is not `X`. So now two characters of the flag are known to us. **1** and **X**.

The sequence so far is:

- char1 XOR 0x34
- **1**
- char3 XOR 0x32
- char4 XOR 0xffffff88. Because the binary only works with the last byte anyway, we can write this as: char4 XOR 0x88 
- **X**

***

After this we see another comparison snippet, that is unlike the rest:

```asm
0x080484a8      movzx   eax, byte [data.0804a025] ; [0x804a025:1]=0
0x080484af      test    al, al
0x080484b1      je      0x80484ba
0x080484b3      mov     eax, 0
0x080484b8      jmp     0x80484f5
```

The sixth character is being moved into `eax`. It is then being `test` -ed against itself. `test` performs bitwise `and`, thus if the value in al is 0, then and only then will the zero flag be set. Which brings us to the next line, `je 0x80484ba`(`je`, although the more contextually clear instruction would be `jz`) which is, if the zero flag is set it will not jump to the failure condition and proceed as intended. `0` here refers to `0x00`, which is the null terminator. So this whole block checks if the sixth character is a null-terminator, which is only possible if the string has 5 characters. Now we know that the flag is a 5 character string.

***

Going through the direct comparison the binary does with the mutated characters:

- char1: 

```asm
0x080484cc      movzx   eax, byte [data.0804a020] ; [0x804a020:1]=0
0x080484d3      cmp     al, 0x78   ; 120
0x080484d5      je      0x80484de
0x080484d7      mov     eax, 0
0x080484dc      jmp     0x80484f5
```

This compares the mutated char1, **char1 XOR 0x34** with **0x78**. If it's not equal, the function exits.

- char3

```asm
0x080484ba      movzx   eax, byte [data.0804a022] ; [0x804a022:1]=0
0x080484c1      cmp     al, 0x7c   ; 124
0x080484c3      je      0x80484cc
0x080484c5      mov     eax, 0
0x080484ca      jmp     0x80484f5
```

This compares the mutated char3, **char3 XOR 0x32** with **0x7c**. If it's not equal, the function exits.

- char4

```asm
0x080484de      movzx   eax, byte [data.0804a023] ; [0x804a023:1]=0
0x080484e5      cmp     al, 0xdd   ; 221
0x080484e7      je      0x80484f0
0x080484e9      mov     eax, 0
0x080484ee      jmp     0x80484f5
```

This compares the mutated char4, **char4 XOR 0x88** with **0xdd**. If it's not equal, the function exits.

***

The final sequence becomes:

- **char1 XOR 0x34 = 0x78**
- **1**
- **char3 XOR 0x32 = 0x7c**
- **char4 XOR 0x88 = 0xdd**
- **X**

**Solving char1:**
char1 XOR 0x34 = 0x78, *or*
char1 = 0x34 XOR 0x78, *or*
char1 = 0x4c

0x4c is the letter **L** in `ASCII`. **char1 = L**

**Solving char3:**
char3 XOR 0x32 = 0x7c, *or*
char3 = 0x7c XOR 0x32, *or*
char3 = 0x4e

0x4e is the letter **N** in `ASCII`. **char3 = N**

**Solving char4:**
char4 XOR 0x88 = 0xdd
char4 = 0xdd XOR 0x88
char4 = 0x55

0x55 is the letter **U** in `ASCII`. **char4 = U**

***

**The Final Flag: L1NUX**




