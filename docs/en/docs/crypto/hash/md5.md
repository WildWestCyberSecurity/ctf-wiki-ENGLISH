# MD5

## Basic Description

The input and output of MD5 are as follows:

- Input: A message of arbitrary length, processed in 512-bit blocks.
- Output: A 128-bit message digest.

For a detailed introduction, please search on your own.

Additionally, sometimes the MD5 we obtain is 16 characters long. In fact, those 16 characters are derived from the 32-character MD5 value. It is obtained by removing the first eight characters and the last eight characters from the 32-character MD5.

Generally, we can determine whether a function is an MD5 function by its initialization. If a function has the following four initialization variables, we can guess that it is an MD5 function, as these are the initialization IVs of the MD5 function.

```
0x67452301，0xEFCDAB89，0x98BADCFE，0x10325476
```

## Cracking

Currently, MD5 can be considered basically broken. Common MD5 collisions can be obtained from the following websites:

- http://www.cmd5.com/
- http://www.ttmd5.com/
- http://pmd5.com/
- https://www.win.tue.nl/hashclash/fastcoll_v1.0.0.5.exe.zip (generates MD5 collisions with a specified prefix)

## Challenges

- CFF 2016 Many Salts
- JarvisOJ Many Salts
