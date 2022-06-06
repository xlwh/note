FROM starrocks/dev-env:main

# install epel repo for ccache
RUN yum install epel-release -y && yum clean all && yum makecache

# install dependencies
RUN yum install -y ccache wget

# clean cache
RUN yum clean all

# disable auto enable ccache
RUN rm -f /etc/profile.d/ccache.*

RUN wget https://github.com/ninja-build/ninja/releases/download/v1.11.0/ninja-linux.zip && \
  unzip ninja-linux.zip && mv ./ninja /usr/local/bin/ && rm ninja-linux.zip