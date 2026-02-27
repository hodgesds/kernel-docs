# Linux Kernel Netfilter Subsystem Overview

## Table of Contents
1. [Introduction](#introduction)
2. [Netfilter Hook Architecture](#netfilter-hook-architecture)
3. [Hook Registration and Execution](#hook-registration-and-execution)
4. [nf_tables Architecture](#nf_tables-architecture)
5. [Tables, Chains, and Rules](#tables-chains-and-rules)
6. [Expressions and the Evaluation Loop](#expressions-and-the-evaluation-loop)
7. [Sets and Maps](#sets-and-maps)
8. [Connection Tracking (conntrack)](#connection-tracking-conntrack)
9. [Network Address Translation (NAT)](#network-address-translation-nat)
10. [Flowtables and Hardware Offload](#flowtables-and-hardware-offload)
11. [iptables vs nftables](#iptables-vs-nftables)
12. [Locking and Concurrency](#locking-and-concurrency)
13. [Source Files Reference](#source-files-reference)

---

## Introduction

Netfilter is the Linux kernel's framework for packet filtering, network
address translation (NAT), port translation, and connection tracking. It
provides a set of hooks at well-defined points in the network stack where
kernel modules can register callback functions to inspect and manipulate
packets. The framework enables stateful firewalling, NAT, packet mangling,
and packet logging.

### Key Components

```
+-------------------------------------------------------------+
|                    Netfilter Subsystem                       |
+-------------------------------------------------------------+
|                                                             |
|  1. Hook Framework       (net/netfilter/core.c)             |
|     - 5 IPv4/IPv6 hooks + ingress/egress hooks              |
|     - Priority-ordered callback chains                      |
|     - RCU-protected fast path                               |
|                                                             |
|  2. nf_tables            (net/netfilter/nf_tables_api.c)    |
|     - Modern packet classification engine                   |
|     - Tables, chains, rules, expressions                    |
|     - Sets, maps, stateful objects                          |
|     - Atomic ruleset replacement via transactions            |
|                                                             |
|  3. Connection Tracking  (net/netfilter/nf_conntrack_core.c)|
|     - Stateful packet inspection                            |
|     - Protocol-specific trackers (TCP, UDP, ICMP, etc.)     |
|     - Expectation framework for ALGs                        |
|                                                             |
|  4. NAT                  (net/netfilter/nf_nat_core.c)      |
|     - Source/destination NAT                                |
|     - Masquerading                                          |
|     - Built on connection tracking                          |
|                                                             |
|  5. Flowtables           (net/netfilter/nf_flow_table_core.c)|
|     - Software fast path for established flows               |
|     - Hardware offload to SmartNICs                          |
|                                                             |
+-------------------------------------------------------------+
```

---

## Netfilter Hook Architecture

Netfilter defines hooks at five points in the IPv4/IPv6 packet processing
path, plus ingress and egress hooks at the device level. These hooks allow
registered callback functions to inspect, modify, accept, drop, or queue
packets.

### Hook Points (include/uapi/linux/netfilter.h:42)

```c
enum nf_inet_hooks {
    NF_INET_PRE_ROUTING,      /* After sanity checks, before routing */
    NF_INET_LOCAL_IN,          /* After routing, destined for local host */
    NF_INET_FORWARD,           /* After routing, destined for another host */
    NF_INET_LOCAL_OUT,         /* Locally generated, before routing */
    NF_INET_POST_ROUTING,      /* After routing, just before outgoing */
    NF_INET_NUMHOOKS,
    NF_INET_INGRESS = NF_INET_NUMHOOKS, /* Ingress hook (device level) */
};

enum nf_dev_hooks {
    NF_NETDEV_INGRESS,         /* Device-level ingress */
    NF_NETDEV_EGRESS,          /* Device-level egress */
    NF_NETDEV_NUMHOOKS
};
```

### Protocol Families (include/uapi/linux/netfilter.h:58)

```c
enum {
    NFPROTO_UNSPEC =  0,
    NFPROTO_INET   =  1,      /* IPv4 + IPv6 combined */
    NFPROTO_IPV4   =  2,
    NFPROTO_ARP    =  3,
    NFPROTO_NETDEV =  5,      /* Device-level hooks */
    NFPROTO_BRIDGE =  7,
    NFPROTO_IPV6   = 10,
    NFPROTO_NUMPROTO,
};
```

### Verdict Values (include/uapi/linux/netfilter.h:10)

```c
#define NF_DROP   0   /* Drop packet silently */
#define NF_ACCEPT 1   /* Continue normal processing */
#define NF_STOLEN 2   /* Packet taken over by hook, don't free */
#define NF_QUEUE  3   /* Queue packet to userspace (nfqueue) */
#define NF_REPEAT 4   /* Call this hook function again */
#define NF_STOP   5   /* Deprecated, for userspace compatibility */
```

### Packet Flow Through Hooks

```
  Incoming packet
       |
       v
  +-------------------------+
  | NF_INET_PRE_ROUTING     |  <-- conntrack, DNAT, raw
  +------------+------------+
               |
               v
       +---------------+
       | Routing        |
       | Decision       |
       +-------+-------+
               |
       +-------+-------+
       |               |
       v               v
    (local)         (forward)
       |               |
       v               v
  +---------------+ +---------------+
  |NF_INET_LOCAL_ | | NF_INET_      |
  |    INPUT      | |   FORWARD     |  <-- filter
  +-------+-------+ +-------+-------+
          |                  |
          v                  |
     Local process           |
          |                  |
          v                  |
  +---------------+          |
  |NF_INET_LOCAL_ |          |
  |    OUTPUT     |          |  <-- filter, DNAT, raw
  +-------+-------+          |
          |                  |
          +--------+---------+
                   |
                   v
           +---------------+
           | Routing        |
           | Decision       |
           +-------+-------+
                   |
                   v
          +------------------+
          |NF_INET_POST_     |
          |     ROUTING      |  <-- SNAT, conntrack confirm
          +--------+---------+
                   |
                   v
           Outgoing packet
```

### Hook Priority Order (include/uapi/linux/netfilter_ipv4.h:30)

Hooks at each point execute in ascending priority order:

```c
enum nf_ip_hook_priorities {
    NF_IP_PRI_FIRST              = INT_MIN,
    NF_IP_PRI_RAW_BEFORE_DEFRAG  = -450,
    NF_IP_PRI_CONNTRACK_DEFRAG   = -400,   /* Defragmentation */
    NF_IP_PRI_RAW                = -300,    /* raw table */
    NF_IP_PRI_SELINUX_FIRST      = -225,
    NF_IP_PRI_CONNTRACK          = -200,    /* Connection tracking IN */
    NF_IP_PRI_MANGLE             = -150,    /* mangle table */
    NF_IP_PRI_NAT_DST            = -100,    /* DNAT */
    NF_IP_PRI_FILTER             =    0,    /* filter table */
    NF_IP_PRI_SECURITY           =   50,    /* security table */
    NF_IP_PRI_NAT_SRC            =  100,    /* SNAT */
    NF_IP_PRI_SELINUX_LAST       =  225,
    NF_IP_PRI_CONNTRACK_HELPER   =  300,    /* CT helpers */
    NF_IP_PRI_CONNTRACK_CONFIRM  = INT_MAX, /* CT confirmation */
    NF_IP_PRI_LAST               = INT_MAX,
};
```

This ordering determines how the subsystems interact at each hook:

```
  NF_INET_PRE_ROUTING priority order:
  +---------+--------+-----+--------+--------+--------+
  | defrag  |  raw   | CT  | mangle |  DNAT  | filter |
  | (-400)  | (-300) |(-200)| (-150) | (-100) |  (0)   |
  +---------+--------+-----+--------+--------+--------+
  lowest priority --------------------------> highest priority
```

---

## Hook Registration and Execution

### nf_hook_ops (include/linux/netfilter.h:97)

The `nf_hook_ops` structure defines a single hook registration. Callers
fill in the hook function, protocol family, hook number, and priority.

```c
struct nf_hook_ops {
    nf_hookfn               *hook;          /* Callback function */
    struct net_device        *dev;           /* Device (for ingress/egress) */
    void                     *priv;         /* Private data passed to hook */
    u8                       pf;            /* Protocol family (NFPROTO_*) */
    enum nf_hook_ops_type    hook_ops_type:8; /* NF_HOOK_OP_NF_TABLES, _BPF */
    unsigned int             hooknum;       /* Hook point (NF_INET_*) */
    int                      priority;      /* Execution order */
};
```

### Hook Function Signature (include/linux/netfilter.h:88)

```c
typedef unsigned int nf_hookfn(void *priv,
                               struct sk_buff *skb,
                               const struct nf_hook_state *state);
```

### nf_hook_state (include/linux/netfilter.h:78)

The `nf_hook_state` captures the context at the point a hook is invoked --
the hook number, protocol family, input/output devices, associated socket,
network namespace, and the "ok function" to call if the hook accepts.

```c
struct nf_hook_state {
    u8                   hook;      /* Hook number */
    u8                   pf;        /* Protocol family */
    struct net_device    *in;       /* Input device */
    struct net_device    *out;      /* Output device */
    struct sock          *sk;       /* Associated socket */
    struct net           *net;      /* Network namespace */
    int (*okfn)(struct net *, struct sock *, struct sk_buff *);
};
```

### nf_hook_entries (include/linux/netfilter.h:119)

Hooks are stored in a flat array for cache-efficient traversal. The
`nf_hook_entries` structure holds the array of `nf_hook_entry` items (each
containing a function pointer and private data), followed by back-pointers
to the original `nf_hook_ops` for slow-path management.

```c
struct nf_hook_entry {
    nf_hookfn       *hook;
    void            *priv;
};

struct nf_hook_entries {
    u16                      num_hook_entries;
    struct nf_hook_entry     hooks[];
    /* Trailer (not in struct): nf_hook_ops *orig_ops[] + rcu_head */
};
```

### Registration API (net/netfilter/core.c:557)

```c
/* Register a single hook */
int nf_register_net_hook(struct net *net, const struct nf_hook_ops *ops);
void nf_unregister_net_hook(struct net *net, const struct nf_hook_ops *ops);

/* Register multiple hooks atomically */
int nf_register_net_hooks(struct net *net, const struct nf_hook_ops *reg,
                          unsigned int n);
void nf_unregister_net_hooks(struct net *net, const struct nf_hook_ops *reg,
                             unsigned int n);
```

When registering with `pf == NFPROTO_INET` and a hook other than
`NF_INET_INGRESS`, the kernel registers the hook for both `NFPROTO_IPV4`
and `NFPROTO_IPV6` automatically (core.c:561).

### The NF_HOOK() Macro (include/linux/netfilter.h:307)

The protocol stack invokes netfilter via the `NF_HOOK()` inline function.
If no hooks are registered (detected via static keys for zero overhead),
it calls `okfn` directly. Otherwise, it calls `nf_hook_slow()` to iterate
through all registered hooks.

```c
static inline int
NF_HOOK(uint8_t pf, unsigned int hook, struct net *net, struct sock *sk,
        struct sk_buff *skb, struct net_device *in, struct net_device *out,
        int (*okfn)(struct net *, struct sock *, struct sk_buff *))
{
    int ret = nf_hook(pf, hook, net, sk, skb, in, out, okfn);
    if (ret == 1)
        ret = okfn(net, sk, skb);
    return ret;
}
```

### nf_hook_slow() -- The Hook Iterator (net/netfilter/core.c:619)

This function iterates through the `nf_hook_entries` array, calling each
hook function and acting on its verdict:

```c
int nf_hook_slow(struct sk_buff *skb, struct nf_hook_state *state,
                 const struct nf_hook_entries *e, unsigned int s)
{
    unsigned int verdict;
    int ret;

    for (; s < e->num_hook_entries; s++) {
        verdict = nf_hook_entry_hookfn(&e->hooks[s], skb, state);
        switch (verdict & NF_VERDICT_MASK) {
        case NF_ACCEPT:
            break;                  /* Continue to next hook */
        case NF_DROP:
            kfree_skb_reason(skb, SKB_DROP_REASON_NETFILTER_DROP);
            ret = NF_DROP_GETERR(verdict);
            if (ret == 0) ret = -EPERM;
            return ret;
        case NF_QUEUE:
            ret = nf_queue(skb, state, s, verdict);
            if (ret == 1) continue; /* Re-injected, keep going */
            return ret;
        case NF_STOLEN:
            return NF_DROP_GETERR(verdict);
        }
    }
    return 1;                       /* All hooks accepted */
}
```

### Per-Namespace Hook Storage (net/netfilter/core.c:278)

Each network namespace stores its hooks in `net->nf.hooks_ipv4[]`,
`net->nf.hooks_ipv6[]`, `net->nf.hooks_arp[]`, and
`net->nf.hooks_bridge[]`. Ingress hooks are stored per-device in
`dev->nf_hooks_ingress`, and egress hooks in `dev->nf_hooks_egress`.

```c
static struct nf_hook_entries __rcu **
nf_hook_entry_head(struct net *net, int pf, unsigned int hooknum,
                   struct net_device *dev)
{
    switch (pf) {
    case NFPROTO_IPV4:
        return net->nf.hooks_ipv4 + hooknum;
    case NFPROTO_IPV6:
        return net->nf.hooks_ipv6 + hooknum;
    case NFPROTO_NETDEV:
        /* Uses dev->nf_hooks_ingress or dev->nf_hooks_egress */
        ...
    }
}
```

---

## nf_tables Architecture

nf_tables (nft) is the modern packet classification engine that replaces
iptables, ip6tables, arptables, and ebtables with a single unified
framework. It uses a virtual machine approach: rules consist of sequences
of expressions that operate on a register set.

### Architecture Overview

```
  User space: nft tool
       |
       | Netlink (nfnetlink)
       v
  +--------------------------------------------------+
  | nf_tables_api.c  (control plane)                 |
  |                                                  |
  |  +--------+    +--------+    +--------+          |
  |  | Table  | -> | Chain  | -> | Rule   |          |
  |  +--------+    +--------+    +--------+          |
  |       |             |            |               |
  |       |         +--------+   +--------+          |
  |       |         | Base   |   | Expr 1 |          |
  |       |         | Chain  |   | Expr 2 |          |
  |       |         | (hook) |   | Expr N |          |
  |       |         +--------+   +--------+          |
  |       |                                          |
  |  +--------+    +--------+    +--------+          |
  |  | Sets   |    | Maps   |    | Objects|          |
  |  +--------+    +--------+    +--------+          |
  +--------------------------------------------------+
               |
               | netfilter hooks
               v
  +--------------------------------------------------+
  | nf_tables_core.c  (data plane)                   |
  |                                                  |
  |  nft_do_chain():                                 |
  |    for each rule in chain:                       |
  |      for each expression in rule:                |
  |        expr->ops->eval(expr, regs, pkt)          |
  |      check verdict register                      |
  +--------------------------------------------------+
```

### Transaction Model

nf_tables uses a two-phase commit protocol for atomic ruleset updates.
Changes are staged as transactions and only become active when the entire
batch is committed. This avoids the "rule-by-rule" updates of iptables
that could leave the ruleset in an inconsistent state.

The generation cursor (`net->nft.gencursor`) tracks two generations. Each
object has a 2-bit genmask: a set bit means the object is inactive in
that generation. During commit, the generation cursor flips and a
synchronize_rcu ensures no packet-path readers see stale state
(include/net/netfilter/nf_tables.h:1528).

---

## Tables, Chains, and Rules

### struct nft_table (include/net/netfilter/nf_tables.h:1287)

A table is a container for chains, sets, stateful objects, and flowtables.
Each table belongs to a single protocol family.

```c
struct nft_table {
    struct list_head        list;           /* List node in per-ns table list */
    struct rhltable          chains_ht;      /* Hash table for chain lookup */
    struct list_head        chains;         /* Linked list of chains */
    struct list_head        sets;           /* Sets in this table */
    struct list_head        objects;        /* Stateful objects */
    struct list_head        flowtables;     /* Flow tables */
    u64                     hgenerator;     /* Handle generator */
    u64                     handle;         /* Unique table handle */
    u32                     use;            /* Chain reference count */
    u16                     family:6,       /* Address family */
                            flags:8,        /* NFT_TABLE_F_* */
                            genmask:2;      /* Generation mask */
    u32                     nlpid;          /* Netlink port ID of owner */
    char                    *name;          /* Table name */
    u16                     udlen;          /* User data length */
    u8                      *udata;         /* User data */
    u8                      validate_state; /* Transaction validation state */
};
```

### struct nft_chain (include/net/netfilter/nf_tables.h:1117)

A chain is a container for rules. Base chains are attached to netfilter
hooks; regular chains can be jumped to from rules.

```c
struct nft_chain {
    struct nft_rule_blob    __rcu *blob_gen_0; /* Rules for generation 0 */
    struct nft_rule_blob    __rcu *blob_gen_1; /* Rules for generation 1 */
    struct list_head        rules;          /* Linked list of rules */
    struct list_head        list;           /* Table's chain list */
    struct rhlist_head      rhlhead;        /* Table's chain hash */
    struct nft_table        *table;         /* Owning table */
    u64                     handle;         /* Unique chain handle */
    u32                     use;            /* Jump reference count */
    u8                      flags:5,        /* NFT_CHAIN_BASE, etc. */
                            bound:1,        /* Bound to rule */
                            genmask:2;      /* Generation mask */
    char                    *name;          /* Chain name */
    u16                     udlen;          /* User data length */
    u8                      *udata;         /* User data */
    struct nft_rule_blob    *blob_next;     /* Next gen blob (commit phase) */
};
```

### struct nft_base_chain (include/net/netfilter/nf_tables.h:1218)

A base chain extends `nft_chain` with a netfilter hook registration,
chain type, default policy, and per-CPU statistics.

```c
struct nft_base_chain {
    struct nf_hook_ops          ops;         /* Netfilter hook registration */
    struct list_head            hook_list;   /* Per-device hooks (NETDEV) */
    const struct nft_chain_type *type;       /* Chain type (filter/route/nat) */
    u8                          policy;      /* Default policy (NF_ACCEPT/DROP) */
    u8                          flags;       /* Disabled flag */
    struct nft_stats __percpu   *stats;      /* Per-CPU byte/packet counters */
    struct nft_chain            chain;       /* Embedded chain (MUST BE LAST) */
    struct flow_block           flow_block;  /* Hardware offload block */
};
```

### Chain Types (include/net/netfilter/nf_tables.h:1145)

```c
enum nft_chain_types {
    NFT_CHAIN_T_DEFAULT = 0,  /* filter -- standard packet filtering */
    NFT_CHAIN_T_ROUTE,        /* route -- reroute after mangling */
    NFT_CHAIN_T_NAT,          /* nat   -- NAT chains */
    NFT_CHAIN_T_MAX
};
```

Each chain type specifies which hooks it may attach to and what hook
function to use (include/net/netfilter/nf_tables.h:1164):

```c
struct nft_chain_type {
    const char              *name;
    enum nft_chain_types    type;
    int                     family;
    struct module           *owner;
    unsigned int            hook_mask;          /* Bitmask of valid hooks */
    nf_hookfn               *hooks[NFT_MAX_HOOKS]; /* Hook functions */
    int (*ops_register)(struct net *net, const struct nf_hook_ops *ops);
    void (*ops_unregister)(struct net *net, const struct nf_hook_ops *ops);
};
```

### struct nft_rule (include/net/netfilter/nf_tables.h:1001)

A rule is a sequence of expressions packed into a variable-length data
area. The `dlen` field gives the total length of expression data.

```c
struct nft_rule {
    struct list_head    list;           /* Chain's rule list */
    u64                 handle:42,     /* Unique rule handle */
                        genmask:2,     /* Generation mask */
                        dlen:12,       /* Expression data length */
                        udata:1;       /* Has user data appended */
    unsigned char       data[]         /* Expression data */
        __attribute__((aligned(__alignof__(struct nft_expr))));
};
```

Expressions within a rule are iterated with:

```c
#define nft_rule_for_each_expr(expr, last, rule)                    \
    for ((expr) = nft_expr_first(rule), (last) = nft_expr_last(rule); \
         (expr) != (last);                                          \
         (expr) = nft_expr_next(expr))
```

---

## Expressions and the Evaluation Loop

### struct nft_expr (include/net/netfilter/nf_tables.h:408)

An expression is the smallest unit of packet inspection or manipulation in
nf_tables. Each expression has an operations pointer and a variable-length
private data area.

```c
struct nft_expr {
    const struct nft_expr_ops   *ops;     /* Expression operations */
    unsigned char               data[]    /* Private data */
        __attribute__((aligned(__alignof__(u64))));
};
```

### struct nft_expr_ops (include/net/netfilter/nf_tables.h:952)

The expression operations table defines how an expression is evaluated,
initialized, destroyed, and optionally offloaded to hardware.

```c
struct nft_expr_ops {
    void    (*eval)(const struct nft_expr *expr,
                    struct nft_regs *regs,
                    const struct nft_pktinfo *pkt);    /* Packet evaluation */
    int     (*clone)(struct nft_expr *dst,
                     const struct nft_expr *src, gfp_t gfp);
    unsigned int            size;                       /* Total expr size */
    int     (*init)(const struct nft_ctx *ctx,
                    const struct nft_expr *expr,
                    const struct nlattr * const tb[]);  /* Initialize */
    void    (*destroy)(const struct nft_ctx *ctx,
                       const struct nft_expr *expr);    /* Destroy */
    int     (*dump)(struct sk_buff *skb,
                    const struct nft_expr *expr, bool reset); /* Dump to NL */
    bool    (*reduce)(struct nft_regs_track *track,
                      const struct nft_expr *expr);     /* Optimization */
    int     (*offload)(struct nft_offload_ctx *ctx,
                       struct nft_flow_rule *flow,
                       const struct nft_expr *expr);    /* HW offload */
    const struct nft_expr_type  *type;
};
```

### Register Set (include/net/netfilter/nf_tables.h:119)

The nft virtual machine operates on a set of 20 u32 registers. The first
four registers alias to the verdict register.

```c
#define NFT_REG32_NUM   20

struct nft_regs {
    union {
        u32                 data[NFT_REG32_NUM]; /* Data registers */
        struct nft_verdict  verdict;             /* Verdict register */
    };
};

struct nft_verdict {
    u32                 code;    /* NF_ACCEPT, NF_DROP, NFT_JUMP, etc. */
    struct nft_chain    *chain;  /* Target chain for JUMP/GOTO */
};
```

### Packet Info (include/net/netfilter/nf_tables.h:30)

```c
struct nft_pktinfo {
    struct sk_buff              *skb;       /* The packet */
    const struct nf_hook_state  *state;     /* Hook context */
    u8                          flags;      /* NFT_PKTINFO_L4PROTO, etc. */
    u8                          tprot;      /* Transport protocol */
    u16                         fragoff;    /* Fragment offset */
    u16                         thoff;      /* Transport header offset */
    u16                         inneroff;   /* Inner header offset */
};
```

### nft_do_chain() -- The Evaluation Loop (net/netfilter/nf_tables_core.c:252)

This is the core packet-path function called from the netfilter hook. It
walks the rule blob for the current generation, evaluating each expression
in sequence. Common expressions (cmp, payload, bitwise) have fast-path
inlined variants to avoid indirect calls (retpoline mitigation).

```c
unsigned int
nft_do_chain(struct nft_pktinfo *pkt, void *priv)
{
    const struct nft_chain *chain = priv, *basechain = chain;
    const struct nft_expr *expr, *last;
    const struct nft_rule_dp *rule;
    struct nft_regs regs;
    unsigned int stackptr = 0;
    struct nft_jumpstack jumpstack[NFT_JUMP_STACK_SIZE]; /* 16 deep */
    bool genbit = READ_ONCE(net->nft.gencursor);
    struct nft_rule_blob *blob;

do_chain:
    blob = genbit ? rcu_dereference(chain->blob_gen_1)
                  : rcu_dereference(chain->blob_gen_0);
    rule = (struct nft_rule_dp *)blob->data;

next_rule:
    regs.verdict.code = NFT_CONTINUE;
    for (; !rule->is_last; rule = nft_rule_next(rule)) {
        /* Evaluate each expression in the rule */
        nft_rule_dp_for_each_expr(expr, last, rule) {
            /* Fast-path: inline common expressions */
            if (expr->ops == &nft_cmp_fast_ops)
                nft_cmp_fast_eval(expr, &regs);
            else if (expr->ops == &nft_payload_fast_ops)
                nft_payload_fast_eval(expr, &regs, pkt);
            else
                expr_call_ops_eval(expr, &regs, pkt);

            if (regs.verdict.code != NFT_CONTINUE)
                break;
        }

        switch (regs.verdict.code) {
        case NFT_BREAK:     /* Expression didn't match, try next rule */
            regs.verdict.code = NFT_CONTINUE;
            continue;
        case NFT_CONTINUE:  /* All exprs matched, continue to next rule */
            continue;
        }
        break;              /* Terminal verdict, stop rule iteration */
    }

    switch (regs.verdict.code & NF_VERDICT_MASK) {
    case NF_ACCEPT: case NF_DROP: case NF_QUEUE: case NF_STOLEN:
        return regs.verdict.code;
    }

    switch (regs.verdict.code) {
    case NFT_JUMP:
        jumpstack[stackptr++].rule = nft_rule_next(rule);
        /* fall through */
    case NFT_GOTO:
        chain = regs.verdict.chain;
        goto do_chain;
    case NFT_RETURN:
        break;
    }

    if (stackptr > 0) {
        rule = jumpstack[--stackptr].rule;
        goto next_rule;
    }

    /* Fell off end of base chain -- apply default policy */
    return nft_base_chain(basechain)->policy;
}
```

### Built-in Expression Types (net/netfilter/nf_tables_core.c:353)

```c
static struct nft_expr_type *nft_basic_types[] = {
    &nft_imm_type,          /* Immediate value load */
    &nft_cmp_type,          /* Comparison */
    &nft_lookup_type,       /* Set/map lookup */
    &nft_bitwise_type,      /* Bitwise operations */
    &nft_byteorder_type,    /* Byte order conversion */
    &nft_payload_type,      /* Payload extraction */
    &nft_dynset_type,       /* Dynamic set update */
    &nft_range_type,        /* Range comparison */
    &nft_meta_type,         /* Packet metadata (mark, iif, etc.) */
    &nft_rt_type,           /* Routing info */
    &nft_exthdr_type,       /* Extension header matching */
    &nft_last_type,         /* Last-seen timestamp */
    &nft_counter_type,      /* Packet/byte counter */
    &nft_objref_type,       /* Stateful object reference */
    &nft_inner_type,        /* Inner packet matching (tunnels) */
};
```

Additional expressions are loaded as modules: `nft_ct` (conntrack),
`nft_nat`, `nft_log`, `nft_limit`, `nft_quota`, `nft_reject`,
`nft_masq`, `nft_redir`, `nft_hash`, `nft_fib`, `nft_socket`,
`nft_tproxy`, `nft_synproxy`, `nft_connlimit`, `nft_tunnel`,
`nft_xfrm`, `nft_osf`.

---

## Sets and Maps

Sets and maps are nf_tables data structures for efficient lookups. A set
is a collection of keys; a map associates keys with data values (including
verdict values for "verdict maps").

### struct nft_set (include/net/netfilter/nf_tables.h:585)

```c
struct nft_set {
    struct list_head            list;       /* Table's set list */
    struct list_head            bindings;   /* Rules bound to this set */
    refcount_t                  refs;       /* Async GC refcount */
    struct nft_table            *table;     /* Owning table */
    possible_net_t              net;        /* Network namespace */
    char                        *name;      /* Set name */
    u64                         handle;     /* Unique handle */
    u32                         ktype;      /* Key type */
    u32                         dtype;      /* Data type (or NFT_DATA_VERDICT) */
    u32                         objtype;    /* Object type for objref maps */
    u32                         size;       /* Maximum number of elements */
    u8                          field_len[NFT_REG32_COUNT]; /* Concatenation */
    u8                          field_count;/* Number of concat fields */
    u32                         use;        /* Rule reference count */
    atomic_t                    nelems;     /* Current element count */
    u64                         timeout;    /* Default element timeout */
    u32                         gc_int;     /* GC interval (msecs) */
    u16                         policy;     /* Set implementation policy */
    /* --- runtime data (cacheline aligned) --- */
    const struct nft_set_ops    *ops;       /* Backend implementation */
    u16                         flags:13,   /* NFT_SET_* flags */
                                dead:1,     /* Being freed */
                                genmask:2;  /* Generation mask */
    u8                          klen;       /* Key length */
    u8                          dlen;       /* Data length */
    u8                          num_exprs;  /* Attached expressions */
    struct nft_expr             *exprs[NFT_SET_EXPR_MAX]; /* Expressions */
    struct list_head            catchall_list; /* Catch-all elements */
    unsigned char               data[];     /* Backend private data */
};
```

### struct nft_set_ops (include/net/netfilter/nf_tables.h:461)

Each set backend (hash, rbtree, bitmap, pipapo) implements this interface:

```c
struct nft_set_ops {
    /* Fast path -- packet processing */
    bool    (*lookup)(const struct net *net, const struct nft_set *set,
                      const u32 *key, const struct nft_set_ext **ext);
    bool    (*update)(struct nft_set *set, const u32 *key, ...);
    bool    (*delete)(const struct nft_set *set, const u32 *key);

    /* Control path -- element management */
    int     (*insert)(const struct net *net, const struct nft_set *set,
                      const struct nft_set_elem *elem, ...);
    void    (*activate)(const struct net *net, const struct nft_set *set,
                        struct nft_elem_priv *elem_priv);
    struct nft_elem_priv *(*deactivate)(...);
    void    (*remove)(...);
    void    (*walk)(const struct nft_ctx *ctx, struct nft_set *set,
                    struct nft_set_iter *iter);

    /* Sizing and initialization */
    bool    (*estimate)(const struct nft_set_desc *desc, u32 features,
                        struct nft_set_estimate *est);
    int     (*init)(const struct nft_set *set, const struct nft_set_desc *desc,
                    const struct nlattr * const nla[]);
    void    (*destroy)(const struct nft_ctx *ctx, const struct nft_set *set);
    void    (*gc_init)(const struct nft_set *set);

    unsigned int    elemsize;   /* Per-element private data size */
};
```

### Set Backends

| Backend | File | Lookup | Use Case |
|---------|------|--------|----------|
| `nft_set_hash` | `net/netfilter/nft_set_hash.c` | O(1) hash | Exact match sets |
| `nft_set_rbtree` | `net/netfilter/nft_set_rbtree.c` | O(log N) | Interval/range sets |
| `nft_set_bitmap` | `net/netfilter/nft_set_bitmap.c` | O(1) bitmap | Small integer keys |
| `nft_set_pipapo` | `net/netfilter/nft_set_pipapo.c` | O(fields) | Multi-field concat |

### Performance Classes (include/net/netfilter/nf_tables.h:378)

```c
enum nft_set_class {
    NFT_SET_CLASS_O_1,       /* Constant time */
    NFT_SET_CLASS_O_LOG_N,   /* Logarithmic */
    NFT_SET_CLASS_O_N,       /* Linear */
};
```

### Set Extensions (include/net/netfilter/nf_tables.h:701)

Set elements can carry optional extensions for timeout, counters,
expressions, and object references:

```c
enum nft_set_extensions {
    NFT_SET_EXT_KEY,           /* Element key */
    NFT_SET_EXT_KEY_END,       /* Upper bound key (ranges) */
    NFT_SET_EXT_DATA,          /* Mapping data */
    NFT_SET_EXT_FLAGS,         /* Element flags */
    NFT_SET_EXT_TIMEOUT,       /* Element timeout + expiration */
    NFT_SET_EXT_USERDATA,      /* User-defined data */
    NFT_SET_EXT_EXPRESSIONS,   /* Attached expressions */
    NFT_SET_EXT_OBJREF,        /* Stateful object reference */
    NFT_SET_EXT_NUM
};
```

### Verdict Maps

A verdict map is a set with `dtype == NFT_DATA_VERDICT`, where each
element's data is a verdict (accept, drop, jump, goto). This enables
efficient multi-way branching:

```
nft add map inet filter my_vmap { type ipv4_addr : verdict ; }
nft add element inet filter my_vmap { 10.0.0.1 : accept, \
                                       10.0.0.2 : drop }
nft add rule inet filter input ip saddr vmap @my_vmap
```

---

## Connection Tracking (conntrack)

Connection tracking (conntrack) provides stateful packet inspection by
maintaining a table of active network connections. Every packet traversing
netfilter is associated with a connection entry, enabling rules to match
on connection state (NEW, ESTABLISHED, RELATED).

### struct nf_conn (include/net/netfilter/nf_conntrack.h:75)

The `nf_conn` structure represents a single tracked connection:

```c
struct nf_conn {
    struct nf_conntrack ct_general;         /* Refcount (via skb->_nfct) */
    spinlock_t          lock;               /* Per-entry lock */
    u32                 timeout;            /* Jiffies when considered dead */

    /* Tuples for both directions */
    struct nf_conntrack_tuple_hash tuplehash[IP_CT_DIR_MAX];

    unsigned long       status;             /* IPS_* status bits */
    possible_net_t      ct_net;             /* Network namespace */

#if IS_ENABLED(CONFIG_NF_NAT)
    struct hlist_node   nat_bysource;       /* NAT source hash */
#endif

    struct nf_conn      *master;            /* Master (for RELATED) */

#if defined(CONFIG_NF_CONNTRACK_MARK)
    u_int32_t           mark;               /* User-settable mark */
#endif

    struct nf_ct_ext    *ext;               /* Extensions (NAT, acct, etc.) */
    union nf_conntrack_proto proto;          /* Protocol-specific state */
};
```

### Connection Tuple (include/net/netfilter/nf_conntrack_tuple.h:37)

Each connection is identified by a tuple that encodes the 5-tuple
(src IP, dst IP, src port, dst port, protocol) plus the L3 protocol:

```c
struct nf_conntrack_tuple {
    struct nf_conntrack_man src;            /* Source: addr + port + l3num */
    struct {
        union nf_inet_addr u3;              /* Dest address */
        union {
            __be16 all;
            struct { __be16 port; } tcp;
            struct { __be16 port; } udp;
            struct { u_int8_t type, code; } icmp;
            /* ... dccp, sctp, gre ... */
        } u;
        u_int8_t protonum;                  /* L4 protocol number */
        u_int8_t dir;                       /* Direction (original/reply) */
    } dst;
};

/* Connections have two entries in the hash: one per direction */
struct nf_conntrack_tuple_hash {
    struct hlist_nulls_node hnnode;
    struct nf_conntrack_tuple tuple;
};
```

### Connection States (include/uapi/linux/netfilter/nf_conntrack_common.h:7)

```c
enum ip_conntrack_info {
    IP_CT_ESTABLISHED,           /* Part of established connection */
    IP_CT_RELATED,               /* Related to existing connection */
    IP_CT_NEW,                   /* New connection */
    IP_CT_IS_REPLY,              /* >= this means reply direction */
    IP_CT_ESTABLISHED_REPLY = IP_CT_ESTABLISHED + IP_CT_IS_REPLY,
    IP_CT_RELATED_REPLY     = IP_CT_RELATED     + IP_CT_IS_REPLY,
};
```

### Status Bits (include/uapi/linux/netfilter/nf_conntrack_common.h:44)

```c
/* Key status bits (bitmask in ct->status) */
IPS_EXPECTED       (1 << 0)   /* Expected by an expectation */
IPS_SEEN_REPLY     (1 << 1)   /* Seen reply traffic */
IPS_ASSURED        (1 << 2)   /* Won't be early-dropped */
IPS_CONFIRMED      (1 << 3)   /* Inserted into hash table */
IPS_SRC_NAT        (1 << 4)   /* Source NAT applied */
IPS_DST_NAT        (1 << 5)   /* Destination NAT applied */
IPS_DYING          (1 << 9)   /* Marked for deletion */
IPS_OFFLOAD        (1 << 14)  /* Offloaded to hardware/flowtable */
```

### Conntrack Hash Table (net/netfilter/nf_conntrack_core.c:63)

The conntrack hash table is a global array of `hlist_nulls_head` buckets,
protected by `CONNTRACK_LOCKS` (1024) striped spinlocks:

```c
__cacheline_aligned_in_smp spinlock_t nf_conntrack_locks[CONNTRACK_LOCKS];
struct hlist_nulls_head *nf_conntrack_hash __read_mostly;
```

The table supports dynamic resizing. Hash operations use `siphash` for
security against hash-flooding attacks.

### Conntrack Lifecycle

```
  1. Packet arrives at NF_INET_PRE_ROUTING
     |
     v
  2. nf_conntrack_in() at priority -200
     - Extract tuple from packet
     - Look up in hash table
     - If not found: allocate new nf_conn (NEW)
     - If found: update state (ESTABLISHED/RELATED)
     - Attach ct to skb via skb->_nfct
     |
     v
  3. Packet traverses filter/mangle/NAT hooks
     - Rules can match on ct state
     - NAT modifies reply tuple
     |
     v
  4. NF_INET_POST_ROUTING (or LOCAL_IN)
     - nf_conntrack_confirm() at priority INT_MAX
     - Inserts unconfirmed entries into hash table
     - Sets IPS_CONFIRMED bit
     |
     v
  5. Garbage collection (conntrack_gc_work)
     - Periodic scan of hash table
     - Removes expired entries
     - Adjustable interval: 1s to 60s
```

### Conntrack Garbage Collection (net/netfilter/nf_conntrack_core.c:66)

```c
struct conntrack_gc_work {
    struct delayed_work  dwork;
    u32                  next_bucket;    /* Resume point in hash */
    u32                  avg_timeout;    /* Average timeout seen */
    u32                  count;          /* Entries scanned */
    u32                  start_time;     /* Scan start */
    bool                 exiting;        /* Module exit flag */
    bool                 early_drop;     /* Under memory pressure */
};

#define GC_SCAN_INTERVAL_MAX    (60ul * HZ)
#define GC_SCAN_INTERVAL_MIN    (1ul  * HZ)
#define GC_SCAN_MAX_DURATION    msecs_to_jiffies(10) /* Max 10ms per run */
```

### Protocol-Specific Tracking (include/net/netfilter/nf_conntrack.h:32)

```c
union nf_conntrack_proto {
    struct nf_ct_dccp dccp;   /* DCCP state */
    struct ip_ct_sctp sctp;   /* SCTP state */
    struct ip_ct_tcp  tcp;    /* TCP state (sequence tracking, flags) */
    struct nf_ct_udp  udp;    /* UDP stream detection */
    struct nf_ct_gre  gre;    /* GRE key tracking */
};
```

TCP tracking is the most complex, maintaining per-direction sequence
number windows, TCP flags seen, retransmission detection, and window
scaling tracking in `struct ip_ct_tcp`.

---

## Network Address Translation (NAT)

NAT in netfilter is built on top of connection tracking. When a NAT rule
matches a NEW connection, the conntrack reply tuple is modified. All
subsequent packets in that connection are translated based on the stored
mapping.

### NAT Manipulation Types (include/net/netfilter/nf_nat.h:13)

```c
enum nf_nat_manip_type {
    NF_NAT_MANIP_SRC,    /* Source NAT (POSTROUTING or LOCAL_IN) */
    NF_NAT_MANIP_DST     /* Destination NAT (PREROUTING or LOCAL_OUT) */
};

/* Determine manipulation type from hook */
#define HOOK2MANIP(hooknum) ((hooknum) != NF_INET_POST_ROUTING && \
                             (hooknum) != NF_INET_LOCAL_IN)
```

### NAT Range (include/uapi/linux/netfilter/nf_nat.h)

NAT rules specify a range of addresses and/or ports to map to:

```
nft add rule nat postrouting oif eth0 masquerade
nft add rule nat prerouting tcp dport 80 dnat to 192.168.1.10:8080
nft add rule nat postrouting snat to 10.0.0.1-10.0.0.5
```

### NAT Flow

```
  DNAT (Destination NAT):
  +--------------------+           +-------------------+
  | NF_INET_PRE_ROUTING|           | NF_INET_LOCAL_OUT |
  | priority -100      |           | priority -100     |
  +--------------------+           +-------------------+
          |                                |
          v                                v
  Modify dst addr/port            Modify dst addr/port
  in conntrack reply tuple        in conntrack reply tuple
          |                                |
          v                                v
  Routing uses new dst            Routing uses new dst

  SNAT (Source NAT):
  +---------------------+          +-------------------+
  | NF_INET_POST_ROUTING|          | NF_INET_LOCAL_IN  |
  | priority 100        |          | priority 100      |
  +---------------------+          +-------------------+
          |                                |
          v                                v
  Modify src addr/port            Modify src addr/port
  in conntrack reply tuple        in conntrack reply tuple
```

### Key NAT Functions (include/net/netfilter/nf_nat.h)

```c
/* Set up NAT mapping for a connection */
unsigned int nf_nat_setup_info(struct nf_conn *ct,
                               const struct nf_nat_range2 *range,
                               enum nf_nat_manip_type maniptype);

/* Apply NAT translation to a packet */
unsigned int nf_nat_packet(struct nf_conn *ct,
                           enum ip_conntrack_info ctinfo,
                           unsigned int hooknum,
                           struct sk_buff *skb);

/* Manipulate packet headers for NAT */
unsigned int nf_nat_manip_pkt(struct sk_buff *skb, struct nf_conn *ct,
                              enum nf_nat_manip_type mtype,
                              enum ip_conntrack_dir dir);

/* Allocate null binding (identity NAT for conntrack) */
unsigned int nf_nat_alloc_null_binding(struct nf_conn *ct,
                                       unsigned int hooknum);
```

### Masquerading

Masquerading is SNAT where the source address is automatically set to the
outgoing interface's address. It handles interface address changes
gracefully. The `nf_conn_nat` extension (include/net/netfilter/nf_nat.h:31)
tracks the masquerade interface index to detect changes.

---

## Flowtables and Hardware Offload

Flowtables provide a software and hardware fast path for established
connections, bypassing the full netfilter hook traversal for flows that
have already been classified.

### struct nf_flowtable (include/net/netfilter/nf_flow_table.h:76)

```c
struct nf_flowtable {
    unsigned int                    flags;          /* NF_FLOWTABLE_HW_OFFLOAD */
    int                             priority;       /* Hook priority */
    struct rhashtable               rhashtable;     /* Flow lookup table */
    struct list_head                list;           /* Slow-path list */
    const struct nf_flowtable_type  *type;          /* Type callbacks */
    struct delayed_work             gc_work;        /* Garbage collection */
    struct flow_block               flow_block;     /* HW offload block */
    struct rw_semaphore             flow_block_lock;
    possible_net_t                  net;
};
```

### struct nft_flowtable (include/net/netfilter/nf_tables.h:1461)

The nft_flowtable wraps `nf_flowtable` within the nf_tables object model:

```c
struct nft_flowtable {
    struct list_head        list;           /* Table's flowtable list */
    struct nft_table        *table;         /* Owning table */
    char                    *name;          /* Flowtable name */
    int                     hooknum;        /* Ingress hook number */
    int                     ops_len;        /* Number of device hooks */
    u32                     genmask:2;      /* Generation mask */
    u32                     use;            /* Reference count */
    u64                     handle;         /* Unique handle */
    struct list_head        hook_list;      /* Per-device hooks */
    struct nf_flowtable     data;           /* Embedded flowtable */
};
```

### Flow Offload Tuple (include/net/netfilter/nf_flow_table.h:110)

```c
struct flow_offload_tuple {
    union { struct in_addr src_v4; struct in6_addr src_v6; };
    union { struct in_addr dst_v4; struct in6_addr dst_v6; };
    struct { __be16 src_port; __be16 dst_port; };
    int             iifidx;         /* Input interface index */
    u8              l3proto;        /* L3 protocol */
    u8              l4proto;        /* L4 protocol */
    /* ... encapsulation, xmit_type, MTU, dst/neigh cache ... */
};
```

### Flowtable Operation

```
  Packet arrives
       |
       v
  +---------------------------+
  | NF_INET_PRE_ROUTING       |
  | Flowtable hook (pri -100) |
  +------------+--------------+
               |
       +-------+-------+
       |               |
       v               v
   (flow found)    (flow not found)
       |               |
       v               v
  Fast path:       Normal path:
  - NAT manip      - Full netfilter
  - Update TTL      hook traversal
  - Forward         - conntrack
  - Skip all        - filter
    other hooks     - etc.
```

### Hardware Offload

When `NF_FLOWTABLE_HW_OFFLOAD` is set, the kernel programs the NIC
hardware to handle matching flows entirely in silicon, achieving line-rate
forwarding without any CPU involvement. The `flow_block` infrastructure
communicates flow rules to NIC drivers via `flow_cls_offload` messages.

```
nft add flowtable inet filter f { hook ingress priority filter ; \
                                   devices = { eth0, eth1 } ; \
                                   flags offload ; }
nft add rule inet filter forward ct state established flow add @f
```

---

## iptables vs nftables

### Comparison

```
+---------------------+----------------------------+---------------------------+
|                     | iptables                   | nftables                  |
+---------------------+----------------------------+---------------------------+
| Kernel component    | x_tables + ip_tables       | nf_tables                 |
| User tool           | iptables, ip6tables,       | nft                       |
|                     | arptables, ebtables        | (single unified tool)     |
| Protocol families   | Separate binary per family | Single framework          |
| Rule evaluation     | Linear match+target        | Expression VM             |
| Atomic updates      | Per-rule (iptables-restore | Full-transaction commit   |
|                     | for batch)                 | (always atomic)           |
| Sets/maps           | ipset (separate module)    | Built-in sets, maps,      |
|                     |                            | verdict maps, concat      |
| Kernel API          | Separate per table type    | Single Netlink-based API  |
| Extensibility       | Match + target modules     | Expression modules        |
| Performance         | Linear rule scan           | Set lookups, JIT compare  |
|                     |                            | fast ops for common exprs |
| HW offload          | Not supported              | Supported via flowtables  |
+---------------------+----------------------------+---------------------------+
```

### Compatibility Layer

The kernel provides `iptables-nft`, which translates iptables commands
into nf_tables rules internally. The `x_tables` infrastructure
(include/linux/netfilter/x_tables.h) remains for legacy compatibility.
Both iptables-legacy (using ip_tables kernel module) and iptables-nft
(using nf_tables kernel module) can coexist, but they must not manage
the same tables.

### nftables Advantages

1. **Atomic ruleset replacement**: Entire rulesets are replaced atomically
   via the transaction/generation mechanism.

2. **Generic set infrastructure**: Built-in support for IP sets, port
   sets, interface sets, and concatenated sets. Verdict maps enable
   efficient multi-way branching.

3. **Reduced kernel/userspace copying**: The nftables VM evaluates
   multiple conditions within a single rule, reducing per-packet overhead.

4. **Single framework**: One tool (`nft`) handles IPv4, IPv6, ARP, bridge,
   and netdev filtering.

5. **Flowtable offload**: Native support for software and hardware flow
   offloading.

---

## Locking and Concurrency

### Control Plane (Rule Management)

| Lock | Type | Protects | File |
|------|------|----------|------|
| `nf_hook_mutex` | mutex | Hook registration/unregistration | `net/netfilter/core.c:42` |
| `nft_net->commit_mutex` | mutex | nf_tables transaction commit | `net/netfilter/nf_tables_api.c` |
| `nf_tables_destroy_list_lock` | spinlock | Deferred object destruction list | `net/netfilter/nf_tables_api.c:39` |
| `nf_tables_gc_list_lock` | spinlock | Set element GC list | `net/netfilter/nf_tables_api.c:40` |

### Data Plane (Packet Processing)

| Mechanism | Protects | Details |
|-----------|----------|---------|
| RCU | Hook entries, rule blobs, conntrack lookup | `nf_hook()` holds `rcu_read_lock()` |
| Generation cursor | nf_tables objects | 2-bit genmask for lockless read/commit |
| Per-bucket spinlocks | Conntrack hash table | `nf_conntrack_locks[CONNTRACK_LOCKS]` (1024 locks) |
| `nf_conntrack_expect_lock` | Expectation table | Global spinlock |
| `nf_conntrack_mutex` | Hash resize, cleanup iteration | Serializes structural changes |

### RCU Usage

The data plane is fully RCU-protected. Hook entries are arrays swapped
atomically via `rcu_assign_pointer()` and freed via `call_rcu()`. The
nf_tables rule blobs are similarly RCU-protected with two generation
pointers (`blob_gen_0`, `blob_gen_1`).

Hook registration (`nf_register_net_hook`) takes `nf_hook_mutex`, allocates
a new `nf_hook_entries` array with the new hook inserted in priority order,
publishes it with `rcu_assign_pointer()`, and frees the old array after an
RCU grace period (core.c:75).

Hook unregistration replaces the to-be-removed hook with a `dummy_ops`
(accept-all) entry atomically (using `WRITE_ONCE`), then shrinks the array
after the RCU grace period. This guarantees unregistration never fails
(core.c:470).

### Conntrack Locking

Conntrack uses 1024 striped spinlocks indexed by hash bucket. Double-lock
ordering is enforced to prevent deadlocks when manipulating entries that
appear in two hash chains (original and reply direction). The locking also
handles a global lock mode for hash table resizing
(nf_conntrack_core.c:104).

---

## Source Files Reference

### Core Framework

| File | Purpose | Key Functions |
|------|---------|---------------|
| `net/netfilter/core.c` | Hook framework, registration, dispatch | `nf_hook_slow()`, `nf_register_net_hook()`, `nf_hook_entries_grow()` |
| `include/linux/netfilter.h` | Hook structures, NF_HOOK macro | `nf_hook()`, `NF_HOOK()`, `nf_hook_state_init()` |
| `include/uapi/linux/netfilter.h` | Hook enums, verdict codes, NFPROTO | `enum nf_inet_hooks`, `NF_DROP`/`NF_ACCEPT` |
| `include/uapi/linux/netfilter_ipv4.h` | IPv4 hook priorities | `enum nf_ip_hook_priorities` |

### nf_tables

| File | Purpose | Key Functions |
|------|---------|---------------|
| `net/netfilter/nf_tables_api.c` | Control plane: Netlink API, transactions | `nft_ctx_init()`, transaction commit/abort |
| `net/netfilter/nf_tables_core.c` | Data plane: rule evaluation | `nft_do_chain()`, `expr_call_ops_eval()` |
| `include/net/netfilter/nf_tables.h` | All nft structures and inline helpers | `struct nft_table`, `nft_chain`, `nft_rule`, `nft_expr`, `nft_set` |
| `net/netfilter/nft_set_hash.c` | Hash-based set backend | Exact match lookups |
| `net/netfilter/nft_set_rbtree.c` | Red-black tree set backend | Interval/range lookups |
| `net/netfilter/nft_set_pipapo.c` | PIPAPO set backend | Multi-field concatenation |
| `net/netfilter/nft_set_bitmap.c` | Bitmap set backend | Small integer key sets |
| `net/netfilter/nft_payload.c` | Payload expression | `nft_payload_eval()`, `nft_payload_fast_eval()` |
| `net/netfilter/nft_cmp.c` | Comparison expression | `nft_cmp_eval()`, `nft_cmp_fast_eval()` |
| `net/netfilter/nft_immediate.c` | Immediate load expression | `nft_immediate_eval()` |
| `net/netfilter/nft_lookup.c` | Set lookup expression | `nft_lookup_eval()` |
| `net/netfilter/nft_bitwise.c` | Bitwise operations | `nft_bitwise_eval()` |
| `net/netfilter/nft_meta.c` | Metadata extraction (mark, iif, etc.) | `nft_meta_get_eval()` |
| `net/netfilter/nft_ct.c` | Conntrack matching/setting | `nft_ct_get_fast_eval()` |
| `net/netfilter/nft_nat.c` | NAT expression | SNAT/DNAT actions |
| `net/netfilter/nft_log.c` | Logging expression | Packet logging |
| `net/netfilter/nft_counter.c` | Counter expression/object | Byte/packet counting |
| `net/netfilter/nft_limit.c` | Rate limiting | Token bucket algorithm |
| `net/netfilter/nft_dynset.c` | Dynamic set updates | Add/update elements from packet path |
| `net/netfilter/nft_range.c` | Range comparison | Value range matching |
| `net/netfilter/nft_reject.c` | Reject with ICMP/TCP-RST | `nft_reject_eval()` |
| `net/netfilter/nft_masq.c` | Masquerade expression | Dynamic SNAT |
| `net/netfilter/nft_redir.c` | Port redirect expression | DNAT to local ports |
| `net/netfilter/nft_fib.c` | FIB lookup expression | Routing-based matching |
| `net/netfilter/nft_inner.c` | Inner packet matching | Tunnel payload inspection |
| `net/netfilter/nf_tables_offload.c` | Hardware offload infrastructure | Flow rule translation |

### Connection Tracking

| File | Purpose | Key Functions |
|------|---------|---------------|
| `net/netfilter/nf_conntrack_core.c` | Core conntrack: hash table, GC, lifecycle | `nf_conntrack_in()`, `nf_conntrack_confirm()`, `nf_conntrack_alloc()` |
| `include/net/netfilter/nf_conntrack.h` | `struct nf_conn` and conntrack API | `nf_ct_get()`, `nf_ct_put()`, `nf_ct_is_confirmed()` |
| `include/net/netfilter/nf_conntrack_tuple.h` | Connection tuple definitions | `struct nf_conntrack_tuple`, `nf_ct_tuple_equal()` |
| `include/uapi/linux/netfilter/nf_conntrack_common.h` | CT state enum, status bits | `enum ip_conntrack_info`, `IPS_*` flags |
| `net/netfilter/nf_conntrack_proto_tcp.c` | TCP state tracking | Sequence validation, state machine |
| `net/netfilter/nf_conntrack_proto_udp.c` | UDP state tracking | Stream detection |
| `net/netfilter/nf_conntrack_proto_icmp.c` | ICMP tracking | ICMP error correlation |
| `net/netfilter/nf_conntrack_expect.c` | Expectation framework | ALG support (FTP, SIP, etc.) |
| `net/netfilter/nf_conntrack_helper.c` | CT helper framework | Protocol helper registration |
| `net/netfilter/nf_conntrack_extend.c` | CT extension mechanism | Dynamic per-ct storage |

### NAT

| File | Purpose | Key Functions |
|------|---------|---------------|
| `net/netfilter/nf_nat_core.c` | NAT core: mapping, packet manip | `nf_nat_setup_info()`, `nf_nat_packet()` |
| `include/net/netfilter/nf_nat.h` | NAT structures and API | `enum nf_nat_manip_type`, `nf_nat_manip_pkt()` |
| `net/netfilter/nf_nat_masquerade.c` | Masquerade support | Interface address tracking |
| `net/netfilter/nf_nat_redirect.c` | Port redirect (local DNAT) | `nf_nat_redirect_ipv4/6()` |

### Flow Tables

| File | Purpose | Key Functions |
|------|---------|---------------|
| `net/netfilter/nf_flow_table_core.c` | Flow table core: insertion, lookup, GC | `flow_offload_alloc()`, `flow_offload_add()` |
| `net/netfilter/nf_flow_table_ip.c` | IPv4/IPv6 flow forwarding | `nf_flow_offload_ip_hook()` |
| `net/netfilter/nf_flow_table_offload.c` | Hardware offload translation | Flow rule programming to NIC drivers |
| `include/net/netfilter/nf_flow_table.h` | Flow table structures | `struct nf_flowtable`, `struct flow_offload_tuple` |

### Headers

| File | Purpose |
|------|---------|
| `include/linux/netfilter.h` | Core netfilter API, hook state, NF_HOOK macros |
| `include/linux/netfilter_defs.h` | NF_MAX_HOOKS and related constants |
| `include/uapi/linux/netfilter.h` | UAPI: verdicts, hook enums, protocol families |
| `include/uapi/linux/netfilter/nf_tables.h` | UAPI: nft Netlink message types and attributes |
| `include/net/netfilter/nf_tables.h` | Internal nft structures: table, chain, rule, set, expr |
| `include/net/netfilter/nf_tables_core.h` | Fast-path expression declarations |
| `include/net/netfilter/nf_conntrack.h` | `struct nf_conn`, conntrack API |
| `include/net/netfilter/nf_conntrack_tuple.h` | Connection tuple structures |
| `include/net/netfilter/nf_nat.h` | NAT structures and functions |
| `include/net/netfilter/nf_flow_table.h` | Flowtable structures |
