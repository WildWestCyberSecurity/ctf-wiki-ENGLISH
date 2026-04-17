# Challenges

## 2018 Wangding Cup Round 1 - clip

Using 010 Editor, we can see that the file header contains the word "cloop". After some searching, we found that this is an ancient Linux compressed device. The challenge also states that the device is damaged, so we need to find a normal one. We then searched for how to compress and obtain a cloop file, as follows:

```shell
mkisofs -r test | create_compressed_fs - 65536 > test.cloop
```

Refer to https://github.com/KlausKnopper/cloop. So we compress a file and then discover that the original file header has issues, which we need to repair. Then we consider how to extract files from the cloop file, using:

```
extract_compressed_fs test.cloop now
```

Refer to https://manned.org/create_compressed_fs/f2f838da.

We obtain an ext4 type file. Next, we figure out how to access the contents of this filesystem:

```shell
➜  clip losetup -d /dev/loop0
losetup: /dev/loop0: detach failed: Permission denied
➜  clip sudo losetup -d /dev/loop0
➜  clip sudo losetup /dev/loop0 now                                                 
losetup: now: failed to set up loop device: Device or resource busy
➜  clip sudo losetup /dev/loop0 /home/iromise/ctf/2018/0820网鼎杯/misc/clip/now        
losetup: /home/iromise/ctf/2018/0820网鼎杯/misc/clip/now: failed to set up loop device: Device or resource busy
➜  clip losetup -f           
/dev/loop10
➜  clip sudo losetup /dev/loop10 /home/iromise/ctf/2018/0820网鼎杯/misc/clip/now
➜  clip sudo mount /dev/loop10 /mnt/now
➜  clip cd /mnt/now 
➜  now ls        
clip-clip.png  clip-clop.png  clop-clip.png  clop-clop.jpg  flag.png
```

The last step is to repair the flag. It was just missing a few characters from the file header.
