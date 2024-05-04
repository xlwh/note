FROM starrocks/dev-env-ubuntu:latest

# 设置时区
ARG TZ=Asia/Shanghai
ENV TZ ${TZ}
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

#安装基本的网络调试工具，和tmxu,git
RUN yum -y install wget \
  && yum -y install net-tools nc telnet \
  && yum -y install tmux \
  && yum -y install vim

#开启ssh服务
RUN yum -y install passwd openssh-server openssl-devel
#设置root用户密码为root
RUN /bin/echo "root" | passwd --stdin root
RUN ssh-keygen -t rsa -f /etc/ssh/ssh_host_rsa_key \
  && ssh-keygen -t rsa -f /etc/ssh/ssh_host_ecdsa_key \
  && ssh-keygen -t rsa -f /etc/ssh/ssh_host_ed25519_key

#启动 ssh 服务
CMD ["/usr/sbin/sshd","-D"]

