### 构建镜像
docker build -t xlwh/sr-dev:main .

### 运行镜像
docker run -it \
-v /Users/zhb/.m2:/root/.m2 \
-v /Users/zhb/workspace/starrocks:/root/starrocks \
--name sr-dev \
-d xlwh/sr-dev:main

### 进入镜像
docker exec -it sr-dev /bin/bash