# Testing Support

The following code is used for testing support. By default, perturb_byte is 0.

```c
static int perturb_byte;

static void alloc_perturb(char *p, size_t n) {
    if (__glibc_unlikely(perturb_byte)) memset(p, perturb_byte ^ 0xff, n);
}

static void free_perturb(char *p, size_t n) {
    if (__glibc_unlikely(perturb_byte)) memset(p, perturb_byte, n);
}
```


