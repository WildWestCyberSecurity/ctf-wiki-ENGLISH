# Audio Steganography

Audio-related CTF challenges mainly use steganography strategies, primarily including MP3 steganography, LSB steganography, waveform steganography, spectrum steganography, and more.

## Common Techniques

Information that can be discovered through `binwalk` and `strings` will not be discussed in detail here.

## MP3 Steganography

### Principle

MP3 steganography mainly uses the [Mp3Stego](http://www.petitcolas.net/steganography/mp3stego/) tool. Its basic introduction and usage are as follows:

> MP3Stego will hide information in MP3 files during the compression process. The data is first compressed, encrypted and then hidden in the MP3 bit stream.

```shell
encode -E hidden_text.txt -P pass svega.wav svega_stego.mp3
decode -X -P pass svega_stego.mp3
```

### Example

> ISCC-2016: Music Never Sleep

After initial observation, `strings` reveals nothing, and listening to the audio shows no abnormalities, suggesting that steganography software was used to hide data.

![](./figure/1.jpg)

After obtaining the password, use `Mp3Stego` to decrypt.

```shell
decode.exe -X ISCC2016.mp3 -P bfsiscc2016
```

The resulting file `iscc2016.mp3.txt`:
```
Flag is SkYzWEk0M1JOWlNHWTJTRktKUkdJTVpXRzVSV0U2REdHTVpHT1pZPQ== ???
```

After Base64 && Base32 decoding, we get the flag.

## Waveform

### Principle

Typically, for waveform-related challenges, after observing anomalies, use relevant software (Audacity, Adobe Audition, etc.) to observe waveform patterns and further convert the waveform into 01 strings, etc., to extract and convert the final flag.

### Example

> ISCC-2017: Misc-04

Actually, the hidden information is in the very beginning of the audio. If you don't listen carefully, you might mistake it for steganography software.

![](./figure/3.png)

Using high as 1 and low as 0, convert to a `01` string.

```
110011011011001100001110011111110111010111011000010101110101010110011011101011101110110111011110011111101
```

Convert to ASCII and decrypt the Morse code to get the flag.

!!! note
    Some more complex challenges may first perform a series of processing on the audio, such as filtering. For example, [JarvisOJ - Voice of God Writeup](https://www.40huo.cn/blog/jarvisoj-misc-writeup.html)

## Spectrum

### Principle

Spectrum steganography in audio hides strings in the frequency spectrum. Such audio usually has an obvious characteristic — it sounds like noise or is rather harsh.

### Example

> Su-ctf-quals-2014:hear_with_your_eyes

![](./figure/4.png)

## LSB Audio Steganography

### Principle

Similar to LSB steganography in image steganography, there is also corresponding LSB steganography in audio. The main tool used is [Silenteye](http://silenteye.v1kings.io/), which is introduced as follows:

> SilentEye is a cross-platform application design for an easy use of steganography, in this case hiding messages into pictures or sounds. It provides a pretty nice interface and an easy integration of new steganography algorithm and cryptography process by using a plug-ins system.

### Example

> 2015 Guangdong Province Qiangwang Cup - Little Apple

Simply use `silenteye` directly.

![](./figure/2.jpg)

## Further Reading

-   [LSB in Audio](https://ethackal.github.io/2015/10/05/derbycon-ctf-wav-steganography/)
-   [Steganography Summary](http://bobao.360.cn/learning/detail/243.html)
