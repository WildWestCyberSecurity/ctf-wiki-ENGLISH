# JPG

## File Structure

-   JPEG is a lossy compression format. When pixel information is saved to a file using JPEG and then read back, some pixel values may change slightly. When saving, there is a quality parameter that can be selected between 0 and 100. The larger the parameter, the more faithful the image, but the larger the file size. Generally, selecting 70 or 80 is sufficient.
-   JPEG does not have transparency information.

The basic data structure of JPG consists of two major types: "segments" and compressed encoded image data.

| Name             | Bytes | Data | Description                                                                  |
| ---------------- | ----- | ---- | ---------------------------------------------------------------------------- |
| Segment Marker   | 1     | FF   | Start marker of each new segment                                             |
| Segment Type     | 1     |      | Type code (called marker code)                                               |
| Segment Length   | 2     |      | Includes segment content and segment length itself, excludes segment marker and segment type |
| Segment Content  | 2     |      | ≤65533 bytes                                                                 |

-   Some segments have no length description or content, only the segment marker and segment type. The file header and file trailer both belong to this type of segment.
-   Any number of `FF` bytes between segments is legal. These `FF` bytes are called "padding bytes" and must be ignored.

Some common segment types:

![](./figure/jpgformat.png)

`0xffd8` and `0xffd9` are the start and end markers of a JPG file.

## Steganography Software

### [Stegdetect](https://github.com/redNixon/stegdetect)

A steganography tool that evaluates the DCT frequency coefficients of JPEG files through statistical analysis techniques. It can detect information hidden by steganography tools such as JSteg, JPHide, OutGuess, Invisible Secrets, F5, appendX, and Camouflage. It also has dictionary-based brute force password cracking methods to extract hidden information embedded through Jphide, outguess, and jsteg-shell.

```shell
-q Only display images that may contain hidden content.
-n Enable JPEG header checking to reduce false positive rates. If enabled, all files with comment sections will be considered as not having embedded information. If the version number in the JFIF identifier of a JPEG file is not 1.1, OutGuess detection is disabled.
-s Modify the sensitivity of the detection algorithm. The default value is 1. The match of detection results is proportional to the sensitivity of the detection algorithm. The higher the sensitivity value, the more likely the detected suspicious files contain sensitive information.
-d Print debug information with line numbers.
-t Set which steganography tools to detect (default detects jopi). The available options are as follows:
j Detect if information in the image was embedded using jsteg.
o Detect if information in the image was embedded using outguess.
p Detect if information in the image was embedded using jphide.
i Detect if information in the image was embedded using invisible secrets.
```

### [JPHS](http://linux01.gwdg.de/~alatham/stego.html)

JPHS is an information hiding software for JPEG images, developed and designed by Allan Latham. It is a tool for encrypting, hiding, detecting, and extracting information in lossy compressed JPEG files on Windows and Linux platforms. The software mainly contains two programs: JPHIDE and JPSEEK. The JPHIDE program mainly implements the function of encrypting and hiding information files into JPEG images, while the JPSEEK program mainly implements the detection and extraction of information files from JPEG images encrypted and hidden by the JPHIDE program. The JPHSWIN program in the Windows version of JPHS has a graphical user interface and provides the functionality of both JPHIDE and JPSEEK.

### [SilentEye](http://silenteye.v1kings.io/)

> SilentEye is a cross-platform application design for an easy use of steganography, in this case hiding messages into pictures or sounds. It provides a pretty nice interface and an easy integration of new steganography algorithm and cryptography process by using a plug-ins system.
