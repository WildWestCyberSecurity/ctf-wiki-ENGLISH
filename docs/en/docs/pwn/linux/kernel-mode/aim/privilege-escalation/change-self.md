# Change Self

The kernel indexes the cred structure through the cred pointer in the process's `task_struct` structure, and then determines the privileges a process has based on the contents of the cred. If the uid-fsgid members of the cred structure are all 0, the process is generally considered to have root privileges.

```c
struct cred {
	atomic_t	usage;
#ifdef CONFIG_DEBUG_CREDENTIALS
	atomic_t	subscribers;	/* number of processes subscribed */
	void		*put_addr;
	unsigned	magic;
#define CRED_MAGIC	0x43736564
#define CRED_MAGIC_DEAD	0x44656144
#endif
	kuid_t		uid;		/* real UID of the task */
	kgid_t		gid;		/* real GID of the task */
	kuid_t		suid;		/* saved UID of the task */
	kgid_t		sgid;		/* saved GID of the task */
	kuid_t		euid;		/* effective UID of the task */
	kgid_t		egid;		/* effective GID of the task */
	kuid_t		fsuid;		/* UID for VFS ops */
	kgid_t		fsgid;		/* GID for VFS ops */
  ...
}
```

Therefore, the approach is quite intuitive. We can escalate privileges in the following ways:

- Directly modify the contents of the cred structure
- Modify the cred pointer in the task_struct structure to point to a cred that meets the requirements

Regardless of the method, it generally consists of two steps: locating and modifying. It's like putting an elephant into a refrigerator.

## Directly Modify cred

### Locate the Exact Position

We can first obtain the exact address of cred, then modify the cred.

#### Locating

There are many ways to locate the exact address of cred. Here we categorize them based on whether the locating is direct:

##### Direct Locating

The beginning of the cred structure records various id information. For a normal process, uid-fsgid are all the identity of the user executing the process. Therefore, we can locate cred by scanning memory.

```c
struct cred {
	atomic_t	usage;
#ifdef CONFIG_DEBUG_CREDENTIALS
	atomic_t	subscribers;	/* number of processes subscribed */
	void		*put_addr;
	unsigned	magic;
#define CRED_MAGIC	0x43736564
#define CRED_MAGIC_DEAD	0x44656144
#endif
	kuid_t		uid;		/* real UID of the task */
	kgid_t		gid;		/* real GID of the task */
	kuid_t		suid;		/* saved UID of the task */
	kgid_t		sgid;		/* saved GID of the task */
	kuid_t		euid;		/* effective UID of the task */
	kgid_t		egid;		/* effective GID of the task */
	kuid_t		fsuid;		/* UID for VFS ops */
	kgid_t		fsgid;		/* GID for VFS ops */
  ...
}
```

**During the actual locating process, we may find many creds that meet the requirements. This is mainly because the cred structure may be copied or freed.** An intuitive idea is to filter out some creds by checking that usage is not 0 during the locating process, but we may still find some creds with usage equal to 0. This is because there is a certain delay between when cred's usage becomes 0 and when it is freed. Additionally, cred uses RCU for deferred freeing.

##### Indirect Locating

###### task_struct

The process's `task_struct` structure stores a pointer to cred, so we can:

1. Locate the address of the current process's `task_struct` structure

2. Calculate the storage address of the `cred` pointer based on its offset relative to the task_struct structure

3. Obtain the actual address of `cred`

###### comm

comm is used to mark the name of the executable file and is located in the process's `task_struct` structure. We can see that comm is actually right below cred, so we can also first locate comm, then locate the address of cred.

```c
	/* Process credentials: */

	/* Tracer's credentials at attach: */
	const struct cred __rcu		*ptracer_cred;

	/* Objective and real subjective task credentials (COW): */
	const struct cred __rcu		*real_cred;

	/* Effective (overridable) subjective task credentials (COW): */
	const struct cred __rcu		*cred;

#ifdef CONFIG_KEYS
	/* Cached requested key. */
	struct key			*cached_requested_key;
#endif

	/*
	 * executable name, excluding path.
	 *
	 * - normally initialized setup_new_exec()
	 * - access it with [gs]et_task_comm()
	 * - lock it with task_lock()
	 */
	char				comm[TASK_COMM_LEN];
```

However, if the process name is not unique, there may be multiple identical strings in the kernel, which affects the correctness and efficiency of the search. Therefore, we can use prctl to set the process's comm to a special string before starting to locate comm.

#### Modification

With this method, we can directly modify uid-fsgid in cred to 0. Of course, there are many ways to modify it, for example:

- After we have arbitrary address read/write, we can directly modify cred.
- After we can execute code via ROP, we can use ROP gadgets to modify cred.

### Indirect Locating

Although we do want to modify the contents of cred, we don't necessarily need to know the exact location of cred; we just need to be able to modify cred.

#### (Obsolete) UAF Using the Same Heap Chunk

If we can control the position of the cred structure during process initialization and we can modify that region's contents after initialization, then we can easily achieve privilege escalation. Here is a typical example:

1. Allocate a heap chunk with the same size as the cred structure
2. Free the heap chunk
3. Fork a new process, which happens to use the just-freed heap chunk
4. At this point, modify specific memory in the cred structure to escalate privileges

However, **this method is no longer viable in newer kernel versions, as we can no longer directly allocate objects from cred\_jar**. This is because cred\_jar is created with the `SLAB_ACCOUNT` flag set, and when `CONFIG_MEMCG_KMEM=y` (enabled by default), **cred\_jar will no longer be merged with kmalloc-192 of the same size**.

```c
void __init cred_init(void)
{
	/* allocate a slab in which we can store credentials */
	cred_jar = kmem_cache_create("cred_jar", sizeof(struct cred), 0,
			SLAB_HWCACHE_ALIGN|SLAB_PANIC|SLAB_ACCOUNT, NULL);
}
``` 

## Modify the cred Pointer

### Locate the Exact Position

With this approach, we need to know the exact address of the cred pointer.

#### Locating

##### Direct Locating

Obviously, the cred pointer does not have any particularly special characteristics, so it is difficult to locate the cred pointer through direct locating.

##### Indirect Locating

###### task_struct

The process's `task_struct` structure stores a pointer to cred, so we can:

1. Locate the address of the current process's `task_struct` structure

2. Calculate the storage address of the `cred` pointer based on its offset relative to the task_struct structure

###### common

comm is used to mark the name of the executable file and is located in the process's `task_struct` structure. We can see that comm is actually right below the cred pointer, so we can also first locate comm, then locate the address of the cred pointer.

```c
	/* Process credentials: */

	/* Tracer's credentials at attach: */
	const struct cred __rcu		*ptracer_cred;

	/* Objective and real subjective task credentials (COW): */
	const struct cred __rcu		*real_cred;

	/* Effective (overridable) subjective task credentials (COW): */
	const struct cred __rcu		*cred;

#ifdef CONFIG_KEYS
	/* Cached requested key. */
	struct key			*cached_requested_key;
#endif

	/*
	 * executable name, excluding path.
	 *
	 * - normally initialized setup_new_exec()
	 * - access it with [gs]et_task_comm()
	 * - lock it with task_lock()
	 */
	char				comm[TASK_COMM_LEN];
```

However, if the process name is not unique, there may be multiple identical strings in the kernel, which affects the correctness and efficiency of the search. Therefore, we can use prctl to set the process's comm to a special string before starting to locate comm.

#### Modification

When performing the actual modification, we can use the following two approaches:

- Modify the cred pointer to the address of the existing init_cred in the kernel image. This method is suitable when we can directly modify the cred pointer and know the address of init_cred.
- Forge a cred, then modify the cred pointer to point to that address. This approach is more cumbersome and is generally not used.

### Indirect Locating

#### commit_creds(&init_cred)

The `commit_creds()` function is used to set a new cred as the real_cred and cred fields of the current process's task_struct. Therefore, if we can hijack the kernel execution flow to call this function and pass in a cred with root privileges, we can directly complete the privilege escalation of the current process:

```c
int commit_creds(struct cred *new)
{
	struct task_struct *task = current;// kernel macro, used to get the current process's PCB from the percpu segment
	const struct cred *old = task->real_cred;

	//...
	rcu_assign_pointer(task->real_cred, new);
	rcu_assign_pointer(task->cred, new);
```

During the kernel initialization process, the `init` process is started with root privileges, and its cred structure is the **statically defined** `init_cred`. From this, it is easy to think that we can complete the privilege escalation through `commit_creds(&init_cred)`:

```c
/*
 * The initial credentials for the initial task
 */
struct cred init_cred = {
	.usage			= ATOMIC_INIT(4),
#ifdef CONFIG_DEBUG_CREDENTIALS
	.subscribers		= ATOMIC_INIT(2),
	.magic			= CRED_MAGIC,
#endif
	.uid			= GLOBAL_ROOT_UID,
	.gid			= GLOBAL_ROOT_GID,
	.suid			= GLOBAL_ROOT_UID,
	.sgid			= GLOBAL_ROOT_GID,
	.euid			= GLOBAL_ROOT_UID,
	.egid			= GLOBAL_ROOT_GID,
	.fsuid			= GLOBAL_ROOT_UID,
	.fsgid			= GLOBAL_ROOT_GID,
	.securebits		= SECUREBITS_DEFAULT,
	.cap_inheritable	= CAP_EMPTY_SET,
	.cap_permitted		= CAP_FULL_SET,
	.cap_effective		= CAP_FULL_SET,
	.cap_bset		= CAP_FULL_SET,
	.user			= INIT_USER,
	.user_ns		= &init_user_ns,
	.group_info		= &init_groups,
	.ucounts		= &init_ucounts,
};
```

#### (Obsolete) commit_creds(prepare_kernel_cred(0))

The kernel provides the `prepare_kernel_cred()` function to copy the cred structure of a specified process. When we pass in NULL as the parameter, this function copies `init_cred` and returns a cred with root privileges:

```c
struct cred *prepare_kernel_cred(struct task_struct *daemon)
{
	const struct cred *old;
	struct cred *new;

	new = kmem_cache_alloc(cred_jar, GFP_KERNEL);
	if (!new)
		return NULL;

	kdebug("prepare_kernel_cred() alloc %p", new);

	if (daemon)
		old = get_task_cred(daemon);
	else
		old = get_cred(&init_cred);
```

It is easy to think that if we can call `commit_creds(prepare_kernel_cred(NULL))` in kernel space, we can also directly complete the privilege escalation.

![72b919b7-87bb-4312-97ea-b59fe4690b2e](figure/elevation-of-privilege.png)

However, since kernel version 6.2, `prepare_kernel_cred(NULL)` will **no longer copy init\_cred, but instead treats it as a runtime error and returns NULL**, making this privilege escalation method no longer applicable to kernel version 6.2 and later:

```c
struct cred *prepare_kernel_cred(struct task_struct *daemon)
{
	const struct cred *old;
	struct cred *new;

	if (WARN_ON_ONCE(!daemon))
		return NULL;

	new = kmem_cache_alloc(cred_jar, GFP_KERNEL);
	if (!new)
		return NULL;
```
