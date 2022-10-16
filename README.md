# buffer

Write your exploit explanations here!

Level One:
In the object code of getbuf, we notice that the function subtracts 0x60, which
is 96 bytes from %rsp, meaning that there is a 96 bytes space from below the
%rbp. The next line, lea -0x50(%rbp), %rdi, tells us that our input will
be read in from 80 bytes below %rbp, that is start to occupy the stack from 80
bytes below. We also know that our address for lights off should be the last 8
bytes of our input, so the rest of our input should take up the 8 bytes of the 
old %rbp. Thus, the total length of out input should be 88 bytes of random input
and 8 bytes consisting of the address, 96 bytes in total. 

Level Two:
Like what we done in level one, we need to let the program go to sandwich_order,
so we do the same by writing 88 random bytes and the last 8 bytes the address
of sandwich_order. We use gdb to check the behavior of the %rsp and %rbp during 
the program's jump to sandwich_order, and discovered that after the call getbuf
ended, rsp is at the position in the stack just below where the return address
is stored, and when after it jumps, rsp switches to the position 8 bytes above, 
jumping over the address of sandwich_order in our input. Stepping into sandwich_
order, the function pushes the old rbp onto stack, so now rsp switch again to
8 bytes lower, at 0x557834d8, 8 bytes below the rest of our input. By observing
the behavior of sandwich_order, we can see that 16 bytes above rbp (which 
becomes the same position as rsp after the second line), the value stored there
will be moved to %rax, and 20 bytes above, the value will be moved to %rcx and
then to %rdi. The function call of find_total will then sum up the value stored
inside %rdi (supposedly 4 4-byte numbers). From these behaviors, and from the
given c code of sandwich_order, we can reasonably infer that our cookie, which
is 4 bytes long, will be at above 16 bytes above rbp, and after it, comes the
4 4-bytes-long number that represent the size 4 array. Therefore, in our input
file, right after where we write the address of the sandwich_order, we should
add 8 bytes of input that overwrite the 8 bytes space where the return address
of the current function is stored. Then, we write our cookie in little endian, 
which will take up 4 bytes, and in total 16 bytes of 4 numbers that by summing
up, exceed 20. 

Level Three:
In level three, the main goal of our machine code is to let getbuf return our
cookie, and not to corrupt the stack. By looking through the change of registers
in getbuf, we can see that the one byte below %rbp is where the return value is
returned. If our buffer is shorter than 80 bytes, it will return 1, which has
been stored there before calling gets, if longer, it will return what our input
is at 80 byte. This value will be stored inside %rax, so what our machine code
will do is, store our cookie in side %rax afterwards. I put my object code at
the start of my buffer, and at the end of my input, after the 80 bytes of 
buffer, I put in the old %rbp value, 0x55783500, to avoid potential dammage to
the stack, and overwrite the original return address with 0x55783480, which is
the address of where my machine code starts, so after calling getbuf, the 
program will jump to my machine code instead of returning to line 7 of test_
exploit. However, after I modifies the value stored in %rax, I want to return
to line 7 again, and to not get a seg fault for corrupting the stack. From 
overwriting the original address with our input, we can observe that %rsp has
moved 8 bytes up from where is orignally is when inside our machine code. We
want to put it back to its original place so that the program won't know it
has jumpped to our machine code. To do this, we push the address of line 7 
(where we want out code to jump to next) onto stack by writing pushq $0x4011c5
at the start of our machine code. In this way, our program knows where to go
to after the retq in our machine code, and we will not cause a seg fault. 

Level Four:
Just like what we have done in level three, but this time, we need to consider
that the %rbp changes everytime when the getbufn is called again, meaning that 
our buffer start at a new address each time. So we cannot simply overwrite the 
return address with the address of the start of our 1st buffer. Therefore, we 
can make use of no ops to solve this problem. I locate my machine code at the
half of my buffer, with the first half filled with no ops. Then, by using gdb,
I recoreded down the five %rbp values, and use them to find the 5 addresses 
where each of my buffer start. I then pick an address that is larger than all
five of them but no larger than the address I place my machine code, 
and set it as the return address I overwrites, since this will avoid my code 
from jumping to somewhere out of my buffer and cause a seg fault. Then I 
discover that my stack will be corrupted each time, which is caused by the way
I overwrite the stored old %rbp value. With each turn the %rbp value after
stepping into my exploit will always be my input value, but I want it to be
the old %rbp originally stored there. Moreover, the old %rbp is different each
time, which tells me that I need to make this change in my exploit code. After
observing register values, I find that the old %rbp should be 0x20 larger than 
the %rsp value the moment it enters my exploit code. Thus, at the first line
of my exploit code, I added lea 0x20(%rsp), %rbp to change %rbp to store the
the address of %rsp plus 0x20, which is what we want: the old %rbp. This way, 
we will not corrupt the stack. 