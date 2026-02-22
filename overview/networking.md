# Linux Kernel Networking Stack Overview

## Table of Contents
1. [Introduction](#introduction)
2. [Network Stack Architecture](#network-stack-architecture)
3. [Socket Layer](#socket-layer)
4. [sk_buff (Socket Buffer)](#sk_buff-socket-buffer)
5. [Network Device Layer](#network-device-layer)
6. [NAPI (New API)](#napi-new-api)
7. [TCP/IP Stack](#tcpip-stack)
8. [Netfilter and iptables](#netfilter-and-iptables)
9. [XDP (eXpress Data Path)](#xdp-express-data-path)
10. [Traffic Control (tc)](#traffic-control-tc)

---

## Introduction

The Linux networking stack is one of the most sophisticated and optimized
subsystems in the kernel. It handles everything from raw packet processing at
the device level to high-level socket abstractions used by applications.

### Key Design Goals

```
┌─────────────────────────────────────────────────────────────┐
│              Networking Stack Design Goals                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Performance                                             │
│     - Zero-copy where possible                              │
│     - Efficient packet batching                             │
│     - Lock-free fast paths                                  │
│     - CPU cache optimization                                │
│                                                             │
│  2. Modularity                                              │
│     - Protocol-independent core                             │
│     - Pluggable protocol handlers                           │
│     - Extensible filtering (netfilter, XDP)                 │
│                                                             │
│  3. Scalability                                             │
│     - Per-CPU queues                                        │
│     - Multi-queue NIC support                               │
│     - Receive-side scaling (RSS)                            │
│     - Transmit packet steering (XPS)                        │
│                                                             │
│  4. Flexibility                                             │
│     - Network namespaces                                    │
│     - Virtual devices                                       │
│     - Traffic control                                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Network Stack Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      User Space                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Application (browser, server, etc.)                │    │
│  │  send(), recv(), connect(), bind(), accept()        │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
                              │
                     ─────────┴─────────────
                    │ System Call Interface │
                     ───────────────────────
                              │
┌─────────────────────────────────────────────────────────────┐
│                      Kernel Space                           │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              Socket Layer (BSD sockets)             │    │
│  │  struct socket, struct sock, protocol operations    │    │
│  └─────────────────────────────────────────────────────┘    │
│                              │                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              Protocol Layer (L4)                    │    │
│  │  TCP: struct tcp_sock    UDP: struct udp_sock       │    │
│  │  SCTP, DCCP, raw sockets                            │    │
│  └─────────────────────────────────────────────────────┘    │
│                              │                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              Network Layer (L3)                     │    │
│  │  IPv4: ip_rcv(), ip_output()                        │    │
│  │  IPv6: ipv6_rcv(), ip6_output()                     │    │
│  │  Routing: fib_lookup(), route cache                 │    │
│  └─────────────────────────────────────────────────────┘    │
│                             │                               │
│  ┌──────────────────────┬───┴───┬──────────────────────┐    │
│  │     Netfilter        │       │      XDP             │    │
│  │  iptables/nftables   │       │  (before stack)      │    │
│  └──────────────────────┴───────┴──────────────────────┘    │
│                              │                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │           Traffic Control (tc/qdisc)                │    │
│  └─────────────────────────────────────────────────────┘    │
│                              │                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              Link Layer (L2)                        │    │
│  │  struct net_device, dev_queue_xmit()                │    │
│  │  ARP, bridge, bonding, VLAN                         │    │
│  └─────────────────────────────────────────────────────┘    │
│                              │                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              Device Driver                          │    │
│  │  NAPI, DMA, ring buffers                            │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                              │
                    ──────────┴──────────
                    │   Physical NIC      │
                    ──────────────────────
```

---

## Socket Layer

The socket layer provides the user-space interface to networking through BSD
socket API.

### Core Data Structures

The socket structure is the kernel's representation of a BSD socket and serves
as the primary interface between user space and the networking stack. It is
created when an application calls the socket() system call and is associated
with a file descriptor for I/O operations. The structure contains state
information, type flags, a pointer to the underlying internal socket (struct
sock), and protocol-specific operations. Each socket maintains a wait queue for
blocking I/O operations like read and write.

```c
/* include/linux/net.h */
struct socket {
    socket_state            state;          /* SS_UNCONNECTED, etc. */
    short                   type;           /* SOCK_STREAM, SOCK_DGRAM */
    unsigned long           flags;          /* SOCK_NOSPACE, etc. */
    struct file             *file;          /* Associated file */
    struct sock             *sk;            /* Internal network socket */
    const struct proto_ops  *ops;           /* Protocol operations */
    struct socket_wq        wq;             /* Wait queue for I/O */
};

/* Socket states */
typedef enum {
    SS_FREE = 0,             /* Not allocated */
    SS_UNCONNECTED,          /* Unconnected to any socket */
    SS_CONNECTING,           /* In process of connecting */
    SS_CONNECTED,            /* Connected to socket */
    SS_DISCONNECTING         /* In process of disconnecting */
} socket_state;
```

The sock structure (commonly called "sk") is the protocol-independent internal
representation of a socket within the kernel. Unlike struct socket which is
user-facing, struct sock contains the actual networking state including send
and receive queues, buffer limits, and callback functions for state changes and
data arrival. It embeds sock_common for shared fields and is extended by
protocol-specific structures like tcp_sock and udp_sock. The sk_prot pointer
links to protocol handlers, allowing the same sock interface to support TCP,
UDP, and other protocols.

```c
/* include/net/sock.h */
struct sock {
    struct sock_common      __sk_common;

    /* Socket flags */
    unsigned long           sk_flags;

    /* Receive/Send buffers */
    struct sk_buff_head     sk_receive_queue;
    struct sk_buff_head     sk_write_queue;

    /* Buffer limits */
    int                     sk_rcvbuf;
    int                     sk_sndbuf;
    atomic_t                sk_wmem_alloc;
    atomic_t                sk_rmem_alloc;

    /* Callbacks */
    void (*sk_state_change)(struct sock *sk);
    void (*sk_data_ready)(struct sock *sk);
    void (*sk_write_space)(struct sock *sk);
    void (*sk_error_report)(struct sock *sk);

    /* Protocol-specific */
    struct proto            *sk_prot;

    /* ... many more fields ... */
};

The sock_common structure contains the fields shared by all socket types and is
embedded at the beginning of struct sock. It holds fundamental connection
information including address family, connection state, local and remote
addresses (for both IPv4 and IPv6), and port numbers. The structure also
contains hash table links for efficient socket lookup and a pointer to the
network namespace. By factoring out these common fields, the kernel reduces
code duplication across different protocol implementations.

```c
/* Common socket fields */
struct sock_common {
    unsigned short          skc_family;     /* Address family */
    volatile unsigned char  skc_state;      /* Connection state */
    unsigned char           skc_reuse;      /* SO_REUSEADDR */
    int                     skc_bound_dev_if;

    union {
        struct hlist_node   skc_bind_node;
        struct hlist_node   skc_portaddr_node;
    };

    struct proto            *skc_prot;
    struct net              *skc_net;       /* Network namespace */

    /* Hash table links */
    union {
        struct hlist_node   skc_node;
        struct hlist_nulls_node skc_nulls_node;
    };

    atomic64_t              skc_cookie;

    /* Local and remote addresses */
    union {
        /* IPv4 */
        struct {
            __be32          skc_daddr;
            __be32          skc_rcv_saddr;
        };
        /* IPv6 */
        struct {
            struct in6_addr skc_v6_daddr;
            struct in6_addr skc_v6_rcv_saddr;
        };
    };

    /* Ports */
    __be16                  skc_dport;
    __u16                   skc_num;        /* Local port */
};
```

### Protocol Operations

The proto_ops structure defines the operations table for a socket protocol
family. It contains function pointers that implement the BSD socket API
operations such as bind(), connect(), accept(), listen(), and
sendmsg()/recvmsg(). Each address family (AF_INET, AF_UNIX, etc.) provides its
own proto_ops implementation, allowing the socket layer to dispatch operations
to the appropriate protocol handler. The operations are invoked by the kernel's
socket system calls and handle the protocol-specific details of each operation.

```c
/* include/linux/net.h */
struct proto_ops {
    int             family;
    struct module   *owner;

    int (*release)(struct socket *sock);
    int (*bind)(struct socket *sock, struct sockaddr *addr, int addrlen);
    int (*connect)(struct socket *sock, struct sockaddr *addr,
                   int addrlen, int flags);
    int (*socketpair)(struct socket *sock1, struct socket *sock2);
    int (*accept)(struct socket *sock, struct socket *newsock, int flags);
    int (*getname)(struct socket *sock, struct sockaddr *addr, int peer);
    __poll_t (*poll)(struct file *file, struct socket *sock,
                     struct poll_table_struct *wait);
    int (*ioctl)(struct socket *sock, unsigned int cmd, unsigned long arg);
    int (*listen)(struct socket *sock, int backlog);
    int (*shutdown)(struct socket *sock, int how);
    int (*setsockopt)(struct socket *sock, int level, int optname,
                      sockptr_t optval, unsigned int optlen);
    int (*getsockopt)(struct socket *sock, int level, int optname,
                      char __user *optval, int __user *optlen);
    int (*sendmsg)(struct socket *sock, struct msghdr *msg, size_t len);
    int (*recvmsg)(struct socket *sock, struct msghdr *msg, size_t len,
                   int flags);
    int (*mmap)(struct file *file, struct socket *sock,
                struct vm_area_struct *vma);
    ssize_t (*sendpage)(struct socket *sock, struct page *page, int offset,
                        size_t size, int flags);
};
```

### Socket System Call Flow

```
┌─────────────────────────────────────────────────────────────┐
│                  socket() System Call                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  User: socket(AF_INET, SOCK_STREAM, 0)                      │
│           │                                                 │
│           ▼                                                 │
│  1. __sys_socket()                                          │
│           │                                                 │
│           ▼                                                 │
│  2. sock_create()                                           │
│     - Allocate struct socket                                │
│     - Find protocol family (AF_INET)                        │
│           │                                                 │
│           ▼                                                 │
│  3. pf->create() → inet_create()                            │
│     - Find protocol (TCP for SOCK_STREAM)                   │
│     - Allocate struct sock (sk_alloc)                       │
│     - Initialize TCP-specific (tcp_v4_init_sock)            │
│           │                                                 │
│           ▼                                                 │
│  4. sock_map_fd()                                           │
│     - Create file descriptor                                │
│     - Associate with socket                                 │
│           │                                                 │
│           ▼                                                 │
│  5. Return fd to userspace                                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## sk_buff (Socket Buffer)

The sk_buff is the fundamental data structure for packet handling in Linux. It
represents a network packet and its metadata throughout the stack.

### Core Data Structure

The sk_buff structure is the central data structure for network packet handling
in the Linux kernel. It represents a single network packet as it traverses the
entire networking stack, from device driver through protocol layers to user
space. The structure contains metadata about the packet including pointers to
the various protocol headers (MAC, IP, transport), the associated socket and
network device, timestamps, and checksum information. Key buffer pointers
(head, data, tail, end) define the packet data boundaries and allow efficient
header manipulation through push/pull operations. The sk_buff can be cloned to
allow multiple references to the same packet data, with the users field
tracking the reference count.

```c
/* include/linux/skbuff.h */
struct sk_buff {
    /* These two must be first */
    union {
        struct {
            struct sk_buff      *next;
            struct sk_buff      *prev;
        };
        struct rb_node          rbnode;
    };

    union {
        struct sock             *sk;
        int                     ip_defrag_offset;
    };

    union {
        ktime_t                 tstamp;
        u64                     skb_mstamp_ns;
    };

    /* Reference to network device */
    struct net_device           *dev;

    /* Packet type */
    __u8                        pkt_type:3;    /* PACKET_HOST, etc. */
    __u8                        ignore_df:1;
    __u8                        dst_pending_confirm:1;
    __u8                        ip_summed:2;   /* Checksum status */
    __u8                        ooo_okay:1;

    /* Protocol information */
    __u8                        l4_hash:1;
    __u8                        sw_hash:1;
    __u8                        wifi_acked_valid:1;
    __u8                        wifi_acked:1;
    __u8                        no_fcs:1;
    __u8                        encapsulation:1;
    __u8                        encap_hdr_csum:1;
    __u8                        csum_valid:1;

    __be16                      protocol;      /* Ethernet protocol */
    __u16                       transport_header;
    __u16                       network_header;
    __u16                       mac_header;

    /* Buffer pointers */
    sk_buff_data_t              tail;
    sk_buff_data_t              end;
    unsigned char               *head;
    unsigned char               *data;

    /* Buffer length */
    unsigned int                len;           /* Total length */
    unsigned int                data_len;      /* Paged data length */
    __u16                       mac_len;
    __u16                       hdr_len;

    /* Cloned skb reference count */
    refcount_t                  users;

    /* ... many more fields ... */
};
```

### sk_buff Memory Layout

```
┌─────────────────────────────────────────────────────────────┐
│                    sk_buff Memory Layout                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  struct sk_buff (metadata)                                  │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ head, data, tail, end pointers                      │    │
│  │ len, data_len, headers, device, etc.                │    │
│  └─────────────────────────────────────────────────────┘    │
│              │                                              │
│              ▼                                              │
│  Data buffer (kmalloc'd or from page pool)                  │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ head ──────────────────────────────────────── end   │    │
│  │   │                                             │   │    │
│  │   ▼                                             │   │    │
│  │ ┌──────┬──────┬──────┬──────┬────────┬────────┐ │   │    │
│  │ │head- │ MAC  │ IP   │ TCP  │ Data   │ tail-  │ │   │    │
│  │ │room  │header│header│header│ payload│ room   │ │   │    │
│  │ └──────┴──────┴──────┴──────┴────────┴────────┘ │   │    │
│  │        ▲      ▲      ▲      ▲        ▲          │   │    │
│  │        │      │      │      │        │          │   │    │
│  │      data   mac_   net_  trans-    tail         │   │    │
│  │             header header port_                 │   │    │
│  │                          header                 │   │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
│  Headroom: Space for prepending headers                     │
│  Tailroom: Space for appending trailers                     │
│                                                             │
│  Paged data (skb_shared_info):                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ struct skb_shared_info                              │    │
│  │   nr_frags, frags[], frag_list                      │    │
│  │   Points to additional pages for large packets      │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### sk_buff Operations

```c
/* Allocation */
struct sk_buff *skb = alloc_skb(size, GFP_KERNEL);
struct sk_buff *skb = netdev_alloc_skb(dev, size);  /* For device drivers */
struct sk_buff *skb = napi_alloc_skb(napi, size);   /* In NAPI context */

/* Free */
kfree_skb(skb);              /* Free if refcount drops to 0 */
consume_skb(skb);            /* Free, indicates successful consumption */
dev_kfree_skb_any(skb);      /* For device drivers, any context */

/* Reserve headroom */
skb_reserve(skb, NET_SKB_PAD);  /* Reserve space before pushing headers */

/* Add data */
void *ptr = skb_put(skb, len);       /* Add data at tail, advance tail */
void *ptr = skb_push(skb, len);      /* Add data at head, back up data */

/* Remove data */
void *ptr = skb_pull(skb, len);      /* Remove from head, advance data */
skb_trim(skb, len);                  /* Trim to length, adjust tail */

/* Header pointers */
skb_reset_mac_header(skb);
skb_set_network_header(skb, offset);
skb_set_transport_header(skb, offset);

/* Get header pointers */
struct ethhdr *eth = eth_hdr(skb);
struct iphdr *iph = ip_hdr(skb);
struct tcphdr *th = tcp_hdr(skb);
struct udphdr *uh = udp_hdr(skb);

/* Cloning (share data, copy metadata) */
struct sk_buff *clone = skb_clone(skb, GFP_ATOMIC);

/* Copy (complete copy) */
struct sk_buff *copy = skb_copy(skb, GFP_ATOMIC);

/* Check available space */
unsigned int headroom = skb_headroom(skb);
unsigned int tailroom = skb_tailroom(skb);

/* Expand if needed */
skb = skb_realloc_headroom(skb, new_headroom);
pskb_expand_head(skb, nhead, ntail, GFP_ATOMIC);
```

### skb_shared_info

The skb_shared_info structure is located at the end of the sk_buff's linear
data buffer and contains information shared between cloned sk_buffs. It manages
fragmented packet data through the frags array, which holds references to page
fragments for large packets that exceed the linear buffer size. The structure
also contains Generic Segmentation Offload (GSO) parameters that enable the
kernel to create large "super packets" that are segmented by hardware or
software. The dataref field tracks how many sk_buffs share this data, enabling
copy-on-write semantics when modifying shared packet data.

```c
/* Located at end of linear data buffer */
struct skb_shared_info {
    __u8            nr_frags;           /* Number of page fragments */
    __u8            tx_flags;
    unsigned short  gso_size;           /* Generic segmentation offload */
    unsigned short  gso_segs;
    unsigned short  gso_type;

    struct sk_buff  *frag_list;         /* List of additional skbs */
    struct skb_shared_hwtstamps hwtstamps;

    u32             tskey;
    atomic_t        dataref;            /* Data reference count */

    skb_frag_t      frags[MAX_SKB_FRAGS];  /* Page fragments */
};

/* Get shared info */
struct skb_shared_info *shinfo = skb_shinfo(skb);
```

---

## Network Device Layer

The network device layer abstracts hardware NICs and provides a uniform
interface for the protocol stack.

### struct net_device

The net_device structure represents a network interface in the Linux kernel,
whether physical (Ethernet NICs, wireless adapters) or virtual (bridges, VLANs,
tunnels). It contains all the information needed to manage network devices
including the interface name, hardware addresses, MTU, and operational state
flags. The netdev_ops pointer references the driver's callback functions for
transmitting packets, configuring the device, and handling hardware features.
For multi-queue NICs, the structure manages arrays of TX and RX queues, and the
napi_list links to NAPI instances for interrupt mitigation. Each net_device
also belongs to a network namespace, enabling network isolation for containers.

```c
/* include/linux/netdevice.h */
struct net_device {
    /* Device name and identifiers */
    char                    name[IFNAMSIZ];
    int                     ifindex;

    /* Hardware addresses */
    unsigned char           dev_addr[MAX_ADDR_LEN];
    unsigned char           broadcast[MAX_ADDR_LEN];

    /* Device characteristics */
    unsigned int            mtu;
    unsigned short          type;           /* ARPHRD_ETHER, etc. */
    unsigned short          hard_header_len;
    unsigned char           min_header_len;

    /* State and flags */
    unsigned int            flags;          /* IFF_UP, IFF_BROADCAST, etc. */
    unsigned int            priv_flags;
    unsigned short          gflags;
    unsigned int            operstate;      /* RFC 2863 operstate */

    /* Network operations */
    const struct net_device_ops *netdev_ops;
    const struct ethtool_ops *ethtool_ops;

    /* Header operations */
    const struct header_ops *header_ops;

    /* Hardware features */
    netdev_features_t       features;
    netdev_features_t       hw_features;
    netdev_features_t       wanted_features;

    /* Queues */
    unsigned int            num_tx_queues;
    unsigned int            num_rx_queues;
    struct netdev_queue     *_tx;           /* TX queues */
    struct netdev_rx_queue  *_rx;           /* RX queues */

    /* NAPI */
    struct list_head        napi_list;

    /* Statistics */
    struct net_device_stats stats;
    struct pcpu_sw_netstats __percpu *tstats;

    /* Reference counting */
    refcount_t              dev_refcnt;

    /* Network namespace */
    possible_net_t          nd_net;

    /* Driver private data */
    void                    *priv;

    /* ... many more fields ... */
};
```

### net_device_ops

The net_device_ops structure contains the callback functions that a network
device driver must implement to integrate with the kernel networking stack. The
most critical callback is ndo_start_xmit(), which the kernel calls to transmit
packets through the device. Other callbacks handle device lifecycle
(open/stop), address configuration, receive mode settings for
multicast/promiscuous operation, and statistics retrieval. Modern callbacks
include ndo_bpf() and ndo_xdp_xmit() for XDP (eXpress Data Path) support,
enabling high-performance packet processing at the driver level.

```c
/* include/linux/netdevice.h */
struct net_device_ops {
    int  (*ndo_init)(struct net_device *dev);
    void (*ndo_uninit)(struct net_device *dev);
    int  (*ndo_open)(struct net_device *dev);
    int  (*ndo_stop)(struct net_device *dev);

    /* Transmit */
    netdev_tx_t (*ndo_start_xmit)(struct sk_buff *skb,
                                   struct net_device *dev);

    /* Receive mode */
    void (*ndo_set_rx_mode)(struct net_device *dev);

    /* Address handling */
    int  (*ndo_set_mac_address)(struct net_device *dev, void *addr);
    int  (*ndo_validate_addr)(struct net_device *dev);

    /* Statistics */
    void (*ndo_get_stats64)(struct net_device *dev,
                            struct rtnl_link_stats64 *storage);

    /* VLAN */
    int  (*ndo_vlan_rx_add_vid)(struct net_device *dev,
                                 __be16 proto, u16 vid);
    int  (*ndo_vlan_rx_kill_vid)(struct net_device *dev,
                                  __be16 proto, u16 vid);

    /* XDP */
    int  (*ndo_bpf)(struct net_device *dev, struct netdev_bpf *bpf);
    int  (*ndo_xdp_xmit)(struct net_device *dev, int n,
                         struct xdp_frame **xdp, u32 flags);

    /* ... many more callbacks ... */
};
```

### Transmit Path

```
┌─────────────────────────────────────────────────────────────┐
│                    Transmit Path                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Application                                                │
│      │ send()/write()                                       │
│      ▼                                                      │
│  Socket Layer                                               │
│      │ tcp_sendmsg() / udp_sendmsg()                        │
│      ▼                                                      │
│  Transport Layer (TCP/UDP)                                  │
│      │ Build TCP/UDP header                                 │
│      │ tcp_transmit_skb()                                   │
│      ▼                                                      │
│  IP Layer                                                   │
│      │ ip_queue_xmit() / ip_local_out()                     │
│      │ Route lookup, build IP header                        │
│      │ Netfilter: NF_INET_LOCAL_OUT                         │
│      ▼                                                      │
│  Neighboring/ARP                                            │
│      │ Resolve MAC address                                  │
│      │ neigh_output()                                       │
│      ▼                                                      │
│  Device Layer                                               │
│      │ dev_queue_xmit()                                     │
│      │ Traffic control (qdisc)                              │
│      ▼                                                      │
│  Driver                                                     │
│      │ ndo_start_xmit()                                     │
│      │ DMA to hardware                                      │
│      ▼                                                      │
│  Hardware NIC                                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Receive Path

```
┌─────────────────────────────────────────────────────────────┐
│                    Receive Path                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Hardware NIC                                               │
│      │ Packet arrives, DMA to ring buffer                   │
│      │ Raise interrupt                                      │
│      ▼                                                      │
│  IRQ Handler                                                │
│      │ Acknowledge interrupt                                │
│      │ Schedule NAPI poll                                   │
│      ▼                                                      │
│  NAPI Poll (softirq context)                                │
│      │ napi_poll() → driver poll function                   │
│      │ Allocate skb, copy/map data                          │
│      │ napi_gro_receive() or netif_receive_skb()            │
│      ▼                                                      │
│  GRO (Generic Receive Offload)                              │
│      │ Coalesce related packets                             │
│      ▼                                                      │
│  netif_receive_skb()                                        │
│      │ XDP (if attached, runs first)                        │
│      │ Packet type demux                                    │
│      ▼                                                      │
│  IP Layer                                                   │
│      │ ip_rcv()                                             │
│      │ Netfilter: NF_INET_PRE_ROUTING                       │
│      │ Route lookup                                         │
│      │ NF_INET_LOCAL_IN (for local delivery)                │
│      ▼                                                      │
│  Transport Layer                                            │
│      │ tcp_v4_rcv() / udp_rcv()                             │
│      │ Find socket, queue to receive buffer                 │
│      ▼                                                      │
│  Socket Layer                                               │
│      │ sk_data_ready() callback                             │
│      │ Wake up waiting processes                            │
│      ▼                                                      │
│  Application                                                │
│      recv()/read()                                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## NAPI (New API)

NAPI is the standard mechanism for high-performance packet processing in Linux.
It combines interrupt-driven notification with polling-based processing.

### Why NAPI?

```
┌─────────────────────────────────────────────────────────────┐
│              Problem: Interrupt Overhead                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Traditional interrupt-per-packet:                          │
│                                                             │
│  ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐                         │
│  │PKT1│ │PKT2│ │PKT3│ │PKT4│ │PKT5│  → 5 packets            │
│  └────┘ └────┘ └────┘ └────┘ └────┘                         │
│     │      │      │      │      │                           │
│     ▼      ▼      ▼      ▼      ▼                           │
│  [IRQ]  [IRQ]  [IRQ]  [IRQ]  [IRQ]   → 5 interrupts!        │
│                                                             │
│  At high packet rates:                                      │
│  - CPU drowns in interrupt handling                         │
│  - "Interrupt livelock"                                     │
│  - Actual work never gets done                              │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│              Solution: NAPI Polling                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐                         │
│  │PKT1│ │PKT2│ │PKT3│ │PKT4│ │PKT5│  → 5 packets            │
│  └────┘ └────┘ └────┘ └────┘ └────┘                         │
│     │                                                       │
│     ▼                                                       │
│  [IRQ] → Disable interrupts → [Poll loop processes all]     │
│                                    │                        │
│                                    ▼                        │
│                               [Re-enable when done]         │
│                                                             │
│  Only 1 interrupt for 5 packets!                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### NAPI Data Structures

The napi_struct is the core data structure for NAPI (New API) polling, which
provides interrupt mitigation for high-speed network packet processing. Each
napi_struct is associated with a network device and contains a poll function
pointer that the kernel calls in softirq context to process received packets.
The weight field limits how many packets can be processed in a single poll
invocation, preventing any single device from monopolizing the CPU. The
structure also maintains state flags for scheduling, GRO (Generic Receive
Offload) hash tables for packet coalescing, and optionally a dedicated kernel
thread for threaded NAPI mode.

```c
/* include/linux/netdevice.h */
struct napi_struct {
    struct list_head        poll_list;      /* Link in poll list */

    unsigned long           state;          /* NAPI_STATE_* flags */
    int                     weight;         /* Max work per poll */
    int                     defer_hard_irqs_count;

    unsigned long           gro_bitmask;
    int (*poll)(struct napi_struct *napi, int budget);

    int                     poll_owner;
    int                     list_owner;
    struct net_device       *dev;
    struct gro_list         gro_hash[GRO_HASH_BUCKETS];

    struct sk_buff          *skb;           /* Current GRO skb */
    struct list_head        rx_list;        /* For bulk flushing */
    int                     rx_count;

    struct hrtimer          timer;          /* Busy poll timer */
    struct task_struct      *thread;        /* For threaded NAPI */
};

/* NAPI states */
enum {
    NAPI_STATE_SCHED,           /* Poll is scheduled */
    NAPI_STATE_MISSED,          /* Rescheduled while polling */
    NAPI_STATE_DISABLE,         /* Disable pending */
    NAPI_STATE_NPSVC,           /* Netpoll in use */
    NAPI_STATE_LISTED,          /* In NAPI list */
    NAPI_STATE_NO_BUSY_POLL,    /* Busy poll disabled */
    NAPI_STATE_IN_BUSY_POLL,    /* Currently busy polling */
    NAPI_STATE_PREFER_BUSY_POLL,
    NAPI_STATE_THREADED,        /* Threaded NAPI enabled */
    NAPI_STATE_SCHED_THREADED,  /* Threaded poll scheduled */
};
```

### NAPI Driver Implementation

```c
/* Driver initialization */
static int my_driver_probe(struct pci_dev *pdev)
{
    struct net_device *dev;
    struct my_priv *priv;

    dev = alloc_etherdev(sizeof(*priv));
    priv = netdev_priv(dev);

    /* Initialize NAPI */
    netif_napi_add(dev, &priv->napi, my_poll, NAPI_POLL_WEIGHT);

    return register_netdev(dev);
}

/* Interrupt handler */
static irqreturn_t my_irq_handler(int irq, void *dev_id)
{
    struct net_device *dev = dev_id;
    struct my_priv *priv = netdev_priv(dev);

    /* Disable further interrupts */
    my_disable_irq(priv);

    /* Schedule NAPI poll */
    if (napi_schedule_prep(&priv->napi))
        __napi_schedule(&priv->napi);

    return IRQ_HANDLED;
}

/* NAPI poll function */
static int my_poll(struct napi_struct *napi, int budget)
{
    struct my_priv *priv = container_of(napi, struct my_priv, napi);
    int work_done = 0;

    while (work_done < budget) {
        struct sk_buff *skb;

        /* Check for packet in ring buffer */
        if (!my_rx_pending(priv))
            break;

        /* Allocate and fill skb */
        skb = napi_alloc_skb(napi, RX_BUF_SIZE);
        if (!skb)
            break;

        my_rx_copy_data(priv, skb);

        /* Set protocol and deliver */
        skb->protocol = eth_type_trans(skb, priv->dev);
        napi_gro_receive(napi, skb);

        work_done++;
    }

    /* If we processed everything, re-enable interrupts */
    if (work_done < budget) {
        napi_complete_done(napi, work_done);
        my_enable_irq(priv);
    }

    return work_done;
}
```

### GRO (Generic Receive Offload)

```
┌─────────────────────────────────────────────────────────────┐
│                  GRO Aggregation                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Without GRO:                                               │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐                │
│  │1500 B  │ │1500 B  │ │1500 B  │ │1500 B  │                │
│  │TCP seg │ │TCP seg │ │TCP seg │ │TCP seg │                │
│  └────────┘ └────────┘ └────────┘ └────────┘                │
│      │          │          │          │                     │
│      ▼          ▼          ▼          ▼                     │
│  4 trips through the entire network stack                   │
│                                                             │
│  With GRO:                                                  │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐                │
│  │1500 B  │ │1500 B  │ │1500 B  │ │1500 B  │                │
│  │TCP seg │ │TCP seg │ │TCP seg │ │TCP seg │                │
│  └────────┘ └────────┘ └────────┘ └────────┘                │
│      │          │          │          │                     │
│      └──────────┴──────────┴──────────┘                     │
│                     │                                       │
│                     ▼                                       │
│              ┌──────────────────┐                           │
│              │   6000 B merged  │                           │
│              │   "super packet" │                           │
│              └──────────────────┘                           │
│                     │                                       │
│                     ▼                                       │
│  1 trip through the network stack                           │
│                                                             │
│  GRO merges packets with:                                   │
│  - Same flow (5-tuple)                                      │
│  - Sequential sequence numbers                              │
│  - No special flags (SYN, FIN, etc.)                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## TCP/IP Stack

### TCP State Machine

The TCP state enumeration defines the possible states of a TCP connection as
specified by RFC 793. These states track the connection lifecycle from initial
handshake (SYN_SENT, SYN_RECV) through established communication (ESTABLISHED)
to graceful termination (FIN_WAIT, CLOSE_WAIT, TIME_WAIT). The kernel uses
these states to determine valid actions and transitions for each connection,
ensuring proper protocol behavior.

```c
/* include/net/tcp_states.h */
enum {
    TCP_ESTABLISHED = 1,
    TCP_SYN_SENT,
    TCP_SYN_RECV,
    TCP_FIN_WAIT1,
    TCP_FIN_WAIT2,
    TCP_TIME_WAIT,
    TCP_CLOSE,
    TCP_CLOSE_WAIT,
    TCP_LAST_ACK,
    TCP_LISTEN,
    TCP_CLOSING,
    TCP_NEW_SYN_RECV,
    TCP_MAX_STATES
};
```

```
┌─────────────────────────────────────────────────────────────┐
│                   TCP State Machine                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│                        ┌────────┐                           │
│                        │ CLOSED │                           │
│                        └───┬────┘                           │
│            ┌───────────────┴───────────────┐                │
│            │ (passive open)    (active open) │              │
│            ▼                               ▼                │
│      ┌──────────┐                   ┌───────────┐           │
│      │  LISTEN  │                   │ SYN_SENT  │           │
│      └────┬─────┘                   └─────┬─────┘           │
│           │ rcv SYN                       │ rcv SYN,ACK     │
│           │ send SYN,ACK                  │ send ACK        │
│           ▼                               ▼                 │
│      ┌───────────┐                  ┌─────────────┐         │
│      │ SYN_RECV  │─────────────────▶│ ESTABLISHED │         │
│      └───────────┘ rcv ACK          └──────┬──────┘         │
│                                            │                │
│           ┌──────────────┬─────────────────┤                │
│           │ close        │ rcv FIN         │                │
│           ▼              ▼                 ▼                │
│     ┌───────────┐  ┌────────────┐   ┌────────────┐          │
│     │ FIN_WAIT1 │  │ CLOSE_WAIT │   │ (continue) │          │
│     └─────┬─────┘  └──────┬─────┘   └────────────┘          │
│           │               │                                 │
│           │ rcv ACK       │ close                           │
│           ▼               ▼                                 │
│     ┌───────────┐  ┌───────────┐                            │
│     │ FIN_WAIT2 │  │ LAST_ACK  │                            │
│     └─────┬─────┘  └─────┬─────┘                            │
│           │              │ rcv ACK                          │
│           │ rcv FIN      ▼                                  │
│           ▼         ┌────────┐                              │
│     ┌───────────┐   │ CLOSED │                              │
│     │ TIME_WAIT │   └────────┘                              │
│     └─────┬─────┘                                           │
│           │ 2MSL timeout                                    │
│           ▼                                                 │
│      ┌────────┐                                             │
│      │ CLOSED │                                             │
│      └────────┘                                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### TCP Socket Structure

The tcp_sock structure extends inet_connection_sock with TCP-specific state and
is the kernel's complete representation of a TCP connection. It contains the
essential TCP protocol state including sequence numbers (snd_nxt, snd_una,
rcv_nxt) that track transmitted and received data, the congestion window
(snd_cwnd) and slow start threshold (snd_ssthresh) for congestion control, and
RTT measurements (srtt_us, rttvar_us) for retransmission timeout calculation.
The structure also maintains SACK (Selective Acknowledgment) information for
efficient loss recovery. This is the structure that the TCP implementation
manipulates as packets flow through the stack.

```c
/* include/linux/tcp.h */
struct tcp_sock {
    struct inet_connection_sock inet_conn;

    /* Sequence numbers */
    u32     rcv_nxt;        /* Next sequence expected */
    u32     copied_seq;     /* Last seq copied to user */
    u32     rcv_wup;        /* rcv_nxt on last window update */
    u32     snd_nxt;        /* Next sequence to send */
    u32     snd_una;        /* First unacknowledged seq */
    u32     snd_sml;        /* Last byte of small packet sent */

    /* Receive window */
    u32     rcv_wnd;        /* Current receive window */
    u32     write_seq;      /* Tail of data held in send buffer */

    /* Congestion control */
    u32     snd_cwnd;       /* Congestion window */
    u32     snd_ssthresh;   /* Slow start threshold */

    /* RTT measurement */
    u32     srtt_us;        /* Smoothed RTT << 3 in usecs */
    u32     mdev_us;        /* Medium deviation */
    u32     rttvar_us;      /* Smoothed mean deviation */

    /* Retransmission */
    u32     rto;            /* Retransmission timeout */
    u8      retransmits;    /* Number of retransmits */

    /* Timestamps */
    u32     tsoffset;
    u32     rcv_tsval;
    u32     rcv_tsecr;

    /* SACK */
    struct tcp_sack_block selective_acks[4];

    /* ... many more fields ... */
};
```

### IP Layer Functions

```c
/* Receive path */
int ip_rcv(struct sk_buff *skb, struct net_device *dev,
           struct packet_type *pt, struct net_device *orig_dev);
int ip_local_deliver(struct sk_buff *skb);

/* Transmit path */
int ip_queue_xmit(struct sock *sk, struct sk_buff *skb, struct flowi *fl);
int ip_local_out(struct net *net, struct sock *sk, struct sk_buff *skb);
int ip_output(struct net *net, struct sock *sk, struct sk_buff *skb);

/* Routing */
struct rtable *ip_route_output_flow(struct net *net, struct flowi4 *flp4,
                                     const struct sock *sk);
int fib_lookup(struct net *net, struct flowi4 *flp,
               struct fib_result *res, unsigned int flags);

/* Fragmentation */
int ip_fragment(struct net *net, struct sock *sk, struct sk_buff *skb,
                unsigned int mtu,
                int (*output)(struct net *, struct sock *, struct sk_buff *));
```

---

## Netfilter and iptables

Netfilter provides the framework for packet filtering, NAT, and connection
tracking.

### Netfilter Hooks

```
┌─────────────────────────────────────────────────────────────┐
│                   Netfilter Hooks                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Incoming packet                                            │
│       │                                                     │
│       ▼                                                     │
│  ┌───────────────────────┐                                  │
│  │   NF_INET_PRE_ROUTING │ ← PREROUTING chain               │
│  └───────────┬───────────┘                                  │
│              │                                              │
│              ▼                                              │
│      ┌─────────────┐                                        │
│      │   Routing   │                                        │
│      │  Decision   │                                        │
│      └──────┬──────┘                                        │
│             │                                               │
│     ┌───────┴───────┐                                       │
│     │               │                                       │
│     ▼               ▼                                       │
│  (local)         (forward)                                  │
│     │               │                                       │
│     ▼               ▼                                       │
│ ┌──────────────┐ ┌──────────────┐                           │
│ │NF_INET_LOCAL_│ │ NF_INET_     │                           │
│ │    INPUT     │ │   FORWARD    │ ← INPUT/FORWARD chains    │
│ └──────┬───────┘ └──────┬───────┘                           │
│        │                │                                   │
│        ▼                │                                   │
│   Local process         │                                   │
│        │                │                                   │
│        ▼                │                                   │
│ ┌──────────────┐        │                                   │
│ │NF_INET_LOCAL_│        │                                   │
│ │    OUTPUT    │        │ ← OUTPUT chain                    │
│ └──────┬───────┘        │                                   │
│        │                │                                   │
│        └────────┬───────┘                                   │
│                 │                                           │
│                 ▼                                           │
│        ┌────────────────┐                                   │
│        │    Routing     │                                   │
│        │    Decision    │                                   │
│        └───────┬────────┘                                   │
│                │                                            │
│                ▼                                            │
│       ┌─────────────────────┐                               │
│       │ NF_INET_POST_ROUTING│ ← POSTROUTING chain           │
│       └─────────┬───────────┘                               │
│                 │                                           │
│                 ▼                                           │
│         Outgoing packet                                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Hook Registration

The nf_hook_ops structure defines a netfilter hook registration, allowing
kernel modules to intercept packets at specific points in the network stack.
Each registration specifies a hook function that is called when packets
traverse the corresponding hook point (PRE_ROUTING, LOCAL_IN, FORWARD,
LOCAL_OUT, POST_ROUTING). The priority field determines the order in which
multiple hooks at the same point are called, enabling layered filtering such as
connection tracking before NAT before packet filtering. Hook functions return
verdicts like NF_ACCEPT, NF_DROP, or NF_QUEUE to control packet fate.

```c
/* include/linux/netfilter.h */
struct nf_hook_ops {
    nf_hookfn           *hook;          /* Hook function */
    struct net_device   *dev;
    void                *priv;
    u_int8_t            pf;             /* Protocol family */
    unsigned int        hooknum;        /* Hook number */
    int                 priority;       /* Hook priority */
};

/* Hook function signature */
typedef unsigned int nf_hookfn(void *priv,
                                struct sk_buff *skb,
                                const struct nf_hook_state *state);

/* Return values */
#define NF_DROP         0   /* Drop packet */
#define NF_ACCEPT       1   /* Continue processing */
#define NF_STOLEN       2   /* Packet stolen, don't free */
#define NF_QUEUE        3   /* Queue to userspace */
#define NF_REPEAT       4   /* Call this hook again */
#define NF_STOP         5   /* Stop processing */

/* Registration */
int nf_register_net_hook(struct net *net, const struct nf_hook_ops *ops);
void nf_unregister_net_hook(struct net *net, const struct nf_hook_ops *ops);
```

### Connection Tracking

The nf_conn structure represents a tracked network connection in the netfilter
connection tracking (conntrack) system. It stores the connection tuple
(source/destination IP and port) in both directions via tuplehash, allowing
packets in either direction to be associated with the same connection entry.
The status field tracks the connection state (NEW, ESTABLISHED, RELATED), while
the timeout field determines when inactive connections are garbage collected.
Connection tracking enables stateful firewall rules, NAT, and protocol helpers
that understand application-layer protocols. The structure includes
protocol-specific extensions for TCP, UDP, ICMP, and other protocols.

```c
/* include/net/netfilter/nf_conntrack.h */
struct nf_conn {
    /* Hash table entry */
    struct nf_conntrack_tuple_hash tuplehash[IP_CT_DIR_MAX];

    /* Usage count */
    refcount_t ct_general;

    /* Connection status */
    unsigned long status;

    /* Timeout */
    u32 timeout;

    /* Protocol-specific data */
    union nf_conntrack_proto proto;

    /* NAT info */
    struct nf_nat_ext *nat_ext;

    /* ... more fields ... */
};

/* Connection states */
enum ip_conntrack_info {
    IP_CT_ESTABLISHED,
    IP_CT_RELATED,
    IP_CT_NEW,
    IP_CT_IS_REPLY,
    IP_CT_ESTABLISHED_REPLY,
    IP_CT_RELATED_REPLY,
    IP_CT_NUMBER,
};
```

---

## XDP (eXpress Data Path)

XDP allows BPF programs to run at the earliest possible point in the receive
path, before sk_buff allocation.

### XDP Position in Stack

```
┌─────────────────────────────────────────────────────────────┐
│                   XDP Position                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Hardware NIC                                               │
│       │                                                     │
│       │ Packet arrives in DMA ring buffer                   │
│       │                                                     │
│       ▼                                                     │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                 XDP Hook                            │    │
│  │  - Runs BEFORE sk_buff allocation                   │    │
│  │  - Direct access to packet data                     │    │
│  │  - Decisions: PASS, DROP, TX, REDIRECT, ABORTED     │    │
│  └──────────────────────┬──────────────────────────────┘    │
│                         │                                   │
│           ┌─────────────┼─────────────┬──────────┐          │
│           │             │             │          │          │
│           ▼             ▼             ▼          ▼          │
│       XDP_DROP     XDP_PASS     XDP_TX    XDP_REDIRECT      │
│           │             │             │          │          │
│           ▼             ▼             ▼          ▼          │
│       (dropped)    sk_buff        Same NIC   Other device   │
│                    allocation     (bounce)   (redirect)     │
│                         │                                   │
│                         ▼                                   │
│                   Normal stack                              │
│                   processing                                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### XDP Modes

```
┌─────────────────────────────────────────────────────────────┐
│                    XDP Modes                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Native XDP (XDP_FLAGS_DRV_MODE)                         │
│     - Driver implements XDP support                         │
│     - Highest performance                                   │
│     - Runs in driver's NAPI context                         │
│                                                             │
│  2. Generic XDP (XDP_FLAGS_SKB_MODE)                        │
│     - Works with any NIC                                    │
│     - Runs after sk_buff allocation                         │
│     - Lower performance but universal                       │
│                                                             │
│  3. Offloaded XDP (XDP_FLAGS_HW_MODE)                       │
│     - BPF runs on NIC hardware                              │
│     - Highest performance, limited programs                 │
│     - Requires SmartNIC support                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### XDP Program Structure

The xdp_buff structure provides the execution context for XDP (eXpress Data
Path) programs, enabling packet processing before sk_buff allocation. It
contains pointers that define the packet boundaries: data points to the start
of packet data, data_end marks the end, and data_hard_start indicates the
earliest point where headers can be pushed. The data_meta pointer provides
space for passing metadata between XDP programs and the networking stack.
Unlike sk_buff, xdp_buff is a lightweight structure that allows XDP programs to
achieve near-line-rate packet processing. The xdp_action enum defines the
possible verdicts: DROP, PASS to the stack, TX on the same interface, or
REDIRECT to another device.

```c
/* XDP context */
struct xdp_buff {
    void *data;              /* Start of packet data */
    void *data_end;          /* End of packet data */
    void *data_meta;         /* Metadata before packet */
    void *data_hard_start;   /* Earliest point to push */
    struct xdp_rxq_info *rxq;
    struct xdp_txq_info *txq;
    u32 frame_sz;
};

/* XDP return values */
enum xdp_action {
    XDP_ABORTED = 0,    /* Error, trace and drop */
    XDP_DROP,           /* Drop packet silently */
    XDP_PASS,           /* Pass to normal stack */
    XDP_TX,             /* Transmit on same interface */
    XDP_REDIRECT,       /* Redirect to other interface/CPU */
};

/* Example XDP program */
SEC("xdp")
int xdp_filter(struct xdp_md *ctx)
{
    void *data = (void *)(long)ctx->data;
    void *data_end = (void *)(long)ctx->data_end;

    struct ethhdr *eth = data;
    if ((void *)(eth + 1) > data_end)
        return XDP_ABORTED;

    /* Drop non-IP packets */
    if (eth->h_proto != htons(ETH_P_IP))
        return XDP_DROP;

    return XDP_PASS;
}
```

### XDP Use Cases

```
┌─────────────────────────────────────────────────────────────┐
│                   XDP Use Cases                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. DDoS Mitigation                                         │
│     - Drop malicious packets at earliest point              │
│     - Millions of packets/sec filtering                     │
│                                                             │
│  2. Load Balancing                                          │
│     - Redirect packets to different servers                 │
│     - Avoid full stack processing overhead                  │
│     - Facebook's Katran L4 load balancer                    │
│                                                             │
│  3. Monitoring/Statistics                                   │
│     - Count packets per flow                                │
│     - Sample traffic without overhead                       │
│                                                             │
│  4. Forwarding                                              │
│     - Software router at line rate                          │
│     - Container networking                                  │
│                                                             │
│  5. Pre-processing                                          │
│     - Decapsulation (VXLAN, GRE)                            │
│     - Header rewriting                                      │
│     - Metadata extraction                                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Traffic Control (tc)

The traffic control subsystem manages packet scheduling, shaping, and classification.

### Qdisc Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                  Traffic Control Overview                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Ingress (receive):                                         │
│  ┌─────────┐                                                │
│  │ ingress │ ─── classifier ─── actions (drop, mirror, etc) │
│  │  qdisc  │                                                │
│  └─────────┘                                                │
│                                                             │
│  Egress (transmit):                                         │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                    Root Qdisc                        │   │
│  │  ┌────────────────────────────────────────────────┐  │   │
│  │  │              Classifier                        │  │   │
│  │  │  (filters: u32, flower, bpf, etc.)             │  │   │
│  │  └───────┬────────────────┬────────────┬──────────┘  │   │
│  │          │                │            │             │   │
│  │          ▼                ▼            ▼             │   │
│  │     ┌─────────┐     ┌─────────┐   ┌─────────┐        │   │
│  │     │ Class 1 │     │ Class 2 │   │ Class 3 │        │   │
│  │     │ (qdisc) │     │ (qdisc) │   │ (qdisc) │        │   │
│  │     └────┬────┘     └────┬────┘   └────┬────┘        │   │
│  │          │               │             │             │   │
│  │          ▼               ▼             ▼             │   │
│  │        Leaf            Leaf          Leaf            │   │
│  │       qdiscs          qdiscs        qdiscs           │   │
│  └──────────────────────────────────────────────────────┘   │
│                          │                                  │
│                          ▼                                  │
│                    Hardware TX Queue                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Common Qdiscs

```
┌─────────────────────────────────────────────────────────────┐
│                    Common Qdiscs                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Classless (simple scheduling):                             │
│  ─────────────────────────────                              │
│  pfifo_fast - Default, 3 priority bands                     │
│  fq         - Fair Queue, per-flow pacing                   │
│  fq_codel   - Fair Queue + CoDel AQM                        │
│  pfifo      - Simple FIFO                                   │
│  tbf        - Token Bucket Filter (rate limiting)           │
│  netem      - Network emulator (delay, loss)                │
│                                                             │
│  Classful (hierarchical):                                   │
│  ────────────────────────                                   │
│  htb        - Hierarchical Token Bucket                     │
│  hfsc       - Hierarchical Fair Service Curve               │
│  prio       - Priority scheduler                            │
│  cbq        - Class-Based Queueing (deprecated)             │
│  drr        - Deficit Round Robin                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### tc-bpf (BPF Classifier/Actions)

The __sk_buff structure is the BPF-accessible representation of a socket
buffer, used by tc (traffic control) BPF programs for packet classification and
manipulation. Unlike struct sk_buff which is internal to the kernel, __sk_buff
exposes a stable ABI to BPF programs with fields like packet length, protocol,
VLAN information, and traffic class. The data and data_end pointers provide
bounds-checked access to packet contents. The tc_classid field allows BPF
programs to direct packets to specific traffic control classes for hierarchical
scheduling. TC BPF programs return action codes like TC_ACT_OK, TC_ACT_SHOT
(drop), or TC_ACT_REDIRECT to control packet processing.

```c
/* tc BPF hook */
struct __sk_buff {
    __u32 len;
    __u32 pkt_type;
    __u32 mark;
    __u32 queue_mapping;
    __u32 protocol;
    __u32 vlan_present;
    __u32 vlan_tci;
    __u32 vlan_proto;
    __u32 priority;
    __u32 ingress_ifindex;
    __u32 ifindex;
    __u32 tc_index;
    __u32 cb[5];
    __u32 hash;
    __u32 tc_classid;
    __u32 data;
    __u32 data_end;
    __u32 napi_id;
    /* ... more fields ... */
};

/* tc return values */
#define TC_ACT_UNSPEC       (-1)
#define TC_ACT_OK           0
#define TC_ACT_RECLASSIFY   1
#define TC_ACT_SHOT         2  /* Drop */
#define TC_ACT_PIPE         3
#define TC_ACT_STOLEN       4
#define TC_ACT_QUEUED       5
#define TC_ACT_REPEAT       6
#define TC_ACT_REDIRECT     7

/* Example tc BPF program */
SEC("tc")
int tc_filter(struct __sk_buff *skb)
{
    void *data = (void *)(long)skb->data;
    void *data_end = (void *)(long)skb->data_end;

    struct ethhdr *eth = data;
    if ((void *)(eth + 1) > data_end)
        return TC_ACT_SHOT;

    /* Set class ID for hierarchical qdisc */
    skb->tc_classid = 0x10010;  /* classid 1:1 */

    return TC_ACT_OK;
}
```

---

## Summary

The Linux networking stack is a sophisticated pipeline optimized for both high
performance and flexibility:

| Layer | Key Structures | Key Functions |
|-------|---------------|---------------|
| Socket | socket, sock | socket(), bind(), connect() |
| Transport | tcp_sock, udp_sock | tcp_v4_rcv(), udp_rcv() |
| Network | iphdr, rtable | ip_rcv(), ip_output() |
| Netfilter | nf_hook_ops, nf_conn | nf_hook_slow() |
| Device | net_device, sk_buff | dev_queue_xmit(), netif_receive_skb() |
| Driver | napi_struct | napi_poll(), ndo_start_xmit() |

Key optimization techniques:
- **NAPI**: Interrupt mitigation through polling
- **GRO/GSO**: Packet coalescing for fewer stack traversals
- **XDP**: Early packet processing before sk_buff allocation
- **Zero-copy**: Direct DMA to/from user buffers (io_uring, AF_XDP)
- **Multi-queue**: Per-CPU TX/RX queues with RSS/XPS
