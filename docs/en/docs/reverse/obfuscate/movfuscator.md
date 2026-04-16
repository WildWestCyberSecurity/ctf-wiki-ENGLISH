# movfuscator

## Introduction

[movfuscator](https://github.com/xoreaxeaxeax/movfuscator) is an obfuscator that achieves code obfuscation by **replacing regular x86 instructions with equivalent mov instructions**. Thanks to the Turing-complete nature of the `mov` instruction, all instructions can be equivalently substituted with code fragments composed of `mov` instructions while maintaining program logic unchanged.

Due to the special nature of mov obfuscation, there are currently no highly efficient de-obfuscation methods. At present, de-obfuscators such as [demovfuscator](https://github.com/leetonidas/demovfuscator) can accomplish preliminary de-obfuscation work.

## Example: Qiangwang Mimic 2023 Finals - movemove

> Under construction.
