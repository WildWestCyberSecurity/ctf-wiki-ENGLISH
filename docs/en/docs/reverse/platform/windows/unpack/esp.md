# ESP Law Method

The ESP Law method is a powerful tool for unpacking and one of the most frequently used unpacking methods.

## Key Points

The principle of the ESP Law is to leverage the stack balance in a program to quickly find the OEP.

During the self-decryption or self-decompression process of a program, many packers first push the current register states onto the stack, for example using `pushad`, and after decompression is complete, pop the previous register values, for example using `popad`. Therefore, when the registers are popped, the program code is usually restored, and at that point the hardware breakpoint triggers. Then, from the program's current position, only a few single-step operations are needed to easily reach the correct OEP location.

1. The program starts with `pushad`/`pushfd` right after being loaded
2. After all registers are pushed onto the stack, set a hardware breakpoint on the ESP register
3. Run the program and trigger the breakpoint
4. Delete the hardware breakpoint and begin analysis

## Example

The sample program can be downloaded here: [2_esp.zip](https://github.com/ctf-wiki/ctf-challenges/blob/master/reverse/unpack/2_esp.zip)

Using the same example as the previous section, the entry point starts with a `pushad`. We press F8 to execute `pushad` to save the register state. In the register window on the right, we can see that the value of the `ESP` register has turned red, indicating that its value has changed.

![esp_01.png](./figure/esp_01.png)

We right-click on the value of the `ESP` register, which is `0019FF64` in the figure, and select `HW break[ESP]`. Then we press `F9` to run the program, and the program will break when the breakpoint is triggered. As shown in the figure, we arrive at position `0040D3B0`. This is the same position we reached through single-step tracing in the previous section, so we won't go into further detail.

![esp_02.png](./figure/esp_02.png)
