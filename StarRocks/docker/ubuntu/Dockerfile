FROM ubuntu:20.04

# ARG for quick switch to a given ubuntu mirror
ARG apt_archive="http://archive.ubuntu.com"
RUN sed -i "s|http://archive.ubuntu.com|$apt_archive|g" /etc/apt/sources.list

RUN mkdir /root/soft && cd /root/soft
RUN wget https://doris-thirdparty-repo.bj.bcebos.com/thirdparty/jdk-8u291-linux-aarch64.tar.gz && tar -zxvf jdk-8u291-linux-aarch64.tar.gz && rm jdk-8u291-linux-aarch64.tar.gz
RUN wget https://dlcdn.apache.org/maven/maven-3/3.6.3/binaries/apache-maven-3.6.3-bin.tar.gz && tar -zxvf apache-maven-3.6.3-bin.tar.gz && rm apache-maven-3.6.3-bin.tar.gz
RUN wget https://doris-thirdparty-repo.bj.bcebos.com/thirdparty/node-v16.3.0-linux-arm64.tar.xz && tar -xvf node-v16.3.0-linux-arm64.tar.xz && rm node-v16.3.0-linux-arm64.tar.xz
RUN add-apt-repository ppa:ubuntu-toolchain-r/ppa 
RUN apt update
RUN apt install gcc-10 g++-10 --yes --no-install-recommends
RUN update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-10 50 --slave /usr/bin/g++ g++ /usr/bin/g++-10 --slave /usr/bin/gcov gcov /usr/bin/gcov-10
RUN wget -O - https://apt.kitware.com/keys/kitware-archive-latest.asc 2>/dev/null | sudo apt-key add -
RUN sudo apt-add-repository 'deb https://apt.kitware.com/ubuntu/ bionic main'
RUN sudo apt update --yes --no-install-recommends
RUN sudo apt install cmake --yes --no-install-recommends