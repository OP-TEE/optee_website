---
title: Security Advisories.
permalink: /security-advisories/
layout: jumbotron-container
description: |-
  At this page we will list of all known security vulnerabilities found on OP-TEE.
  Likewise you will find when it was fixed and who reported the issue.

  If you have found a security issue in OP-TEE, please send us an email (see
  About) and then someone from the team will contact you for further discussion.
  The initial email doesn't have to contain any details.
jumbotron:
    triangle-divider: true
    title: Security Advisories
    description: >
        At this page we will list of all known security vulnerabilities found on OP-TEE.
        Likewise you will find when it was fixed and who reported the issue.
    background-image: /assets/images/background-image.jpg
---
At this page we will list of all known security vulnerabilities found on OP-TEE.
Likewise you will find when it was fixed and who reported the issue.

If you have found a security issue in OP-TEE, please send us an email (see
[Contact]) and then someone from the team will contact you for further discussion.
The initial email doesn't have to contain any details.
# May 2019
## Netflix review

### RPC alloc could allocate smaller shared memory than requested
The function thread_rpc_alloc() [used by functions thread_rpc_alloc_payload(),
thread_rpc_alloc_global_payload()] allocates shared memory buffer via RPC. The
required size is copied to the RPC's thread shared memory arg[] from internal
TEE memory params[] in get_rpc_arg(). Then thread_rpc_alloc() calls thread_rpc()
to send the request to the REE.

The REE then process the request, update shared memory arg[] at will, when the
RPC is completed, thread_rpc() returns. thread_rpc_alloc() then call
get_rpc_alloc_res() to parse result from the REE and allocate corresponding
memory object (mobj) in case of success (using buf_ptr and size from arg[]).

Consequences: Resulting allocated memory could be smaller than requested:
Returned memory [PA,SIZE] could be inside shared memory 'NSEC', but if caller
does not check mobj->size, this could be leverage to access invalid or
restricted memory beyond the shared 'NSEC' intended memory; consequence of doing
so depends on memory arrangement and memory map. This could also have other side
effects in the case of attribute [OPTEE_MSG_ATTR_NONCONTIG], like mapping less
pages than expected.

e.g.#1: gprof_send_rpc() uses thread_rpc_alloc_payload(SIZE), then use returned
mobj to get the virtual address, but assume the requested SIZE is served, and
does not check the actual size of the allocated memory object, and uses memcpy()
over it.

e.g.#2: ta_load() uses thread_rpc_alloc_payload() to allocate space for a TA
binary, then get the virtual address (ta_buf) from the associated mobj without
further checks on the size. ta_buf is later used assumed to be of a given size.

e.g.#3: tee_fs_rpc_cache_alloc() uses thread_rpc_alloc_payload(), then get the
va from the mobj, but the size is the one given in argument, and could be
smaller than the one in the mobj. This API is used in various places.

e.g.#4 tee_rpmb_*()->tee_rpmb_alloc()->thread_rpc_alloc_payload(), the va from
the mobj could point to a smaller allocated block than required.

**optee_os.git:**
 - [core: verify size of allocated shared memory (cc6bc5f9421)](
https://github.com/OP-TEE/optee_os/commit/cc6bc5f94210ea24b774c997fd482c936735db71)

| Reported by                 | CVE ID    | OP-TEE ID        | Affected versions  |
| --------------------------- | :-------: | :--------------: | ------------------ |
| [Netflix] (Bastien Simondi) | Not/Ready | OP-TEE-2019-0001 | v3.5.0 and earlier |


### Poison or leak shared secure memory allocated in the kernel
Memory allocated through alloc_temp_sec_mem() is not scrubbed when returned. One
could leverage this to copy arbitrary data into this secure memory
pool or to snoop former data from a previous call done by another TA (e.g. using
TEE_PARAM_TYPE_MEMREF_OUTPUT allows to map the data while not overwriting it,
hence accessing to what is already there).

**optee_os.git:**
 - [core: scrub user-tainted memory returned by alloc_temp_sec_mem() (93488596e6)](
https://github.com/OP-TEE/optee_os/commit/934885496e6db0b7c4c8ca28a1aee6696dcc72a6)

| Reported by                 | CVE ID    | OP-TEE ID        | Affected versions  |
| --------------------------- | :-------: | :--------------: | ------------------ |
| [Netflix] (Bastien Simondi) | Not/Ready | OP-TEE-2019-0002 | v3.5.0 and earlier |

### Poison kernel heap memory
syscall_log(), syscall_open_ta_session(), syscall_get_property()
... can be used to poison kernel heap memory. Data copied from userland is not
scrubbed when the syscall returns. e.g. when doing syscall_log() one can copy
arbitrary data of variable length onto kernel memory. When free() is called, the
block is returned to the memory pool, tainted with that userland data.

**optee_os.git:**
 - [core: scrub user-tainted kernel heap memory before freeing it (70b613102c)](
https://github.com/OP-TEE/optee_os/commit/70b613102ce72808f6a0ad9f6f97f0545fd6ad02)

| Reported by                 | CVE ID    | OP-TEE ID        | Affected versions  |
| --------------------------- | :-------: | :--------------: | ------------------ |
| [Netflix] (Bastien Simondi) | Not/Ready | OP-TEE-2019-0003 | v3.5.0 and earlier |

### Integer overflow on parameters from REE leads to smaller memory object than expected
msg_param_mobj_from_noncontig() does not check that buf_ptr+size do
not overflow. As a result, num_pages could be computed small, while size could
be big. Only 'num_pages' will be mapped/registered in the returned mobj. If the
caller does not compare mobj->size with required size, it can end-up
manipulating memory out of the intended region. Same rational as for 1.1 apply,
one could use it to make the caller accessing memory beyond the shared memory.

e.g. set_tmem_param()->msg_param_mobj_from_nonconfig() takes the returned mobj
as it is, and associate to it the size that came from the REE without checking
the actual mobj->size. The outcome of this depends on memory arrangement and TA
implementation.

e.g. tee_entry_std()->register_shm()->msg_param_mobj_from_noncontig(). This path
should not be problematique for the discussed use case where num_pages is
smaller than expected, but the buffer overflow may have other attack use cases.

e.g. thread_rpc_alloc()->get_rpc_alloc_res()->msg_param_mobj_from_noncontig().
This path provides a similar attack as discussed in "RPC alloc could allocate
smaller shared memory than requested", where the resulting mobj->size is smaller
that requested/expected.

**optee_os.git:**
 - [core: check for overflow in msg_param_mobj_from_noncontig() (e1509d6e61)](
https://github.com/OP-TEE/optee_os/commit/e1509d6e6178011df581c535ee8bf8c147053df2)

| Reported by                 | CVE ID    | OP-TEE ID        | Affected versions  |
| --------------------------- | :-------: | :--------------: | ------------------ |
| [Netflix] (Bastien Simondi) | Not/Ready | OP-TEE-2019-0004 | v3.5.0 and earlier |

### TA binaries should be authenticated before doing operations based on their content
Loading a TA, which is an ELF binary, triggers a lot of operations done on the
content of the ELF structure/info which are under REE control. It would be
preferable to fully authenticate the binary before doing any of those ELF
related operations (loading segments etc). This will not prevent the ELF
manipulation from going wrong, but it will move the threat from anyone on the
REE accessing the binaries, to restricted people having a valid key to sign a
binary.

e.g. for REE FS TA storage, the hash is verified only when full binary have been
consumed, which is at the end of the ELF binding process. See
ree_fs_ta.c::ta_read().

Additionally, this is a weak spot: a lot of operations rely on getting there, if
the check_digest() never happens for X reason, the caller will just keep going
(effectively allowing a non-authenticated TA to run). No attack path where
identified, but some may exist or may be introduced to alter either the
handle->nw_ta_size in ree_fs_ta.c, or the state->data_len in elf_load.c allowing
to bypass the digest check.

**optee_os.git:**
 - [core: REE FS TAs: add option to verify signature before processing (7db24ad625)](
https://github.com/OP-TEE/optee_os/commit/7db24ad625b91a7f4f16c33b7c825cd56952a8cf)

| Reported by                 | CVE ID    | OP-TEE ID        | Affected versions  |
| --------------------------- | :-------: | :--------------: | ------------------ |
| [Netflix] (Bastien Simondi) | Not/Ready | OP-TEE-2019-0005 | v3.5.0 and earlier |

### Constant time memory compare function
A constant time memory compare function should be available for the Trusted
Applications in the TEE/TA API.

**optee_os.git:**
 - [libutee: TEE_MemCompare(): use constant time algorithm (65551e69a0)](
https://github.com/OP-TEE/optee_os/commit/65551e69a006c496fb18d8374389b7b3617c2076)

| Reported by                 | CVE ID    | OP-TEE ID        | Affected versions  |
| --------------------------- | :-------: | :--------------: | ------------------ |
| [Netflix] (Bastien Simondi) | Not/Ready | OP-TEE-2019-0006 | v3.5.0 and earlier |

### Overflows during RPMB operations
During the RPMB initialization process, the TEE request the REE for
some of the device information
[tee_rpmb_init()->tee_rpmb_get_dev_info()->tee_rpmb_invoke()] The returned
information from the REE (struct rpmb_dev_info) is not checked and some of the
fields are used in multiplication, also used to compute rpmb_ctx->rel_wr_blkcnt
which could end up being 0 or very large.

At run time, controlling rel_wr_blkcnt is also important as it leads to better
control in sub functions used later in the code, like inside
tee_rpmb_write_blk(). All of the operations should there be under tight control
and make use of the xxx_OVERFLOW() macros as much as possible, this includes
computing blkcnt, req_size, tmp_blkcnt, ...

**optee_os.git:**
 - [core: RPMB FS: check for potential overflows (ea81076f78)](
https://github.com/OP-TEE/optee_os/commit/ea81076f7896de7278dcd62b47b99d5dc3351caf)

| Reported by                 | CVE ID    | OP-TEE ID        | Affected versions  |
| --------------------------- | :-------: | :--------------: | ------------------ |
| [Netflix] (Bastien Simondi) | Not/Ready | OP-TEE-2019-0007 | v3.5.0 and earlier |

### Use of arbitrary virtual address in TEE crypto service
syscall_authenc_init() dos not check that the given nonce address
is within TA accessible memory.

**optee_os.git:**
 - [core: syscall_authenc_init(): check nonce accessibility (06aa9a9b41)](
https://github.com/OP-TEE/optee_os/commit/06aa9a9b4117a045197c39ba9754422ce0593c0f)

| Reported by                 | CVE ID    | OP-TEE ID        | Affected versions  |
| --------------------------- | :-------: | :--------------: | ------------------ |
| [Netflix] (Bastien Simondi) | Not/Ready | OP-TEE-2019-0008 | v3.5.0 and earlier |

### Integer overflows in TEE crypto service
There is a risk of integer overflowsa in the following locations.
- copy_in_attrs(): if a very large 'attr_count' is given, the following
  operation overflows: "attr_count * sizeof(struct utee_attribute)"
- syscall_cryp_obj_populate(): if a very large 'attr_count' is given, the
  following operation overflows "sizeof(TEE_Attribute) * attr_count"
- syscall_asymm_verify(), syscall_asymm_operate(): if a very large 'num_params'
  is given, the following operation overflows "sizeof(TEE_Attribute) *
  num_params"
- syscall_cryp_derive_key(), syscall_obj_generate_key(): if a very large
  'param_count' is given, the following operation overflows
  "sizeof(TEE_Attribute) * param_count"
- syscall_cryp_derive_key(): if a very large 'params[0].content.ref.length' is
  given, the following overflows "params[0].content.ref.length * 8" [this is
  probably not realistic as params[0].content.ref.len is checked to some extend
  during attrs copy]

**optee_os.git:**
 - [core: crypto: add overflow check when copying attributes (bd81e5b95e)](
https://github.com/OP-TEE/optee_os/commit/bd81e5b95ec910e9e3fa9f1824f3981288af5d50)

| Reported by                 | CVE ID    | OP-TEE ID        | Affected versions  |
| --------------------------- | :-------: | :--------------: | ------------------ |
| [Netflix] (Bastien Simondi) | Not/Ready | OP-TEE-2019-0009 | v3.5.0 and earlier |

### Copying memory with source and destination overlap should use memmove(),merged offset may be wrong.
get_elf_segments() final stage tries to aggregate segments. Inside
the "while (idx < num_segs)" loop, the logic to remove the current index is to
run a memcpy() to shift down everything beyond that point, basically 'moving'
down the rest of the segments.

**optee_os.git:**
 - [core: get_elf_segments(): use memmove on overlapping memory (3bcb882f20)](
https://github.com/OP-TEE/optee_os/commit/3bcb882f200c2dd14ea1937031d5bd97bf6a78ca)

| Reported by                 | CVE ID    | OP-TEE ID        | Affected versions  |
| --------------------------- | :-------: | :--------------: | ------------------ |
| [Netflix] (Bastien Simondi) | Not/Ready | OP-TEE-2019-0010 | v3.5.0 and earlier |

### core: load_elf_from_store(): check stack size
ROUNDUP operations while adding the stack segment could overflow.

Inside load_elf_from_store(), the ta_head structure is retrieved from
un-authenticated area, and contains the stack size. The stack size could either
already be 0, or could be large enough so it become 0 when rounded up to
STACK_ALIGNMENT.

When allocating the memory using alloc_ta_mem(), which can allocate a 0 bytes
size memory block.

The code then call vm_map() to actually add the stack segment, and provide the
stack_size (which can be 0, or very large if CFG_PAGED_USER_TA is used) as an
argument. vm_map() logic round up the size again to SMALL_PAGE_SIZE (which is
larger than STACK_ALIGNMENT). Again here, the size could either already be 0, or
end-up being 0.

vm_map() will in either way return a virtual address to the caller for this 0
bytes memory block. Now there is a disconnection between, ta_head->stack_size,
mobj_stack->size and reg->size=3D0, which all 3 could contain different values.
Consequence on having a disconnection between the various values, or having a 0
bytes stack size has not been analyzed.

**optee_os.git:**
 - [core: load_elf_from_store(): check stack size (b17e2e4444)](
https://github.com/OP-TEE/optee_os/commit/b17e2e44441a6b8233d5e2bdccdac4ec23a0e819)

| Reported by                 | CVE ID    | OP-TEE ID        | Affected versions  |
| --------------------------- | :-------: | :--------------: | ------------------ |
| [Netflix] (Bastien Simondi) | Not/Ready | OP-TEE-2019-0011 | v3.5.0 and earlier |

### SHDR_GET_*() macros integer overflow
The SHDR_GET_SIZE(), SHDR_GET_HASH() and SHDR_GET_SIG() macros
could overflow depending on given 'x' and associated hash_size, sig_size values.

Note: no other attack path than the ones reported by Riscure (not using keyword
'return' when checking img_size and shdr_size) are identified to exploit this
overflow, however it is error prone and could lead to a future vulnerability.

**optee_os.git:**
 - [core: add VA overflow check in shdr_alloc_and_copy() (062765e4f8)](
https://github.com/OP-TEE/optee_os/commit/062765e4f80b97c90fd62d17859b675797af5de9)

| Reported by                 | CVE ID    | OP-TEE ID        | Affected versions  |
| --------------------------- | :-------: | :--------------: | ------------------ |
| [Netflix] (Bastien Simondi) | Not/Ready | OP-TEE-2019-0012 | v3.5.0 and earlier |

### MOBJ_REG_SHM_SIZE() integer overflow
The macro MOBJ_REG_SHM_SIZE() could overflow depending on
'nr_pages'.

e.g. mobj_mapped_shm_alloc()->mobj_reg_shm_alloc() called in various places.

In such case, the mobj_reg_shm memory would be a small memory block, while
num_pages would be large, which could lead to a generous memcpy() when copying
the pages in internal memory, the outcome of this depends on memory mapping.

Note: no attack path are identified to exploit this overflow, however it is
error prone and could lead to a future vulnerability.

**optee_os.git:**
 - [core: add overflow check in mobj_reg_shm_alloc() (8ad7af5027)](
https://github.com/OP-TEE/optee_os/commit/8ad7af50273124c3cf043a798df633ae2c388913)

| Reported by                 | CVE ID    | OP-TEE ID        | Affected versions  |
| --------------------------- | :-------: | :--------------: | ------------------ |
| [Netflix] (Bastien Simondi) | Not/Ready | OP-TEE-2019-0013 | v3.5.0 and earlier |

### Virtual address returned to the REE
session context virtual address is returned to the REE in
entry_open_session(); it is then used back in entry_close_session() and
entry_invoke_command().

Sharing virtual addresses with the REE leads to virtual memory addresses
disclosure that could be leverage to defeat ASLR and/or mount an attack.
Exchanging virtual addresses between REE and TEE is generally a bad idea, it
discloses TEE internal virtual addresses and flows info which could lead to
future vulnerabilities if any error is made while verifying or manipulating the
exchanged virtual address.

Additionally, a vaddr_t is used to carry the virtual address, which on a 64bits
could overflow/swap as the session 'id' is a uint32_t [see tee_ta_get_session()]
and have other side-effects on the execution (being non-unique | N to 1).

**optee_os.git:**
 - [core: do not use virtual addresses as session identifier (99164a05ff)](
https://github.com/OP-TEE/optee_os/commit/99164a05ff515a077ff0f3e1550838d24623665b)

| Reported by                 | CVE ID    | OP-TEE ID        | Affected versions  |
| --------------------------- | :-------: | :--------------: | ------------------ |
| [Netflix] (Bastien Simondi) | Not/Ready | OP-TEE-2019-0014 | v3.5.0 and earlier |

### Integer overflow could lead to too large num_syms and rel_end during relocation process
(shdr[sym_tab_idx].sh_addr + shdr[sym_tab_idx].sh_size) and
(shdr[rel_sidx].sh_addr + shdr[rel_sidx].sh_size) could overflow, resulting in a
large num_syms, or an invalid rel_end. Both could be used to access beyond
legitimate memory. Outcome of such flaw is unclear but could be used to snoop or
alter memory.

**optee_os.git:**
 - [core: ELF relocation: use ADD_OVERFLOW() (781c8f007c)](
https://github.com/OP-TEE/optee_os/commit/781c8f007c4b4ac1d1966f1aacd37fbfe5d628aa)

| Reported by                 | CVE ID    | OP-TEE ID        | Affected versions  |
| --------------------------- | :-------: | :--------------: | ------------------ |
| [Netflix] (Bastien Simondi) | Not/Ready | OP-TEE-2019-0015 | v3.5.0 and earlier |

### ehdr.e_shnum could be very large and used to access out of bound memory
At the end of elf_load_body(), code process relocation information
segment: the relocation information are copied in a system heap memory block,
associated to state->shdr. As the computed size is the result of an uncontrolled
multiplication (ehdr.e_shnum * ehdr.e_shentsize), it could have overflowed and
result in allocating a small memory block.

Later in the code, there are no MUL_OVERFLOW() check either performed, and the
code will access beyond the allocated memory. (e.g. in elf_process_rel() ) The
outcome of this flaw depends on system heap memory arrangement but could be used
to snoop information or alter memory.

e.g. e32_process_rel() retrieved sym_tab_idx from shdr[rel_sidx].sh_link, then
check if sym_tab_idx is smaller than ehdr->e_shnum; as the later can be as large
as needed, shdr[sym_tab_idx] can point beyond the allocated memory block.
Additionally, depending on memory arrangement, one could point to shared memory,
and control the targeted memory block to mount further TOCTOU attacks (for
instance to set sym_tab to an arbitrary value by updating
shdr[sym_tab_idx].sh_addr with proper timing).

**optee_os.git:**
 - [core: elf_load_body(): use MUL_OVERFLOW() to get size of section headers (5787ecdf75)](
https://github.com/OP-TEE/optee_os/commit/5787ecdf758d9edbfb5fb93c49c808f7a51a214b)

| Reported by                 | CVE ID    | OP-TEE ID        | Affected versions  |
| --------------------------- | :-------: | :--------------: | ------------------ |
| [Netflix] (Bastien Simondi) | Not/Ready | OP-TEE-2019-0016 | v3.5.0 and earlier |

### Integer overflow in TA memory map logic
vm_map() and umap_add_region() do not check that given offs +
ROUNDUP(len...) do not overflow. As a result the check to see if the region is
in within a given memory object can be bypassed and both offset and/or size
parameters could be very large.

This could be leverage to alter the intended behavior of functions using either
the region size or the region offset, like tee_mmu_user_pa2va_helper() for
instance.

**optee_os.git:**
 - [core: umap_add_region(): add overflow check (bcc81cf8f0)](
https://github.com/OP-TEE/optee_os/commit/bcc81cf8f0ec93c62ff5bc1b1c3d09e50cc2525f)

| Reported by                 | CVE ID    | OP-TEE ID        | Affected versions  |
| --------------------------- | :-------: | :--------------: | ------------------ |
| [Netflix] (Bastien Simondi) | Not/Ready | OP-TEE-2019-0017 | v3.5.0 and earlier |


# October 2018
## Riscure mini-audit

### Integer overflow in crypto system calls (x2) - part 2
The function `syscall_asymm_verify` is a system call used to verify
cryptographic signatures. One of the parameters passed in by a TA is
`num_params`. The TEE kernel locally allocates a heap buffer of size
`sizeof(TEE_Attribute) * num_params` without checking for an integer overflow
in the multiplication. The lack of checking can result in a smaller heap buffer
than required. The user supplied input `usr_params` is then copied into this
buffer, but making the additional checks in `copy_in_attrs` fail can be used to
terminate the copy at any moment. This allows a heap based buffer overflow with
attacker controlled data written outside the boundaries of the buffer. Such
corruption might allow code execution in the context of the TEE kernel.

**optee_os.git:**
 - [svc: check for allocation overflow in crypto calls part 2 (70697bf3c5d)](
https://github.com/OP-TEE/optee_os/commit/70697bf3c5dc3d201341b01a1a8e5bc6d2fb48f8)

| Reported by  | CVE ID    | OP-TEE ID        | Affected versions  |
| ------------ | :-------: | :--------------: | ------------------ |
| [Riscure]    | Not/Ready | OP-TEE-2018-0011 | v3.3.0 and earlier |

### Integer overflow in crypto system calls (x2)
The function `syscall_obj_generate_key` is a system call which generates a
cryptographic key. This system call is exposed to TAs which supply the length
of the key to be generated, its type, and a number of attributes it should
have. A multiplication operation involving the number of parameters is not
checked for overflow which can lead to an out-of-bounds write. One of the
parameters passed in by a TA is `param_count`. The TEE kernel locally allocates
a heap buffer of size `sizeof(TEE_Attribute) * param_count`, without checking
for an integer overflow in the multiplication. The lack of checking can result
in a smaller heap buffer than required. The user supplied input `usr_params` is
then copied into this buffer, but making the additional checks in
`copy_in_attrs` fail can be used to terminate the copy at any moment. This
allows a heap based buffer overflow with attacker controlled data written
outside the boundaries of the buffer. Such corruption might allow code
execution in the context of the TEE kernel.

**optee_os.git:**
 - [svc: check for allocation overflow in crypto calls (a637243270f)](
https://github.com/OP-TEE/optee_os/commit/a637243270fc1faae16de059091795c32d86e65e)

| Reported by  | CVE ID    | OP-TEE ID        | Affected versions  |
| ------------ | :-------: | :--------------: | ------------------ |
| [Riscure]    | Not/Ready | OP-TEE-2018-0010 | v3.3.0 and earlier |

### Integer overflow in crypto system calls
The function `syscall_cryp_obj_populate` is a system call which initializes the
attributes of a cryptographic object. This system call is exposed to TAs which
supply a reference to the crypto object to be populated along with a number of
attributes it must posses. The number of attributes is used as part of the
multiplication to allocate memory. It is not checked for an overflow, which can
lead to an out-of-bounds write. One of the parameters passed in by a TA is
`attr_count`. The TEE kernel locally allocates a heap buffer of size
`sizeof(TEE_Attribute) * attr_count` without checking for an integer overflow
in the multiplication. The lack of checking can result in a smaller heap buffer
than required. The user supplied input `usr_attrs` is then copied into this
buffer, but making the additional checks in `copy_in_attrs` fail can be used to
terminate the copy at any moment. This allows a heap based buffer overflow with
attacker controlled data written outside the boundaries of the buffer. Such
corruption might allow code execution in the context of the TEE kernel.

**optee_os.git:**
 - [svc: check for allocation overflow in syscall_cryp_obj_populate (b60e1cee406)](
https://github.com/OP-TEE/optee_os/commit/b60e1cee406a1ff521145ab9534370dfb85dd592)

| Reported by  | CVE ID    | OP-TEE ID        | Affected versions  |
| ------------ | :-------: | :--------------: | ------------------ |
| [Riscure]    | Not/Ready | OP-TEE-2018-0009 | v3.3.0 and earlier |

### Buffer checks missing when calling pseudo TAs
The function `tee_svc_copy_param` is used to copy in parameters when a TA wants
to open a session with or invoke a command upon another TA. It is used in
system calls and is therefore indirectly callable by any TA. However, this
function does not do sufficient parameter checking when the called TA is a
pseudo TA. One of the parameters passed in is `callee_params` which is passed
directly through from the TA. It is verified that this structure itself resides
in either shared memory or memory which the calling TA has read access to.
However, this structure can contain pointers as its members. The structure
`callee_params` is first copied into the output parameter param. In the case
that the called TA is a pseudo TA no further checking is done and a success
code is returned. It is not verified that the members of param point to valid
memory. This means there is a mismatch between the validation performed when
invoking a normal TA and when invoking a pseudo TA. If a pseudo TA relies on
the pointers being validated as it would be for a normal TA, it might use these
pointers without further validation. This might result in memory corruption and
memory disclosure.

**optee_os.git:**
 - [core: svc: always check ta parameters (d5c5b0b77b2)](
https://github.com/OP-TEE/optee_os/commit/d5c5b0b77b2b589666024d219a8007b3f5b6faeb)

| Reported by  | CVE ID    | OP-TEE ID        | Affected versions  |
| ------------ | :-------: | :--------------: | ------------------ |
| [Riscure]    | Not/Ready | OP-TEE-2018-0007 | v3.3.0 and earlier |

### Potential disclosure of previously loaded TA code and data
The function `elf_load_body` is used to load the code and data segments while
dynamically loading a TA. The amount of memory allocated for the code and data
segments is previously determined and the sum of it is stored in
`state->vasize`. The actual allocated amount of memory is rounded up the next
multiple of the memory pool granularity. To ensure that the newly loaded TA is
not able to observe any data belonging to a TA previously stored on this exact
location in memory, the memory block is set to zero. The size used to memset
the block to zero is the sum of the sizes of the segments, not the rounded size
of the actual allocation. This means that the remaining space at the end of the
allocation is not cleared, potentially leaking code and/or data of a previous
TA. The information gained by this attack is limited by the memory layout of
the (compromised) TA performing the attack and the flags (i.e. is unloading the
TA prevented when the last session is closed due to
`TA_FLAG_INSTANCE_KEEP_ALIVE`) and layout of the attacked TA.

**optee_os.git:**
 - [core: clear the entire TA area (7e768f8a473)](
https://github.com/OP-TEE/optee_os/commit/7e768f8a473409215fe3fff8f6e31f8a3a0103c6)

| Reported by  | CVE ID    | OP-TEE ID        | Affected versions  |
| ------------ | :-------: | :--------------: | ------------------ |
| [Riscure]    | Not/Ready | OP-TEE-2018-0006 | v3.3.0 and earlier |

### tee_mmu_check_access_rights does not check final page of TA buffer
The function `tee_mmu_check_access_rights` is used to check access rights to a
given memory region. This function is used when a TA performs a system call to
verify that the TA has the correct access rights to the buffer it provides.
However, the function `tee_mmu_check_access_rights` does not check every page of
the TA provided buffer. A TA provides a buffer as a pointer (uaddr) and a
length (a). The provided buffer is checked piecewise in increments of addr_incr
(4KiB) in a for-loop. In the case where len is not already page aligned, the
termination condition a < (uaddr + len) has been passed when addr_incr is added
the last time of the loop iteration. Therefore, the final page of the TA
provided buffer is not checked. A TA could provide a buffer of which up to 4KiB
resides in the context of the TEE kernel or another TA. This could lead to
memory corruption of the TEE itself or another TA. Memory corruption
vulnerabilities can have serious impact such as allowing runtime control.

**optee_os.git:**
 - [core: tee_mmu_check_access_rights() check all pages (95f36d661f2)](
https://github.com/OP-TEE/optee_os/commit/95f36d661f2b75887772ea28baaad904bde96970)

| Reported by  | CVE ID    | OP-TEE ID        | Affected versions  |
| ------------ | :-------: | :--------------: | ------------------ |
| [Riscure]    | Not/Ready | OP-TEE-2018-0005 | v3.3.0 and earlier |

### Unchecked parameters are passed through from REE
The function `set_rmem_param` is a helper function used when copying parameters
locally for TA calls. It is used when a parameter is a buffer of type rmem. The
function receives an input parameter param from the REE and an output parameter
mem. After finding the shared memory object referenced by param the offset and
size members of param are copied into mem as is. There is no validation done to
ensure that these members actually do reside in shared memory. There is no
further checking done on param before it gets passed on to the TA through the
function `pointer sess->ctx->ops->enter_invoke_cmd` in the function
tee_ta_invoke_command. How this problem manifests itself is very dependent on
how the passed parameters are used by the TA. However, it could lead to
corruption of any memory which the TA can access.

**optee_os.git:**
 - [core: ensure that supplied range matches MOBJ (e3adcf566cb)](
https://github.com/OP-TEE/optee_os/commit/e3adcf566cb278444830e7badfdcc3983e334fd1)

| Reported by  | CVE ID    | OP-TEE ID        | Affected versions  |
| ------------ | :-------: | :--------------: | ------------------ |
| [Riscure]    | Not/Ready | OP-TEE-2018-0004 | v3.3.0 and earlier |

# May 2018
## Spectre variant 4 (CVE-2018-3639)
#### Current status:
In the affected Arm cores (Cortex-A57, Cortex-A72, Cortex-A73 and Cortex-A75)
who all are Armv8 based there are configuration control registers
available at EL3 that when enabled effectively mitigate a potential
Spectre v4 attack. This means that the mitigation for this is not being
implemented at S-EL1 where the TEE resides. For more information about
the EL-3 mitigations, please see the [Trusted Firmware A Security Advisory TFV 7](https://github.com/ARM-software/arm-trusted-firmware/wiki/Trusted-Firmware-A-Security-Advisory-TFV-7).
In all officially supported Armv8 OP-TEE setups we are using TF-A as
the firmware and therefore we consider that the TF-A mitigations at
EL-3 effectively stop Spectre v4 attacks in a system running OP-TEE and
TF-A.

As mentioned in the
[whitepaper](https://developer.arm.com/support/arm-security-updates/speculative-processor-vulnerability)
from Arm about these types of attacks, there are new barriers (SSBB and PSSBB)
being introduced also. These could also be used as a mitigation
directly at lower exception levels. But just as for Spectre v1, this
involves manual inspection of code and placement of barriers until tooling
has become better to figure out this on its own. This manual work is
error prone and very time consuming and has to be done over and over
again. We have been doing some manual inspection of the OP-TEE code and
so far have not been able to identify and vulnerable areas. But just as
for Spectre v1, we continuously discuss tooling etc with members of Linaro.

| Reported by  | CVE ID | OP-TEE ID | Affected versions |
| ------------ |:------:| :-------: | ----------------- |
| [Google Project Zero] | [CVE-2018-3639](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2018-3639) | N/A | N/A (EL-3 TF-A implements the mitigation) |


# January 2018
## Meltdown and Spectre
In collaboration a group of different people (see "Reported by") have found out
that it is possible to circumvent security countermeasures and privilege
escalation by using speculative execution, caches, out-of-order execution in a
specific way. All details about the attacks has been thoroughly described in
the whitepapers that can found in the [Meltdown and Spectre] page. A
[blog post](https://www.linaro.org/blog/meltdown-spectre/) is also available on
the Linaro website. So we will not cover the details here, instead we will
highlight how it could affect OP-TEE and what the mitigations are.

### Variant 1: bounds check bypass (CVE-2017-5753)

#### Possible attack
Since user data provided to Trusted Applications most often comes from
non-secure side, it is important to check the code where we are using those
non-secure parameters. The same type of checks are necessary when doing
syscalls from Trusted Applications. In principle, this means that non-secure
side eventually could access secure memory when untrusted value is passed to
secure side.

#### Current status:
We have been doing some manual inspection of the OP-TEE code, and so far have
not been able to identify any vulnerable areas. Code analysis tools and
compiler update are being discussed with members of Linaro.

| Reported by  | CVE ID | OP-TEE ID | Affected versions |
| ------------ |:------:| :-------: | ----------------- |
| [Google Project Zero], [University of Pennsylvania], [University of Maryland], [Rambus], [Graz University of Technology], [University of Adelaide], [Data61] | [CVE-2017-5753](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-5753) | OP-TEE-2018-0001 | All versions |

### Variant 2: branch target injection (CVE-2017-5715)

#### Possible attack
In theory it would be possible for a program in non-secure world to train the
branch predictor to trick the secure monitor to speculatively read secure
memory and as a consequence of that leak information to the cache that can be
observed by a less privileged process. To exploit this an attacker needs to
find a gadget that can be used as a trampoline to get access kernel memory
(from a Trusted Application for example).

The mitigation here is to invalidate the branch predictor when:
* Going from non-secure to the secure environment.
* When doing syscall from S-EL0 to S-EL1.

#### Current status:
For `Armv8-A` builds we are typically running OP-TEE with Arm Trusted Firmware,
patches can be found here:
* https://github.com/ARM-software/arm-trusted-firmware/pull/1214 (merged)

For builds where we are not using Arm TF (typically `Armv7-A` builds) we have
implemented mitigations that can be found here:
* https://github.com/OP-TEE/optee_os/pull/2047 (merged)
* https://github.com/OP-TEE/optee_os/pull/2065 (merged)

For SVC calls, we have patches here:
* https://github.com/OP-TEE/optee_os/pull/2055 (`Armv7-A`, `AArch32`) (merged)
* https://github.com/OP-TEE/optee_os/pull/2072 (`AArch64`) (merged) and
https://github.com/OP-TEE/optee_os/pull/2229 (`AArch64`) (merged)

| Reported by  | CVE ID | OP-TEE ID | Affected versions |
| ------------ |:------:| :-------: | ----------------- |
| [Google Project Zero], [University of Pennsylvania], [University of Maryland], [Rambus], [Graz University of Technology], [University of Adelaide], [Data61] | [CVE-2017-5715](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-5715) | OP-TEE-2018-0002 | All versions prior to OP-TEE 3.0.0 (32 bits) or 3.1.0 (64 bits) |

### Variant 3: rogue data cache load (CVE-2017-5754)

#### Possible attack
Just as in Linux kernel it could be possible to do the same type of attack from
a Trusted Application as being described in the [Meltdown whitepaper]. I.e.,
under some conditions the CPU would read and execute instructions speculatively
before the CPU handles the illegal access (traps).

#### Current status:
Our patches can be found here:
* https://github.com/OP-TEE/optee_os/pull/2048 (merged)

The mitigation ideas are the same as with [KPTI], i.e, we keep the amount of
kernel memory being mapped to a minimum when running in usermode. It should
also be noted that there are currently no known devices running OP-TEE who are
susceptible to the Meltdown attack. Still we have decided to move on and merged
the mitigation patches, since we believe that this gives additional security and
it also means that we are prepared if/when we find OP-TEE running on
Cortex-A75.

| Reported by  | CVE ID | OP-TEE ID | Affected versions |
| ------------ |:------:| :-------: | ----------------- |
| [Google Project Zero], [Cyberus Technology], [Graz University of Technology] | [CVE-2017-5754](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-5754) | OP-TEE-2018-0003 | All versions prior to OP-TEE 3.0.0 |

# December 2016
## RSA key leakage in modular exponentiation
#### Description
[Applus+ Laboratories] found out that OP-TEE is vulnerable to a timing attack
when doing the [Montgomery operations](http://www.phedny.net/papers/Timing%20attacks%20on%20RSA.pdf).

One way to optimize modular exponentiation is to make use of something called
Montgomery multiplication and Montgomery reduction. OP-TEE implements the
Montgomery operations in the big number library, libmpa. The current
implementation uses a binary Left to Right (LtoR) implementation. The LtoR
implementation is vulnerable to timing attacks since it leaks information about
the exponent in use, because it uses different amount of time in each loop when
doing the exponentiation. The leaked information can be used to completely
recover the private key. One mitigation to this attack is to change the
implementation to a constant time exponentiation algorithm instead of LtoR. One
such algorithm is the so called [Montgomery powering
ladder](https://cr.yp.to/bib/2003/joye-ladder.pdf), which does the same amount
of operations in every loop. I.e., it will always do square and multiply in
every loop. The fix (Montgomery ladder) for the timing attack has been
implemented in:

**optee_os.git:**
- [libmpa: Implement Montgomery ladder
(40b1b281a6)](https://github.com/OP-TEE/optee_os/commit/40b1b281a6f85f8658be749dc92b57d6a8bd5e78)

| Reported by  | CVE ID | OP-TEE ID | Affected versions |
| ------------ |:------:| :-------: | ----------------- |
| [Applus+ Laboratories] | [CVE-2017-1000413](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-1000413) | OP-TEE-2016-0003 | All versions prior to OP-TEE 2.5.0 |


## Bellcore attack
#### Description
[Applus+ Laboratories] found out that OP-TEE is vulnerable to the [Bellcore
attack](https://eprint.iacr.org/2012/553.pdf) when using fault injection /
glitching attack.

A common way to speed up RSA calculations is to use something that is called
[Chinese Remainder Theorem (CRT)](https://en.wikipedia.org/wiki/Chinese_remainder_theorem).
This optimization is also used in LibTomCrypt which is currently the default
software crypto library in OP-TEE. In short, when using CRT you are operating on
the individual prime factors 'p' and 'q' separately and then later combine them
to final result instead of just doing the exponentiation directly. However, this
also means that if somethings goes wrong in the intermediate calculations with
'p' or 'q' it is possible to completely recover the private key if you also have
access to a valid signature. I.e. it's the combination of valid and invalid
signature that makes it possible to recover the private key.

The important thing is to never ever return any incorrect signature back to
the caller. LibTomCrypt already has mitigations for this. They have the flag
`LTC_RSA_CRT_HARDENING` which enables code that checks that the signature indeed
is valid before returning it to the user. Then there is also the flag
`LTC_RSA_BLINDING` which mixes in another random prime number when doing the
intermediate calculations. OP-TEE hasn't had those flags enabled by default in
the past and when enabling them there was some code missing related to random
number generation for big number (mpanum). The fixes for this issue can be found
in:

**optee_os.git:**
 - [ltc: Implement mp_rand for mpa_desc
(13c9b83130)](https://github.com/OP-TEE/optee_os/commit/13c9b83130e08ddd53fb3a456a678c7e3040deb9)
 - [ltc: Enable RSA_CRT_HARDENING and RSA_CRT_BLINDING
(93b0a7015c)](https://github.com/OP-TEE/optee_os/commit/93b0a7015c46d68f2bc8d1bc6c57bb6532269777).

The fix can be found in OP-TEE starting from v2.5.0.

| Reported by  | CVE ID | OP-TEE ID | Affected versions |
| ------------ |:------:| :-------: | ----------------- |
| [Applus+ Laboratories] | [CVE-2017-1000412](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-1000412) | OP-TEE-2017-0002 | All versions prior to OP-TEE 2.5.0 |


# June 2016
## Bleichenbacher signature forgery attack
#### Description
A vulnerability in the [OP-TEE] project was found by Intel Security Advanced
Threat Research in June 2016. It appeared that OP-TEE was vulnerable to
[Bleichenbacher signature forgery attack](https://www.ietf.org/mail-archive/web/openpgp/current/msg00999.html).

The problem lies in the [LibTomCrypt] code in OP-TEE, that neglects to check
that the message length is equal to the ASN.1 encoded data length. Upstream
LibTomCrypt already had a
[fix](https://github.com/libtom/libtomcrypt/commit/5eb9743410ce4657e9d54fef26a2ee31a1b5dd0)
and there was also a [test
case](https://github.com/libtom/libtomcrypt/commit/d51715db728d99954219cc42b013db6e48db65),
verifying that the fix resolved the issue.

The fixes from upstream LibTomCrypt has been cherry-picked into OP-TEE. The fix
for TEE core can be found upstream in
[this](https://github.com/OP-TEE/optee_os/commit/30d13250c390c4f56adefdcd3b64b7cc672f9fe2)
patch and a test case has been added to the test suite for OP-TEE and that
can also be found upstream in
[this](https://github.com/OP-TEE/optee_test/commit/b58916e35fe1f73cb7d32eb5ac04ab66f59669)
patch.

| Reported by  | CVE ID | OP-TEE ID | Affected versions |
| ------------ |:------:| :-------: | ----------------- |
| [Intel Security Advanced Threat Research] | [CVE-2016-6129](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2016-6129) | OP-TEE-2016-0001 | All versions prior to OP-TEE v2.2.0 (fixed in OP-TEE v2.2.0) |

[Applus+ Laboratories]: http://www.appluslaboratories.com
[Cyberus Technology]: https://www.cyberus-technology.de
[Contact]: https://optee.readthedocs.io/en/latest/general/contact.html#vulnerability-reporting
[Data61]: https://www.data61.csiro.au
[Google Project Zero]: https://googleprojectzero.blogspot.com
[Graz University of Technology]: https://www.iaik.tugraz.at
[Intel Security Advanced Threat Research]: http://www.intelsecurity.com/advanced-threat-research
[KPTI]: https://lwn.net/Articles/741878
[LibTomCrypt]: https://www.libtom.net/LibTomCrypt/
[Meltdown and Spectre]: https://spectreattack.com
[Meltdown whitepaper]: https://meltdownattack.com/meltdown.pdf
[Netflix]: https://www.netflix.com
[optee_os]: https://github.com/OP-TEE/optee_os
[optee_test]: https://github.com/OP-TEE/optee_test
[OP-TEE]: https://github.com/OP-TEE
[Rambus]: https://www.rambus.com
[Riscure]: https://www.riscure.com
[University of Adelaide]: https://www.adelaide.edu.au
[University of Maryland]: https://www.umd.edu
[University of Pennsylvania]: https://www.upenn.edu
