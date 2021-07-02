FROM centos:latest
RUN yum install openssh-server -y
RUN echo "PasswordAuthentication yes" >> /etc/ssh/sshd_config
RUN echo "root:centos" | chpasswd
RUN ssh-keygen -A
CMD ["/usr/sbin/sshd","-D"]
EXPOSE 22
