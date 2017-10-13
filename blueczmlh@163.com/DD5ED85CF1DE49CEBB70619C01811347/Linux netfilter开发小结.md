# <center> Linux netfilter 开发小结 </center>

### 1. 前置知识

##### IP包:

    struct ip {
    #if BYTE_ORDER == LITTLE_ENDIAN
        unsigned char	ip_hl:4,		/* header length */
        unsigned char   ip_v:4;			/* version */
    #endif
        unsigned char	ip_tos;			/* type of service */
        short	ip_len;	/* total length */
        unsigned short	ip_id;/* identification */
        short	ip_off;		
        unsigned char	ip_ttl;/* time to live */
        unsigned char	ip_p;/* protocol */
        unsigned short	ip_sum;
        struct  in_addr ip_src,ip_dst;/* source and dest address */
    };
IHL(Internet Header Length 报头长度)，位于IP报文的第二个字段，4位，表示IP报文头部按32位字长（32位，4字节）计数的长度，也即报文头的长度等于IHL的值乘以4。 (ip_hl)


##### TCP头:

    struct tcphdr {
	    u_short	th_sport;	/* source port */
	    u_short	th_dport;	/* destination port */
	    tcp_seq	th_seq;	/* sequence number */
	    tcp_seq	th_ack;	/* acknowledgement number */
    #if BYTE_ORDER == LITTLE_ENDIAN
	    u_char	th_x2:4,	/* (unused) */
        u_char  th_off:4;	/* data offset */
    #endif
    #if BYTE_ORDER == BIG_ENDIAN
	    u_char	th_off:4,	/* data offset */
        u_char  th_x2:4;	/* (unused) */
    #endif
	    u_char	th_flags;
    #define	TH_FIN	0x01
    #define	TH_SYN	0x02
    #define	TH_RST	0x04
    #define	TH_PUSH	0x08
    #define	TH_ACK	0x10
    #define	TH_URG	0x20
    #define TH_FLAGS (TH_FIN|TH_SYN | TH_RST | TH_ACK | TH_URG)
	    u_short	th_win;	/* window */
	    u_short	th_sum;	/* checksum */
	    u_short	th_urp;	/* urgent pointer */
    };

##### UDP头：

    struct udphdr
    {
	    u_short	uh_sport;       /* source port */
	    u_short	uh_dport;       /* destination port */
	    short	uh_ulen;        /* udp length */
	    u_short	uh_sum;         /* udp checksum */
    };
    
##### ARP:

    typedef struct _ETHERNET_FRAME
    {
        BYTE    DestinationAddress[6];
        BYTE    SourceAddress[6];
        WORD    FrameType;    // in host-order
    } EHTERNET_FRAME, *PETHERNET_FRAME;

    typedef struct _ARP_HEADER
    {
        WORD    HardType;      //硬件类型
        WORD    ProtocolType;  //协议类型
        BYTE    HardLength;    //硬件地址长度
        BYTE    ProtocolLength; //协议地址长度
        WORD    Opcode;        //操作类型
        BYTE    SourceMAC[6];
        BYTE    SourceIP[4];
        BYTE    DestinationMAC[6];
        BYTE    DestinationIP[4];
    } ARP_HEADER, *PARP_HEADER;

    typedef struct _ARP
    {
        EHTERNET_FRAME EthernetFrame;
        ARP_HEADER  ArpHeader;
    }ARP, *PARP;
    
### 2. 基于linux内核的网络防火墙开发

##### Linux核心网络堆栈中有一个全局变量 : 

    struct list_head nf_hooks[NPROTO][NF_MAX_HOOKS];

该变量是一个二维数组，其中第一维用于指定协议族，第二维用于指定hook的类型（即5个HOOK点 ）。注册一个Netfilter hook实际就是在由协议族和hook类型确定的链表中添加一个新的节点。 

##### 规则链:

五个钩子函数（hook functions）,也叫五个规则链。

1. PREROUTING (路由前)
2. INPUT (数据包流入口)
3. FORWARD (转发管卡)
4. OUTPUT(数据包出口)
5. POSTROUTING（路由后)

这是NetFilter规定的五个规则链，任何一个数据包，只要经过本机，必将经过这五个链中的其中一个链。 

##### 5个HOOK点的定义：

1. NF_INET_PRE_ROUTING    在完整性校验之后，选路确定之前
1. NF_INET_LOCAL_IN        在选路确定之后，且数据包的目的是本地主机
1. NF_INET_FORWARD        目的地是其它主机地数据包
1. NF_INET_LOCAL_OUT     来自本机进程的数据包在其离开本地主机的过程中
1. NF_IP_POST_ROUTING    在数据包离开本地主机“上线”之前 
1. NF_INET_POST_ROUTING 同上

##### 对包的处理结果：

1. NF_DROP        丢弃该数据包
1. NF_ACCEPT    保留该数据包
1. NF_STOLEN    忘掉该数据包
1. NF_QUEUE     将该数据包插入到用户空间
1. NF_REPEAT    再次调用该hook函数 

##### 优先级:

    enum nf_ip_hook_priorities {
        NF_IP_PRI_FIRST = INT_MIN,
        NF_IP_PRI_CONNTRACK_DEFRAG = -400,
        NF_IP_PRI_RAW = -300,
        NF_IP_PRI_SELINUX_FIRST = -225,
        NF_IP_PRI_CONNTRACK = -200,
        NF_IP_PRI_MANGLE = -150,
        NF_IP_PRI_NAT_DST = -100,
        NF_IP_PRI_FILTER = 0,
        NF_IP_PRI_SECURITY = 50,
        NF_IP_PRI_NAT_SRC = 100,
        NF_IP_PRI_SELINUX_LAST = 225,
        NF_IP_PRI_CONNTRACK_CONFIRM = INT_MAX,
        NF_IP_PRI_LAST = INT_MAX,
    };

一般写NF_IP_PRI_FIRST 或者 +1 +2...

### 3. 开始编写防火墙

##### 主要代码如下：

    struct nf_hook_ops {
        struct list_head list;

        /* User fills in from here down. */
        nf_hookfn	*hook;
        struct module	*owner;
        void		*priv;
        u_int8_t	pf;
        unsigned int	hooknum;
        /* Hooks are ordered in ascending priority. */
        int		priority;
    };

    // 其中nf_hookfn为函数指针，原型如下
    typedef unsigned int nf_hookfn(const struct nf_hook_ops *ops,
                                    struct sk_buff *skb,
                                    const struct net_device *in,
                                    const struct net_device *out,
                                    int (*okfn)(struct sk_buff *));

    // 注册hook函数步骤如下
    struct nf_hook_ops nfho = {
        .hook = my_hook_func, // 回调函数
        .owner = THIS_MODULE,
        .pf = PF_INET,
        .hooknum = NF_IP_LOCAL_OUT, // 挂载在本地出口
        .priority = NF_IP_PRI_FIRST, //优先级最高
    }

    // 注册nfho到内核
    int __init myhook_init(void)
    {
        return nf_register_hook(&nfho);
    }

    // 注销nfho
    void __exit myhook_exit(void)
    {
        nf_unregister_hook(&nfhook);
    }

    module_init(myhook_init);
    module_exit(myhook_exit);
    
##### sk_buff:

sk_buff结构的成员skb->head指向一个已分配的空间的头部，即申请到的整个缓冲区的头，skb->end指向该空间的尾部，这两个成员指针从空间创建之后，就不能被修改。skb->data指向分配空间中数据的头部，skb->tail指向数据的尾部，这两个值随着网络数据在各层之间的传递、修改，会被不断改动。刚开始接触skb_buf的时候会产生一种错误的认识，就是以为协议头都会是放在skb->head和skb->data这两个指针之间，但实际上skb_buf的操作函数都无法直接对这一段内存进行操作，所有的操作函数所做的就仅仅是修改skb->data和skb->tail这两个指针而已，向套接字缓冲区拷贝数据也是由其它函数来完成的，所以不管是从网卡接受的数据还是上层发下来的数据，协议头都是被放在了skb->data到skb->tail之间，通过skb_push前移skb->data加入协议头，通过skb_pull后移skb->data剥离协议头。

###### sk_buf常用解析

    struct sk_buff *sb = skb;
    
    // IP地址
    #define NIPQUAD(addr) \
            ((unsigned char *)&addr)[0], \
            ((unsigned char *)&addr)[1], \
            ((unsigned char *)&addr)[2], \
            ((unsigned char *)&addr)[3]

    #define NIPQUAD_FMT "%u.%u.%u.%u" 
    __be sip, dip;
    struct iphdr *iph=ip_hdr(sb);
    sip=iph->saddr;
    dip=iph->daddr;

    printk("sip:%u.%u.%u.%u,dip:%u.%u.%u.%u\m",
            NIPQUAD(sip), NIPQUAD(dip));
            
    // 协议, IPPROTO_IP/IPPROTO_TCP等定义在netinet/in.h中，内核空间则定义在linux/in.h中
    struct iphdr *iph=ip_hdr(sb);
    iph->protocol==IPPROTO_TCP;
    
    // 端口
    struct iphdr *iph=ip_hdr(sb);
    struct tcphdr *tcph = NULL;
    struct udphdr *udph = NULL;
    unsigned short sport = 0;
    unsigned short dport = 0;
    if(iph->protocol==IPPROTO_TCP)
    {
        tcph = (struct tcphdr *)((char *)skb->data + (int)(iph->ihl * 4));
        sport=ntohs(tcph->source);
        dport=ntohs(tcph->dest);
    }
    else if(iph->protocal==IPPROTO_UDP)
    {
        udph = (struct udphdr *)((char *)skb->data + (int)(iph->ihl * 4));
        sport=ntohs(udph->source);
        dport=ntohs(udph->dest);
    }
    
    // tcp的数据
    // IHL(Internet Header Length 报头长度)，位于IP报文的第二个字段，4位，表示IP报文头部按32位
    // 字长（32位，4字节）计数的长度，也即报文头的长度等于IHL的值乘以4。 (ip_hl)
    char *data = NULL;
    struct tcphdr *tcph = (struct tcphdr *)((char *)skb->data + (int)(iph->ihl * 4));;
    data = (char *)((int)tcph + (int)(tcph->doff * 4));
    

通信方法 | 无法介于内核态与用户态的原因
---|---
管道（不包括命名管道） | 局限于父子进程间的通信。
消息队列 | 在硬、软中断中无法无阻塞地接收数据。
信号量 | 无法介于内核态和用户态使用。
内存共享 | 需要信号量辅助，而信号量又无法使用。
套接字 | 在硬、软中断中无法无阻塞地接收数据。

