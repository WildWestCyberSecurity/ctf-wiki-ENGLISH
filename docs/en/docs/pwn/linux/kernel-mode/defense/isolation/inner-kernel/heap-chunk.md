# Internal Isolation

## Heap Chunk Isolation

### Isolation between GFP\_KERNEL & GFP\_KERNEL\_ACCOUNT

`GFP_KERNEL` and `GFP_KERNEL_ACCOUNT` are the most common and general allocation flags in the kernel. Under normal circumstances, their allocations come from the same `kmem_cache` — the generic `kmalloc-xx`.

Before version 5.9, there was an isolation mechanism between `GFP_KERNEL` and `GFP_KERNEL_ACCOUNT`. The isolation mechanism was removed in [this commit](https://github.com/torvalds/linux/commit/10befea91b61c4e2c2d1df06a2e978d182fcf792), and starting from kernel version 5.14, it was reintroduced in [this commit](https://github.com/torvalds/linux/commit/494c1dfe855ec1f70f89552fce5eadf4a1717552):

- For kernels with the `CONFIG_MEMCG_KMEM` compile option enabled (enabled by default), a **separate set of `kmem_cache` named `kmalloc-cg-*`** will be created for generic objects allocated with `GFP_KERNEL_ACCOUNT`, resulting in isolation between objects using these two flags.

### SLAB_ACCOUNT

According to the description, if the `SLAB_ACCOUNT` flag is passed when creating a cache using `kmem_cache_create`, that cache will exist independently and will not be merged with other caches of the same size.

```
Currently, if we want to account all objects of a particular kmem cache,
we have to pass __GFP_ACCOUNT to each kmem_cache_alloc call, which is
inconvenient. This patch introduces SLAB_ACCOUNT flag which if passed to
kmem_cache_create will force accounting for every allocation from this
cache even if __GFP_ACCOUNT is not passed.

This patch does not make any of the existing caches use this flag - it
will be done later in the series.

Note, a cache with SLAB_ACCOUNT cannot be merged with a cache w/o
SLAB_ACCOUNT, i.e. using this flag will probably reduce the number of
merged slabs even if kmem accounting is not used (only compiled in).
```

In the early days, many structures (such as the **cred structure**) did not have their own dedicated cache and shared a cache with chunks of the same size. After this flag was introduced in Linux 4.5, many structures started using their own dedicated cache. However, according to the description above, this feature was apparently not originally introduced for security purposes.

```
Mark those kmem allocations that are known to be easily triggered from
userspace as __GFP_ACCOUNT/SLAB_ACCOUNT, which makes them accounted to
memcg.  For the list, see below:

 - threadinfo
 - task_struct
 - task_delay_info
 - pid
 - cred
 - mm_struct
 - vm_area_struct and vm_region (nommu)
 - anon_vma and anon_vma_chain
 - signal_struct
 - sighand_struct
 - fs_struct
 - files_struct
 - fdtable and fdtable->full_fds_bits
 - dentry and external_name
 - inode for all filesystems. This is the most tedious part, because
   most filesystems overwrite the alloc_inode method.

The list is far from complete, so feel free to add more objects.
Nevertheless, it should be close to "account everything" approach and
keep most workloads within bounds.  Malevolent users will be able to
breach the limit, but this was possible even with the former "account
everything" approach (simply because it did not account everything in
fact).
```

### References

- https://lore.kernel.org/patchwork/patch/616610/
- https://github.com/torvalds/linux/commit/5d097056c9a017a3b720849efb5432f37acabbac#diff-3cb5667a88a24e8d5abc7042f5c4193698d6b962157f637f9729e61198eec63a
- https://github.com/torvalds/linux/commit/230e9fc2860450fbb1f33bdcf9093d92d7d91f5b#diff-cc9aa90e094e6e0f47bd7300db4f33cf4366b98b55d8753744f31eb69c691016
- https://github.com/torvalds/linux/commit/10befea91b61c4e2c2d1df06a2e978d182fcf792
- https://github.com/torvalds/linux/commit/494c1dfe855ec1f70f89552fce5eadf4a1717552

