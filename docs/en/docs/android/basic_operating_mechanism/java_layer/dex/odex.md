# ODEX Files

## Basic Introduction

We know that the Java layer code of an Android application resides in the class.dex file within the apk file. Under normal circumstances, the dex file is retrieved and parsed each time the program is launched, and obviously doing this every time results in relatively low efficiency. Android developers proposed an approach where, when initially loading a dex file, it is optimized to generate an ODEX file, which is stored in the /data/dalvik-cache directory. When the program is run again later, we only need to directly load this optimized ODEX file, saving the time that would otherwise be spent on optimization each time. For system apps that come with the Android ROM, they are directly converted to odex files and stored in the same directory as the apk. This way, each time the phone boots up, it will be much faster.

## Basic Structure

To be added.

## Generation Process

To be added.



## References

- Android Software Security and Reverse Engineering
