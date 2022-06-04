# StarRocks ARM环境下编译

### 系统环境

1. 系统版本：Ubuntu 20.04
2. 系统架构：ARM X64
3. CPU：4 C
4. 内存：16 GB
5. 硬盘：128GB（SSD）


### 编译环境安装

- 更新 apt-get 软件库

```shell
sudo apt update
```

- 检查shell命令集

ubuntu 的 shell 默认安装的是 dash，而不是 bash，要切换成 bash 才能执行，运行以下命令查看 sh 的详细信息，确认 shell 对应的程序是哪个：

```shell
ls -al /bin/sh
```

- 通过以下方式可以使 shell 切换回 bash：

```shell
sudo dpkg-reconfigure dash
```

然后选择 no 或者 否 ，并确认

这样做将重新配置 dash，并使其不作为默认的 shell 工具。

创建环境安装目录

```shell
mkdir ~/soft
```

- JDK8

  ```shell
  # 下载 arm64 架构的安装包，解压配置环境变量后使用
  cd ~/soft
  wget https://doris-thirdparty-repo.bj.bcebos.com/thirdparty/jdk-8u291-linux-aarch64.tar.gz && \
  	tar -zxvf jdk-8u291-linux-aarch64.tar.gz && \
  	rm jdk-8u291-linux-aarch64.tar.gz
  ```

- Maven

  ```shell
  cd ~/soft
  # wget 工具下载后，直接解压缩配置环境变量使用
  wget https://dlcdn.apache.org/maven/maven-3/3.6.3/binaries/apache-maven-3.6.3-bin.tar.gz && \
  	tar -zxvf apache-maven-3.6.3-bin.tar.gz && \
  	rm apache-maven-3.6.3-bin.tar.gz
  ```

- NodeJS

  ```shell
  cd ~/soft
  # 下载 arm64 架构的安装包
  wget https://doris-thirdparty-repo.bj.bcebos.com/thirdparty/node-v16.3.0-linux-arm64.tar.xz && \
  	tar -xvf node-v16.3.0-linux-arm64.tar.xz && \
  	rm node-v16.3.0-linux-arm64.tar.xz
  ```

- 配置环境变量

```shell
# 配置环境变量
vim /etc/profile.d/srenv.sh
export JAVA_HOME=${HOME}/soft/jdk1.8.0_291
export MAVEN_HOME=${HOME}/soft/apache-maven-3.6.3
export NODE_JS_HOME=${HOME}/soft/node-v16.3.0-linux-arm64
export PATH=$JAVA_HOME/bin:$MAVEN_HOME/bin:$NODE_JS_HOME/bin:$PATH
```



- 安装GCC 10

```shell
sudo add-apt-repository ppa:ubuntu-toolchain-r/ppa 
sudo apt update
sudo apt install gcc-10 g++-10
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-10 50 --slave /usr/bin/g++ g++ /usr/bin/g++-10 --slave /usr/bin/gcov gcov /usr/bin/gcov-10
```



- 安装Cmake

```
wget -O - https://apt.kitware.com/keys/kitware-archive-latest.asc 2>/dev/null | sudo apt-key add -
sudo apt-add-repository 'deb https://apt.kitware.com/ubuntu/ bionic main'
sudo apt update
sudo apt install cmake
```



- 安装其他软件包

```shell
sudo apt install -y build-essential cmake flex automake bison binutils-dev libiberty-dev zip libncurses5-dev curl ninja-build
sudo apt-get install -y make unzip python2 byacc automake libtool bzip2 ccache openssl libssl-dev
```

- 安装链接工具：https://github.com/xlwh/mold



### 编译准备

修改aws-sdk

```
vim /media/psf/workspace/starrocks/thirdparty/src/aws-sdk-cpp-1.9.179/crt/aws-crt-cpp/crt/aws-c-io/source/posix/host_resolver.c
增加一行：#define _POSIX_C_SOURCE 200112
```



修改链接器

```
+++ b/build.sh
@@ -200,7 +200,7 @@ if [ ${BUILD_BE} -eq 1 ] ; then
                     -DMAKE_TEST=OFF -DWITH_GCOV=${WITH_GCOV}\
                     -DUSE_AVX2=$USE_AVX2 -DUSE_SSE4_2=$USE_SSE4_2 \
                     -DCMAKE_EXPORT_COMPILE_COMMANDS=ON ..
-    time ${BUILD_SYSTEM} -j${PARALLEL}
+    time mold -run ${BUILD_SYSTEM} -j${PARALLEL}
```

ac local文件解决：https://blog.csdn.net/yetyongjin/article/details/108050490

