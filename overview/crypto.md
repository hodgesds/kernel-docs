# Cryptographic Subsystem

## Overview

The Linux kernel cryptographic subsystem (Crypto API) provides a generic, type-safe,
extensible framework for cryptographic algorithms. All algorithms register into a global
list, are discovered by name and type at runtime, and are accessed through typed transform
handles (`tfm`). The subsystem supports synchronous and asynchronous operation, template
composition (e.g., `cbc(aes)`), hardware offload via the crypto engine framework, and
userspace access via `AF_ALG` sockets.

---

## 1. Architecture

The Crypto API is organized around three concepts:

- **Algorithm (`crypto_alg`)** — describes an implementation (name, priority, block size, ops)
- **Transform (`crypto_tfm`)** — a runtime instance of an algorithm (holds key, state)
- **Request** — a per-operation object (holds I/O buffers, completion callback)

Algorithm types are identified by constants in `include/linux/crypto.h`:

```c
CRYPTO_ALG_TYPE_CIPHER      0x01   /* raw single-block cipher */
CRYPTO_ALG_TYPE_COMPRESS    0x02   /* sync compression (legacy) */
CRYPTO_ALG_TYPE_AEAD        0x03   /* authenticated encryption with associated data */
CRYPTO_ALG_TYPE_LSKCIPHER   0x04   /* linear skcipher (flat buffers, no scatterlists) */
CRYPTO_ALG_TYPE_SKCIPHER    0x05   /* symmetric key cipher (scatter-gather) */
CRYPTO_ALG_TYPE_AKCIPHER    0x06   /* asymmetric key cipher (RSA) */
CRYPTO_ALG_TYPE_SIG         0x07   /* digital signature */
CRYPTO_ALG_TYPE_KPP         0x08   /* key-agreement protocol primitives (DH, ECDH) */
CRYPTO_ALG_TYPE_ACOMPRESS   0x0a   /* async compression */
CRYPTO_ALG_TYPE_SCOMPRESS   0x0b   /* sync compression (modern) */
CRYPTO_ALG_TYPE_RNG         0x0c   /* random number generator */
CRYPTO_ALG_TYPE_SHASH       0x0e   /* synchronous hash */
CRYPTO_ALG_TYPE_AHASH       0x0f   /* asynchronous hash */
```

### Source Files

```
crypto/api.c                  - Core: tfm alloc/free, algorithm lookup, larval mechanism
crypto/algapi.c               - Registration: crypto_register_alg/template/instance, spawns
crypto/algboss.c              - Template manager: parses "cbc(aes)", spawns creation threads
crypto/proc.c                 - /proc/crypto interface
crypto/skcipher.c             - skcipher type dispatch
crypto/aead.c                 - AEAD type dispatch
crypto/ahash.c                - ahash type dispatch
crypto/shash.c                - shash type dispatch
crypto/akcipher.c             - akcipher type
crypto/kpp.c                  - KPP type
crypto/rng.c                  - RNG type
crypto/acompress.c            - async compression type
crypto/cipher.c               - raw block cipher type
crypto/scatterwalk.c          - scatter_walk helpers
crypto/drbg.c                 - SP800-90A DRBG
crypto/testmgr.c              - Self-test runner
crypto/testmgr.h              - Test vector data
crypto/crypto_engine.c        - Hardware offload engine
crypto/af_alg.c               - AF_ALG socket base layer
crypto/algif_skcipher.c       - AF_ALG skcipher socket
crypto/algif_hash.c           - AF_ALG hash socket
crypto/algif_aead.c           - AF_ALG AEAD socket
crypto/algif_rng.c            - AF_ALG RNG socket
crypto/cbc.c                  - CBC mode template
crypto/gcm.c                  - GCM mode template
crypto/hmac.c                 - HMAC template
crypto/ccm.c                  - CCM mode template
crypto/authenc.c              - authenc template
crypto/cryptd.c               - sync-to-async daemon

include/linux/crypto.h        - crypto_alg, crypto_tfm, crypto_wait, type/flag constants
include/crypto/algapi.h       - crypto_type, crypto_instance, crypto_template, scatter_walk
include/crypto/skcipher.h     - skcipher structs and API
include/crypto/aead.h         - AEAD structs and API
include/crypto/hash.h         - hash structs and API (shash + ahash)
include/crypto/akcipher.h     - akcipher structs and API
include/crypto/kpp.h          - KPP structs and API
include/crypto/rng.h          - RNG structs and API
include/crypto/acompress.h    - async compression structs and API
include/crypto/scatterwalk.h  - scatter_walk operations
include/crypto/drbg.h         - DRBG state
include/crypto/engine.h       - crypto engine public API
include/crypto/internal/engine.h - struct crypto_engine
include/crypto/if_alg.h       - AF_ALG socket structs
include/crypto/internal/skcipher.h - skcipher_walk, skcipher_instance
```

---

## 2. Core Data Structures

### struct crypto_alg (include/linux/crypto.h)

Base algorithm descriptor. Every algorithm type embeds this.

```c
struct crypto_alg {
    struct list_head cra_list;       /* link in global crypto_alg_list */
    struct list_head cra_users;      /* spawns (template instances) using this alg */

    u32 cra_flags;                   /* CRYPTO_ALG_TYPE_* | status flags */
    unsigned int cra_blocksize;      /* block size in bytes */
    unsigned int cra_ctxsize;        /* bytes for transform context */
    unsigned int cra_alignmask;      /* (alignment - 1) for I/O buffers */

    int cra_priority;                /* higher wins when multiple impls match */
    refcount_t cra_refcnt;

    char cra_name[CRYPTO_MAX_ALG_NAME];        /* generic name: "aes", "sha256" */
    char cra_driver_name[CRYPTO_MAX_ALG_NAME]; /* unique driver name: "aes-aesni" */

    const struct crypto_type *cra_type;

    union {
        struct cipher_alg cipher;    /* for TYPE_CIPHER */
        struct compress_alg compress;/* for TYPE_COMPRESS */
    } cra_u;

    int (*cra_init)(struct crypto_tfm *tfm);
    void (*cra_exit)(struct crypto_tfm *tfm);
    void (*cra_destroy)(struct crypto_alg *alg);

    struct module *cra_module;
};
```

Algorithm status flags (upper bits of `cra_flags`):
- `CRYPTO_ALG_LARVAL` — placeholder during module load
- `CRYPTO_ALG_DEAD` — being unregistered
- `CRYPTO_ALG_ASYNC` — async algorithm (completion callback model)
- `CRYPTO_ALG_NEED_FALLBACK` — needs software fallback
- `CRYPTO_ALG_TESTED` — passed self-tests
- `CRYPTO_ALG_INSTANCE` — template-derived instance
- `CRYPTO_ALG_KERN_DRIVER_ONLY` — hardware, no userspace access
- `CRYPTO_ALG_INTERNAL` — only usable by other algorithms
- `CRYPTO_ALG_OPTIONAL_KEY` — has setkey but default key exists
- `CRYPTO_ALG_ALLOCATES_MEMORY` — may allocate during request processing
- `CRYPTO_ALG_FIPS_INTERNAL` — not FIPS-approved standalone

### struct crypto_type (include/crypto/algapi.h)

Type dispatch object. Each algorithm type (skcipher, aead, etc.) has one.

```c
struct crypto_type {
    unsigned int (*ctxsize)(struct crypto_alg *alg, u32 type, u32 mask);
    unsigned int (*extsize)(struct crypto_alg *alg);
    int (*init_tfm)(struct crypto_tfm *tfm);
    void (*show)(struct seq_file *m, struct crypto_alg *alg);
    int (*report)(struct sk_buff *skb, struct crypto_alg *alg);
    void (*free)(struct crypto_instance *inst);
    unsigned int type;       /* CRYPTO_ALG_TYPE_* */
    unsigned int maskclear;
    unsigned int maskset;
    unsigned int tfmsize;    /* sizeof(struct crypto_<type>) prepended before crypto_tfm */
};
```

Memory layout: `crypto_alloc_tfm_node()` allocates `tfmsize + sizeof(crypto_tfm) + extsize`
bytes. The typed handle (e.g., `struct crypto_skcipher`) sits before `crypto_tfm` in memory,
allowing `container_of` to recover it.

### struct crypto_tfm (include/linux/crypto.h)

Runtime transform instance.

```c
struct crypto_tfm {
    refcount_t refcnt;
    u32 crt_flags;                   /* CRYPTO_TFM_* flags */
    int node;                        /* NUMA node */
    void (*exit)(struct crypto_tfm *tfm);
    struct crypto_alg *__crt_alg;    /* underlying algorithm */
    void *__crt_ctx[];               /* algorithm private context */
};
```

Transform flags:
- `CRYPTO_TFM_NEED_KEY` — `setkey()` not yet called
- `CRYPTO_TFM_REQ_FORBID_WEAK_KEYS` — reject weak keys
- `CRYPTO_TFM_REQ_MAY_SLEEP` — can sleep during operation
- `CRYPTO_TFM_REQ_MAY_BACKLOG` — can use backlog queue

Algorithm context accessed via `crypto_tfm_ctx(tfm)` which returns `tfm->__crt_ctx`.

### struct crypto_wait (include/linux/crypto.h)

Async-to-sync bridge for callers that need synchronous semantics:

```c
struct crypto_wait {
    struct completion completion;
    int err;
};
```

Usage pattern:
1. `DECLARE_CRYPTO_WAIT(wait)` or `crypto_init_wait(&wait)`
2. Set `req->base.complete = crypto_req_done` and `req->base.data = &wait`
3. Call the async operation
4. `err = crypto_wait_req(err, &wait)` — waits if `-EINPROGRESS` or `-EBUSY`

---

## 3. Symmetric Key Ciphers (skcipher)

### struct skcipher_alg (include/crypto/skcipher.h)

```c
struct skcipher_alg {
    int (*setkey)(struct crypto_skcipher *tfm, const u8 *key, unsigned int keylen);
    int (*encrypt)(struct skcipher_request *req);
    int (*decrypt)(struct skcipher_request *req);
    int (*export)(struct skcipher_request *req, void *out);
    int (*import)(struct skcipher_request *req, const void *in);
    int (*init)(struct crypto_skcipher *tfm);
    void (*exit)(struct crypto_skcipher *tfm);
    unsigned int walksize;

    /* common fields (SKCIPHER_ALG_COMMON macro): */
    unsigned int min_keysize;
    unsigned int max_keysize;
    unsigned int ivsize;       /* IV length in bytes */
    unsigned int chunksize;    /* effective block size */
    unsigned int statesize;    /* internal state for export/import */
    struct crypto_alg base;
};
```

### struct crypto_skcipher (include/crypto/skcipher.h)

```c
struct crypto_skcipher {
    unsigned int reqsize;   /* per-request private context size */
    struct crypto_tfm base;
};
```

### struct skcipher_request (include/crypto/skcipher.h)

```c
struct skcipher_request {
    unsigned int cryptlen;           /* bytes to encrypt/decrypt */
    u8 *iv;                          /* initialization vector */
    struct scatterlist *src;         /* input data */
    struct scatterlist *dst;         /* output data (may == src for in-place) */
    struct crypto_async_request base;
    void *__ctx[];                   /* per-request state */
};
```

### Linear Skcipher (lskcipher_alg)

Newer variant operating on flat buffers rather than scatterlists:

```c
struct lskcipher_alg {
    int (*setkey)(struct crypto_lskcipher *tfm, const u8 *key, unsigned int keylen);
    int (*encrypt)(struct crypto_lskcipher *tfm, const u8 *src,
                   u8 *dst, unsigned len, u8 *siv, u32 flags);
    int (*decrypt)(struct crypto_lskcipher *tfm, const u8 *src,
                   u8 *dst, unsigned len, u8 *siv, u32 flags);
    struct skcipher_alg_common co;
};
```

The `siv` parameter combines IV + state. Flags include `CRYPTO_LSKCIPHER_FLAG_CONT`
(continuing) and `CRYPTO_LSKCIPHER_FLAG_FINAL`. Return value: bytes of leftover data
(positive = partial block remains), or negative on error.

### API

```c
struct crypto_skcipher *crypto_alloc_skcipher(const char *alg_name, u32 type, u32 mask);
void crypto_free_skcipher(struct crypto_skcipher *tfm);
int crypto_skcipher_setkey(struct crypto_skcipher *tfm, const u8 *key, unsigned int keylen);
int crypto_skcipher_encrypt(struct skcipher_request *req);
int crypto_skcipher_decrypt(struct skcipher_request *req);
```

---

## 4. AEAD — Authenticated Encryption with Associated Data

### struct aead_alg (include/crypto/aead.h)

```c
struct aead_alg {
    int (*setkey)(struct crypto_aead *tfm, const u8 *key, unsigned int keylen);
    int (*setauthsize)(struct crypto_aead *tfm, unsigned int authsize);
    int (*encrypt)(struct aead_request *req);
    int (*decrypt)(struct aead_request *req);
    int (*init)(struct crypto_aead *tfm);
    void (*exit)(struct crypto_aead *tfm);
    unsigned int ivsize;
    unsigned int maxauthsize;
    unsigned int chunksize;
    struct crypto_alg base;
};
```

### struct crypto_aead (include/crypto/aead.h)

```c
struct crypto_aead {
    unsigned int authsize;  /* current auth tag size */
    unsigned int reqsize;
    struct crypto_tfm base;
};
```

### struct aead_request (include/crypto/aead.h)

```c
struct aead_request {
    struct crypto_async_request base;
    unsigned int assoclen;   /* associated data length */
    unsigned int cryptlen;   /* plaintext/ciphertext length */
    u8 *iv;
    struct scatterlist *src; /* [assoclen AAD][cryptlen data] */
    struct scatterlist *dst; /* [assoclen AAD][cryptlen data][authsize tag] on encrypt */
    void *__ctx[];
};
```

Memory layout convention: src/dst scatterlists begin with `assoclen` bytes of AAD, followed
by plaintext/ciphertext. On encryption, the auth tag is appended after ciphertext (dst grows
by `authsize`). On decryption, the tag is consumed from src. Decryption returns `-EBADMSG`
on tag mismatch.

---

## 5. Hash Algorithms

### Synchronous Hash (shash)

#### struct shash_alg (include/crypto/hash.h)

```c
struct shash_alg {
    int (*init)(struct shash_desc *desc);
    int (*update)(struct shash_desc *desc, const u8 *data, unsigned int len);
    int (*final)(struct shash_desc *desc, u8 *out);
    int (*finup)(struct shash_desc *desc, const u8 *data, unsigned int len, u8 *out);
    int (*digest)(struct shash_desc *desc, const u8 *data, unsigned int len, u8 *out);
    int (*export)(struct shash_desc *desc, void *out);
    int (*import)(struct shash_desc *desc, const void *in);
    int (*setkey)(struct crypto_shash *tfm, const u8 *key, unsigned int keylen);
    int (*init_tfm)(struct crypto_shash *tfm);
    void (*exit_tfm)(struct crypto_shash *tfm);
    unsigned int descsize;   /* per-desc context size */
    unsigned int digestsize; /* output digest size */
    unsigned int statesize;  /* exported state size */
    struct crypto_alg base;
};
```

#### struct shash_desc (include/crypto/hash.h)

```c
struct shash_desc {
    struct crypto_shash *tfm;
    void *__ctx[];           /* algorithm internal state */
};
```

Stack allocation: `SHASH_DESC_ON_STACK(shash, ctx)` uses `HASH_MAX_DESCSIZE` (360 bytes).
`HASH_MAX_DIGESTSIZE` is 64 bytes (SHA-512).

### Asynchronous Hash (ahash)

#### struct ahash_alg (include/crypto/hash.h)

```c
struct ahash_alg {
    int (*init)(struct ahash_request *req);
    int (*update)(struct ahash_request *req);
    int (*final)(struct ahash_request *req);
    int (*finup)(struct ahash_request *req);
    int (*digest)(struct ahash_request *req);
    int (*export)(struct ahash_request *req, void *out);
    int (*import)(struct ahash_request *req, const void *in);
    int (*setkey)(struct crypto_ahash *tfm, const u8 *key, unsigned int keylen);
    struct hash_alg_common halg;
};
```

#### struct crypto_ahash (include/crypto/hash.h)

```c
struct crypto_ahash {
    bool using_shash;   /* true if backed by a shash (transparent wrapping) */
    unsigned int statesize;
    unsigned int reqsize;
    struct crypto_tfm base;
};
```

The `using_shash` flag: the ahash layer transparently wraps shash algorithms, so
`crypto_alloc_ahash("sha256", ...)` may return a shash-backed ahash.

#### struct ahash_request (include/crypto/hash.h)

```c
struct ahash_request {
    struct crypto_async_request base;
    unsigned int nbytes;
    struct scatterlist *src;
    u8 *result;              /* output buffer for digest */
    void *__ctx[];
};
```

---

## 6. Asymmetric Cipher and Key Agreement

### Asymmetric Key Cipher (akcipher)

```c
struct akcipher_alg {
    int (*encrypt)(struct akcipher_request *req);
    int (*decrypt)(struct akcipher_request *req);
    int (*set_pub_key)(struct crypto_akcipher *tfm, const void *key, unsigned int keylen);
    int (*set_priv_key)(struct crypto_akcipher *tfm, const void *key, unsigned int keylen);
    unsigned int (*max_size)(struct crypto_akcipher *tfm);
    int (*init)(struct crypto_akcipher *tfm);
    void (*exit)(struct crypto_akcipher *tfm);
    struct crypto_alg base;
};
```

Keys are BER-encoded. `max_size()` returns required output buffer size (call after setkey).

### Key-Agreement Protocol Primitives (KPP)

```c
struct kpp_alg {
    int (*set_secret)(struct crypto_kpp *tfm, const void *buffer, unsigned int len);
    int (*generate_public_key)(struct kpp_request *req);
    int (*compute_shared_secret)(struct kpp_request *req);
    unsigned int (*max_size)(struct crypto_kpp *tfm);
    int (*init)(struct crypto_kpp *tfm);
    void (*exit)(struct crypto_kpp *tfm);
    struct crypto_alg base;
};
```

`set_secret` takes a type-tagged buffer (`struct kpp_secret`). DH and ECDH define their
own packing formats in `include/crypto/dh.h` and `include/crypto/ecdh.h`.

---

## 7. RNG and Compression

### Random Number Generator (rng)

```c
struct rng_alg {
    int (*generate)(struct crypto_rng *tfm,
                    const u8 *src, unsigned int slen,
                    u8 *dst, unsigned int dlen);
    int (*seed)(struct crypto_rng *tfm, const u8 *seed, unsigned int slen);
    void (*set_ent)(struct crypto_rng *tfm, const u8 *data, unsigned int len);
    unsigned int seedsize;
    struct crypto_alg base;
};
```

`crypto_get_default_rng()` returns the global default. The "krng" algorithm bridges to
`get_random_bytes()`.

### Async Compression (acomp)

```c
struct crypto_acomp {
    int (*compress)(struct acomp_req *req);
    int (*decompress)(struct acomp_req *req);
    void (*dst_free)(struct scatterlist *dst);
    unsigned int reqsize;
    struct crypto_tfm base;
};

struct acomp_req {
    struct crypto_async_request base;
    struct scatterlist *src;
    struct scatterlist *dst;      /* NULL = algo allocates */
    unsigned int slen;
    unsigned int dlen;            /* in: max output; out: actual output */
    u32 flags;
    void *__ctx[];
};
```

---

## 8. Templates and Instances

Templates implement cipher mode composition. When a user requests `cbc(aes)`:

1. Try to find a registered algorithm named `"cbc(aes)"` exactly
2. If not found, fire `CRYPTO_MSG_ALG_REQUEST` notification to crypto manager (`algboss.c`)
3. Manager parses `"cbc(aes)"` into template `"cbc"` and parameter `"aes"`
4. Spawns kthread calling `tmpl->create(tmpl, tb)` with rtattr encoding
5. Template's `create` looks up `"aes"`, creates a `crypto_instance`, registers it
6. New instance appears in `crypto_alg_list` as `"cbc(aes)"`

### struct crypto_template (include/crypto/algapi.h)

```c
struct crypto_template {
    struct list_head list;
    struct hlist_head instances;
    struct module *module;
    int (*create)(struct crypto_template *tmpl, struct rtattr **tb);
    char name[CRYPTO_MAX_ALG_NAME];
};
```

### struct crypto_instance (include/crypto/algapi.h)

```c
struct crypto_instance {
    struct crypto_alg alg;         /* the instance IS a crypto_alg */
    struct crypto_template *tmpl;
    union {
        struct hlist_node list;    /* in tmpl->instances after registration */
        struct crypto_spawn *spawns;
    };
    struct work_struct free_work;
    void *__ctx[];
};
```

Type-specific wrappers (`skcipher_instance`, `aead_instance`, etc.) embed `crypto_instance`
in a union with the type's `*_alg` struct.

### struct crypto_spawn (include/crypto/algapi.h)

A spawn is the mechanism by which a template instance holds a reference to its inner
algorithm. If the inner algorithm is unregistered, `crypto_remove_spawns()` traverses
the `cra_users` list and schedules instance destruction.

```c
struct crypto_spawn {
    struct list_head list;         /* link in alg->cra_users */
    struct crypto_alg *alg;        /* inner algorithm */
    union {
        struct crypto_instance *inst;
        struct crypto_spawn *next;
    };
    const struct crypto_type *frontend;
    u32 mask;
    bool dead;
    bool registered;
};
```

### Common Templates

- **cbc** (`crypto/cbc.c`) — CBC mode, wraps an lskcipher
- **gcm** (`crypto/gcm.c`) — GCM mode, wraps CTR skcipher + GHASH
- **hmac** (`crypto/hmac.c`) — HMAC, wraps another shash
- **ccm** (`crypto/ccm.c`) — CCM mode AEAD
- **authenc** (`crypto/authenc.c`) — `authenc(hmac(sha256),cbc(aes))`

---

## 9. Scatterlist Walk

### struct scatter_walk (include/crypto/algapi.h)

Low-level iterator over scatterlist elements:

```c
struct scatter_walk {
    struct scatterlist *sg;
    unsigned int offset;
};
```

Key operations (`include/crypto/scatterwalk.h`):
- `scatterwalk_start(walk, sg)` — initialize at beginning
- `scatterwalk_map(walk)` — `kmap_local_page()` current page, return pointer
- `scatterwalk_unmap(vaddr)` — `kunmap_local()`
- `scatterwalk_copychunks(buf, walk, nbytes, out)` — copy between flat buffer and SGL
- `scatterwalk_map_and_copy(buf, sg, start, nbytes, out)` — one-shot map+copy
- `scatterwalk_ffwd(dst, src, len)` — fast-forward src by len bytes

### struct skcipher_walk (include/crypto/internal/skcipher.h)

High-level block cipher walk for algorithm implementations:

```c
struct skcipher_walk {
    union { ... } src, dst;        /* current chunk addresses */
    struct scatter_walk in;        /* input position */
    unsigned int nbytes;           /* bytes available in current chunk */
    struct scatter_walk out;       /* output position */
    unsigned int total;            /* remaining bytes */
    struct list_head buffers;      /* bounce buffers */
    u8 *page;                      /* bounce page */
    void *iv;                      /* current IV */
    unsigned int ivsize;
    unsigned int blocksize;
    unsigned int stride;           /* chunk size */
    unsigned int alignmask;
};
```

Usage pattern for algorithm implementations:

```c
struct skcipher_walk walk;
err = skcipher_walk_virt(&walk, req, false);
while (walk.nbytes > 0) {
    unsigned int processed = encrypt_chunk(walk.src.virt.addr,
                                           walk.dst.virt.addr,
                                           walk.nbytes & ~(blocksize-1));
    err = skcipher_walk_done(&walk, walk.nbytes - processed);
}
```

---

## 10. Crypto Engine — Hardware Offload

The crypto engine provides a FIFO work queue and kthread worker for hardware accelerators.

### struct crypto_engine (include/crypto/internal/engine.h)

```c
struct crypto_engine {
    char name[ENGINE_NAME_LEN];
    bool running;
    bool busy;
    bool retry_support;

    spinlock_t queue_lock;              /* protects queue and state */
    struct crypto_queue queue;          /* FIFO of pending requests */
    struct device *dev;
    bool rt;                            /* run kthread as SCHED_FIFO */

    int (*prepare_crypt_hardware)(struct crypto_engine *engine);
    int (*unprepare_crypt_hardware)(struct crypto_engine *engine);
    int (*do_batch_requests)(struct crypto_engine *engine);

    struct kthread_worker *kworker;
    struct kthread_work pump_requests;
    struct crypto_async_request *cur_req;
};
```

### Engine Algorithm Types

Each algorithm type has an engine variant adding `do_one_request`:

```c
struct skcipher_engine_alg {
    struct skcipher_alg base;
    struct crypto_engine_op op;   /* int (*do_one_request)(engine, areq); */
};
/* similarly: aead_engine_alg, ahash_engine_alg, akcipher_engine_alg, kpp_engine_alg */
```

### Engine Lifecycle

1. **Allocate**: `crypto_engine_alloc_init(dev, rt)` — creates engine, inits queue, starts
   kthread worker
2. **Register algorithms**: `crypto_engine_register_skcipher(alg)` — sets
   `CRYPTO_ALG_ENGINE` in `cra_flags`
3. **Submit requests**: driver's `encrypt`/`decrypt` calls
   `crypto_transfer_skcipher_request_to_engine(engine, req)`
4. **Process**: kthread pump dequeues request, calls `op->do_one_request(engine, req)`
5. **Complete**: hardware ISR calls `crypto_finalize_skcipher_request(engine, req, err)` —
   invokes completion callback, re-queues pump
6. **Stop**: `crypto_engine_stop(engine)` — drains queue
7. **Exit**: `crypto_engine_exit(engine)` — destroys kthread

The `queue_lock` uses `spin_lock_irqsave` because completion runs in interrupt context.

---

## 11. AF_ALG — Userspace Interface

`AF_ALG` (address family 38) exposes kernel crypto via sockets.

### Userspace Workflow

```c
int tfmfd = socket(AF_ALG, SOCK_SEQPACKET, 0);
struct sockaddr_alg sa = { .salg_family = AF_ALG,
                            .salg_type = "skcipher",
                            .salg_name = "cbc(aes)" };
bind(tfmfd, (struct sockaddr*)&sa, sizeof(sa));
setsockopt(tfmfd, SOL_ALG, ALG_SET_KEY, key, keylen);
int opfd = accept(tfmfd, 0, 0);
sendmsg(opfd, &msg, 0);   /* msg contains cmsg with IV and op direction */
read(opfd, buf, len);      /* read result */
```

### struct alg_sock (include/crypto/if_alg.h)

```c
struct alg_sock {
    struct sock sk;
    struct sock *parent;
    atomic_t refcnt;
    const struct af_alg_type *type;
    void *private;               /* type-specific state (e.g., crypto_skcipher handle) */
};
```

### struct af_alg_type (include/crypto/if_alg.h)

```c
struct af_alg_type {
    void *(*bind)(const char *name, u32 type, u32 mask);
    void (*release)(void *private);
    int (*setkey)(void *private, const u8 *key, unsigned int keylen);
    int (*setentropy)(void *private, sockptr_t entropy, unsigned int len);
    int (*accept)(void *private, struct sock *sk);
    int (*setauthsize)(void *private, unsigned int authsize);
    struct proto_ops *ops;
    struct proto_ops *ops_nokey;
    char name[14];             /* "skcipher", "hash", "aead", "rng" */
};
```

### struct af_alg_ctx (include/crypto/if_alg.h)

```c
struct af_alg_ctx {
    struct list_head tsgl_list;  /* TX scatter-gather entries */
    void *iv;
    void *state;
    size_t aead_assoclen;
    struct crypto_wait wait;
    size_t used;                 /* TX bytes pending */
    atomic_t rcvused;
    bool more;
    bool enc;                    /* encrypt or decrypt */
    unsigned int inflight;       /* in-flight AIO requests */
};
```

### Modules

- **algif_skcipher** — TX data collected, `recvmsg` with `ALG_OP_ENCRYPT`/`ALG_OP_DECRYPT`
  triggers operation. IV passed via cmsg `ALG_SET_IV`.
- **algif_hash** — data fed via `sendmsg` to `crypto_ahash_update()`. Zero-length send or
  `recvmsg` triggers `crypto_ahash_final()`.
- **algif_aead** — like skcipher but respects `assoclen` cmsg for AAD.
- **algif_rng** — `recvmsg` calls `crypto_rng_generate()`.

---

## 12. DRBG — Deterministic Random Bit Generator

`crypto/drbg.c` implements NIST SP800-90A with three constructions:
- **HMAC-DRBG** — uses HMAC-SHA-256/384/512
- **Hash-DRBG** — uses SHA-256/384/512 directly
- **CTR-DRBG** — uses AES-128/192/256 in counter mode

### struct drbg_state (include/crypto/drbg.h)

```c
struct drbg_state {
    struct mutex drbg_mutex;        /* serializes all operations */
    unsigned char *V;               /* internal state */
    unsigned char *C;               /* hash: constant C; hmac/ctr: key K */
    size_t reseed_ctr;              /* requests since last reseed */
    size_t reseed_threshold;        /* reseed limit (kernel uses 1<<20) */
    unsigned char *scratchpad;
    void *priv_data;                /* underlying cipher handle */
    enum drbg_seed_state seeded;    /* UNSEEDED / PARTIAL / FULL */
    unsigned long last_seed_time;
    bool pr;                        /* prediction resistance */
    bool fips_primed;               /* FIPS continuous test primed */
    unsigned char *prev;            /* previous output (continuous test) */
    struct crypto_rng *jent;        /* jitter entropy source */
    const struct drbg_state_ops *d_ops;
    const struct drbg_core *core;
};
```

### Seeding

The DRBG is seeded from `get_random_bytes()`. Seed states:
- `DRBG_SEED_STATE_UNSEEDED` — no seed
- `DRBG_SEED_STATE_PARTIAL` — seeded before kernel RNG fully initialized; will reseed
- `DRBG_SEED_STATE_FULL` — seeded from fully initialized RNG

The DRBG does NOT feed `/dev/random`. The kernel's primary RNG (`drivers/char/random.c`)
is separate. The DRBG is a crypto API provider that uses `get_random_bytes()` as entropy.

Reseed threshold: SP800-90A mandates max 2^48 requests; kernel uses 1<<20 (~1M).

---

## 13. Hardware Acceleration

### Priority-Based Selection

`__crypto_alg_lookup()` in `crypto/api.c` walks `crypto_alg_list` and selects the highest
`cra_priority` among matching entries. An exact `cra_driver_name` match takes priority
regardless of priority value.

Typical priority values:
- Generic software: 100
- Platform-specific software (SIMD/AES-NI): 300
- Hardware accelerators: 400+

### Fallback Mechanism

When `CRYPTO_ALG_NEED_FALLBACK` is set, the driver allocates a software fallback in `init`:

```c
ctx->fallback = crypto_alloc_skcipher("cbc(aes)", 0,
                                      CRYPTO_ALG_ASYNC | CRYPTO_ALG_NEED_FALLBACK);
```

For operations hardware cannot handle (unsupported key sizes, etc.), `do_one_request`
uses the fallback.

### Async vs Sync

Async algorithms set `CRYPTO_ALG_ASYNC` in `cra_flags`. Callers must provide a completion
callback in the request. `crypto_wait` bridges async to sync when needed. The `cryptd`
daemon (`crypto/cryptd.c`) converts sync algorithms to async by running them on a workqueue.

---

## 14. Self-Tests

### How Tests Run

When `crypto_register_alg()` is called (`crypto/algapi.c`):
1. `crypto_alloc_test_larval()` creates a larval placeholder
2. `CRYPTO_MSG_ALG_REGISTER` notification fires
3. Crypto manager spawns kthread calling `alg_test()` in `testmgr.c`
4. On success: larval resolved, `CRYPTO_ALG_TESTED` set
5. On failure: `CRYPTO_ALG_DEAD` set, callers get `-ENOENT`

### alg_test() (crypto/testmgr.c)

1. Binary search in sorted `alg_test_descs[]` array by algorithm name
2. In FIPS mode: check `fips_allowed` flag
3. Call type-specific test: `alg_test_skcipher()`, `alg_test_aead()`, `alg_test_hash()`, etc.
4. Tests use vectors from `testmgr.h`

### FIPS Mode

When `fips_enabled`:
- `CRYPTO_ALG_FIPS_INTERNAL` algorithms cannot be used directly
- Module signatures verified — `panic()` if invalid
- HMAC minimum key length enforced (14 bytes)
- Continuous output test for DRBGs

---

## 15. Key Kernel Users

### dm-crypt (drivers/md/dm-crypt.c)

```c
crypto_alloc_skcipher(ciphermode, 0, CRYPTO_ALG_ASYNC);    /* bulk encryption */
crypto_alloc_aead(ciphermode, 0, CRYPTO_ALG_ASYNC);        /* authenticated mode */
```

### IPsec / xfrm (net/xfrm/)

Queries `crypto_has_aead()`, `crypto_has_ahash()`, `crypto_has_skcipher()`. ESP uses
AEAD transforms in `net/ipv4/esp4.c` and `net/ipv6/esp6.c`.

### kTLS (net/tls/)

```c
crypto_alloc_aead(cipher_desc->cipher_name, 0, 0);  /* typically gcm(aes) */
```

### fscrypt (fs/crypto/)

```c
crypto_alloc_skcipher(mode->cipher_str, 0, 0);       /* AES-256-XTS, AES-128-CBC, etc. */
crypto_alloc_shash(HKDF_HMAC_ALG, 0, 0);             /* hmac(sha512) for HKDF */
```

### IMA (security/integrity/ima/)

```c
crypto_alloc_shash(hash_algo_name[ima_hash_algo], 0, 0);  /* file measurement */
crypto_alloc_ahash(hash_algo_name[algo], 0, 0);            /* large file async */
```

---

## 16. Locking Summary

| Lock | Type | Protects |
|---|---|---|
| `crypto_alg_sem` | rw_semaphore | `crypto_alg_list`, `crypto_template_list`, `cra_users`, larvals |
| `alg_types_sem` | rw_semaphore | AF_ALG `alg_types` list |
| `engine->queue_lock` | spinlock (irqsave) | engine queue, running/busy state |
| `drbg->drbg_mutex` | mutex | DRBG state during generate/reseed |
| `lock_sock(sk)` | socket lock | per-socket AF_ALG ctx during sendmsg/recvmsg |

`crypto_alg_sem` is read-locked for lookups (frequent) and write-locked for registration/
unregistration (rare). No per-request locking in the API — callers must not submit concurrent
operations on the same request object. Multiple in-flight requests on the same tfm handle
are safe (tfm is read-only after `setkey`; state lives in request `__ctx`).
