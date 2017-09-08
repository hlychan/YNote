数据包在协议栈中传递时会经过不同的HOOK点，而每个HOOK点上又被Netfilter预先注册了一系列的HOOK回调函数。当数据包经过这些HOOK点时，这些HOOK函数会处理数据包。

要注册一个hook函数需要用到nf_register_hook()或者nf_register_hooks()内核API和一个struct nf_hook_ops{}类型的结构体对象
nf_hook_ops定义在linuxinclude/linux/netfilter.h中，内容如下：
```
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
```
用户空间操作已经注册到内核的hook函数（用户空间和内核空间的通信）
借鉴iptables和内核的通信方式，即getsockopt/setsockopt
上面我们使用nf_register_hokk()注册nf_sockopt_ops结构体对象，而现在我们需要使用nf_register_sockopt()将该对象注册到全局双向链表nf_sockopts中去，当我们在用户空间调用getsockopt/setsockopt时经过层层系统调用，最后就会在nf_sockopts链表中找到我们已经注册的响应函数。
