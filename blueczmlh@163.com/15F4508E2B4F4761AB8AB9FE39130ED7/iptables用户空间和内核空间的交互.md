int setsockopt(int sockfd, int proto, int cmd, void *data, int datlen)
int getsockopt(int sockfd, int proto, int cmd, void *data, int datalen)
参数说明:
sockfd: 为socket的文件描述符
proto: sock协议，IP RAW使用SOL_SOCKET/SOL_IP等，TCP/UDP socket可用SOL_SOCKET/SOL_IP/SOL_TCP/SOL_UDP
cmd: 自定义操作命令字
data：数据缓冲区起始位置指针
datalen：数据长度