# Information Disclosure

## dmesg_restrict

Considering that kernel logs may contain address information or sensitive information, researchers proposed restricting access to kernel logs.

This option controls whether `dmesg` can be used to view kernel logs. When `dmesg_restrict` is 0, there are no restrictions; when this option is set to 1, only users with `CAP_SYSLOG` privileges can view kernel logs through the `dmesg` command.

```
dmesg_restrict:

This toggle indicates whether unprivileged users are prevented
from using dmesg(8) to view messages from the kernel's log buffer.
When dmesg_restrict is set to (0) there are no restrictions. When
dmesg_restrict is set set to (1), users must have CAP_SYSLOG to use
dmesg(8).

The kernel config option CONFIG_SECURITY_DMESG_RESTRICT sets the
default value of dmesg_restrict.
```

## kptr_restrict

This option controls the restrictions imposed when outputting kernel addresses. It primarily restricts the following interfaces:

- Kernel addresses obtained through /proc
- Addresses obtained through other interfaces (to be investigated)

The specific output content depends on the value configured for this option:

- 0: By default, there are no restrictions.
- 1: Kernel pointer addresses output using `%pK` will be replaced with 0, unless the user has CAP_SYSLOG privileges, and the group id and real id are equal.
- 2: Kernel pointers output using `%pK` will all be replaced with 0, regardless of privileges.

```
kptr_restrict:

This toggle indicates whether restrictions are placed on
exposing kernel addresses via /proc and other interfaces.

When kptr_restrict is set to 0 (the default) the address is hashed before
printing. (This is the equivalent to %p.)

When kptr_restrict is set to (1), kernel pointers printed using the %pK
format specifier will be replaced with 0's unless the user has CAP_SYSLOG
and effective user and group ids are equal to the real ids. This is
because %pK checks are done at read() time rather than open() time, so
if permissions are elevated between the open() and the read() (e.g via
a setuid binary) then %pK will not leak kernel pointers to unprivileged
users. Note, this is a temporary solution only. The correct long-term
solution is to do the permission checks at open() time. Consider removing
world read permissions from files that use %pK, and using dmesg_restrict
to protect against uses of %pK in dmesg(8) if leaking kernel pointer
values to unprivileged users is a concern.

When kptr_restrict is set to (2), kernel pointers printed using
%pK will be replaced with 0's regardless of privileges.
```

When this protection is enabled, attackers can no longer obtain sensitive addresses in the kernel through `/proc/kallsyms`, such as commit_creds and prepare_kernel_cred.

## References

- https://blog.csdn.net/gatieme/article/details/78311841
