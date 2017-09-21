# <center> iptables用户空间和内核空间的交互 </center>

libiptc(iptables control library):

    int setsockopt(int sockfd, int proto, int cmd, void *data, int datlen)
    int getsockopt(int sockfd, int proto, int cmd, void *data, int datalen)

参数说明:

    sockfd: 为socket的文件描述符
    proto: sock协议，IP RAW使用SOL_SOCKET/SOL_IP等，TCP/UDP socket可用SOL_SOCKET/SOL_IP/SOL_TCP/SOL_UDP
    cmd: 自定义操作命令字
    data：数据缓冲区起始位置指针
    datalen：数据长度

```
// nf_sockopt_ops结构体定义在include/linux/netfilter中
struct nf_sockopt_ops {
	struct list_head list;

	u_int8_t pf;

	/* Non-inclusive ranges: use 0/0/NULL to never get called. */
	int set_optmin;
	int set_optmax;
	int (*set)(struct sock *sk, int optval, void __user *user, unsigned int len);
#ifdef CONFIG_COMPAT
	int (*compat_set)(struct sock *sk, int optval,
			void __user *user, unsigned int len);
#endif
	int get_optmin;
	int get_optmax;
	int (*get)(struct sock *sk, int optval, void __user *user, int *len);
#ifdef CONFIG_COMPAT
	int (*compat_get)(struct sock *sk, int optval,
			void __user *user, int *len);
#endif
	/* Use the module struct to lock set/get code in place */
	struct module *owner;
};
```
