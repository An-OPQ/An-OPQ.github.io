---
title: ElasticSearch7.16.2版本安装hanlp插件
cover: https://w.wallhaven.cc/full/3z/wallhaven-3z1p3d.jpg
tag: ElasticSearch
---

💡 本教程仅用于记录，若写的不好，请多多包涵！ 

我在工作中，需要安装Hanlp插件。ElasticSearch的版本为：7.16.2 

经过我一番Google之后，发现市面上的教程大多集中在ElasticSearch版本7.15以下，并没有我所需要的7.16.2的版本，大伙儿都知道，ES插件需要和ES 版本一致才能安装。 而：大多教程都是叫自己 拉代码然后自行编译。 而我经过多次尝试：最终都被 版本跨度、和JDK11才能编译 等等一些列问题给劝退了! 那不无解了？ 

🤺🤺🤺 退！退！退！

🤺🤺🤺 退！退！退！ 

经过同事的漫长寻找最后找到了一位大佬遗留的秘籍。那么大伙儿就随我一起来观摩观摩这本秘籍吧！ 

### 下载

原地址： [https://github.com/muxiaobai/elasticsearch-analysis-hanlp/releases/tag/v7.16.2](https://github.com/muxiaobai/elasticsearch-analysis-hanlp/releases/tag/v7.16.2) 

![](1.png)

 `终于在大佬的一番话语中，探明了前进的方向`

 虽说这位大佬提供了jar包，但是仍然需要自己替换zip包。 

所以我找到其他的zip 下载地址： [Releases · KennFalcon/elasticsearch-analysis-hanlp](https://github.com/KennFalcon/elasticsearch-analysis-hanlp/releases)  

### 配置

我们将zip与jar包下载好之后 

1. 替换jar包。并删除原版本 将7.10.0版本的zip解压，然后将7.16.2版本的jar包替换进去，删除原7.10.0的jar包 

![image-20220717200849914](image-20220717200849914-16621905055872.png)

![image-20220717200858007](image-20220717200858007.png)

2. 修改配置 

​	如图：打开plugin-descriptor.properties 文件 修改对于的两个version为我们替换进去的jar版本 

![image-20220717200907763](image-20220717200907763.png)

重新打包成zip包，并修改文件名。

 `ps：这里一定要注意：需要在打包前就将文件夹的名称版本改好，在进行打包成zip` 

3.`修改另一个配置`（重要点！)

 我在测试与实际安装过程中，遇到过好几次这种问题。报错信息请看下图，这里我并没有深究：这里是权限的配置。我开放了所有权限，请大伙儿这里需要注意。 但是若使用原配置启动不了。

![image-20220717200619740](image-20220717200619740.png)

```tex
permission java.net.SocketPermission "*", "connect,resolve";
permission java.io.FilePermission "-","read";
```

![image-20220717200501138](image-20220717200501138.png) 

### 安装

我们使用Docker-compose进行安装。若不会docker-compose的同学需要自行学习一下。

```yaml
version: "3"
services:
  elasticsearch:
    image: elasticsearch:7.16.2
    container_name: elasticsearch
    restart: always
    ports:
      - "9200:9200"
      - "9300:9300"
    environment:
      - discovery.type=single-node
      - http.cors.enabled=true #设置允许跨域
      - http.cors.allow-origin=*
      - bootstrap.memory_lock=true   # 内存交换的选项，官网建议为true
			- TAKE_FILE_OWNERSHIP=true   # 权限问题
    volumes:
      - /docker/elasticsearch/logs:/usr/share/elasticsearch/logs
      - /docker/elasticsearch/data:/usr/share/elasticsearch/data
```



docker-compose启动

``` 
# docker-compose启动
docker-compose up -d 
```

![image-20220717200521619](image-20220717200521619.png)

 `拉取docker镜像中` 

```bash
docker-compose logs -f 
```

查看启动日志![image-20220717200535312](https://an-opq.github.io/2022/07/17/ElasticSearch7-16-2%E7%89%88%E6%9C%AC%E5%AE%89%E8%A3%85hanlp%E6%8F%92%E4%BB%B6/image-20220717200535312.png) `日志也正常启动` 访问： [http://xxxx:9200](http://xxxx:9200/) 

![image-20220717200548881](image-20220717200548881.png)

 可以正常访问 

-------------

### 安装插件

那么就来安装hanlp插件吧！

1. 上传修改好的插件到服务器中 ![image-20220717200558087](image-20220717200558087.png) 

2. 复制到docker容器内部

```bash
docker cp ./elasticsearch-analysis-hanlp-7.16.2.zip elasticsearch:/usr/share/elasticsearch
```

3. 进入容器内部 ![image-20220717200608440](https://an-opq.github.io/2022/07/17/ElasticSearch7-16-2%E7%89%88%E6%9C%AC%E5%AE%89%E8%A3%85hanlp%E6%8F%92%E4%BB%B6/image-20220717200608440.png) 

````bash
docker exec -it elasticsearch bash 
````

4. 安装

```bash
./bin/elasticsearch-plugin install file:///usr/share/elasticsearch/elasticsearch-analysis-hanlp-7.16.2.zip
```

5. 安装成功

由于我权限开放较大，所以会有警告

![image-20220717200628153](image-20220717200628153.png)

6. 重启Docker容器

```bash
# 从容器中退出
exit
# 重启容器
docker restart elasticsearch
```

7. 测试

从原理来说：由于ES小版本之间的区别不大，所以按理说 7.16.x 之后的版本 之后改动不大，都可以按照上述方法改。

![image-20220717200640068](image-20220717200640068.png)