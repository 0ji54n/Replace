![file command](https://media.discordapp.net/attachments/1001097982957068298/1087932235119853669/image.png)

![open exe](https://cdn.discordapp.com/attachments/1001097982957068298/1087933252066615347/image.png)

It has an input box with only number accepted and a **check** button, but after pressing it will makes the whole program crash.
Next step is throw the file into IDA for static analysis.

![IDA first screen](https://cdn.discordapp.com/attachments/1001097982957068298/1087952500168077393/image.png)

In the **WinMain** function, **DialogBoxParamA** has **lpDialogFunc** is **DialogFunc**, so I decided to jump there.

![DialogFunc](https://cdn.discordapp.com/attachments/1001097982957068298/1087952774861443162/image.png)

![GetDlgItemInt](https://cdn.discordapp.com/attachments/1001097982957068298/1087952907489529907/image.png)

Go further down, we can see the **GetDlgItemInt** looks like the function to handle our input in Dialog Box and convert it to a unsigned integer number.
After returning to **DialogFunc**, the result will be stored in *number*. Then *changeable* will be called next, the reason why I called it *changeable* will also be explained in a few next steps.

![Changeable before change](https://media.discordapp.net/attachments/1001097982957068298/1087955580557205566/image.png)

**inc_num** will increase the *number* value by 1 and return, but *call $+5* make the program call it twice.

![how it works](https://cdn.discordapp.com/attachments/1001097982957068298/1087957683375710288/image.png)

Then it returns to some opcode that IDA doesn't recognize as code, and it also won't be converted to correct assembly code sometimes so I decided to switch to x32dbg at this point.

![adding number with something in x32dbg](https://media.discordapp.net/attachments/1001097982957068298/1087943960179265606/image.png)

The *number* will be add with 0x601605c7 and inc twice again. Then it will be stored/saved in eax. Next, *mov dword ptr ds:[changeable], 0xc39000C6* is where the actual fun begins.
*Changeable* will be called right after that, but look at it.

![changeable after change](https://media.discordapp.net/attachments/1001097982957068298/1087945438541398097/image.png)

Looks totally different! Now it just moving a nop to [eax] and return. And eax is *number* after **inc_num** 4 times and adding with 0x601605c7! The program crash because eax will always be a very large number - invalid address.

![inc eax and call changeable again](https://media.discordapp.net/attachments/1001097982957068298/1087946410139332699/image.png)

Increasing eax by 1 and calling changeable again mean we can overwrite 2 opcodes next to each other to 0x90. Sounds interesting! But where do we want to overwrite? And how can we make *eax* to a valid address?
Go back to IDA, we see that the *jmp short loc_401084* (0x11eb) makes the function call SetDlgItemTextA(esi, 0x3e9, &"Correct!") will never be hit.
So the goal will be to overwrite 0x11eb to 0x9090, then the program won't crash anymore and SetDlgItemTextA will be called.

0x401071 = input_number+4+0x601605c7 => input_number = -1,607,857,498 => UINT(input_number) = input_number + 2^32 = 2687109798

And here is the result:

![Correct](https://media.discordapp.net/attachments/1001097982957068298/1087951556634226708/image.png)
