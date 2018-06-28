# 网络搭建

本文将会讲解怎么一步一步去搭建一个本地的fabric网络. 在这篇文章中, 会尽量避免直接使用官方的例子以及那些脚本, 因为它们隐藏了许多细节, 而且翻墙使用起来也很难受. 

本文也只会针对OS X环境, 至于其他的诸如windows或者linux需要读者自行做相应调整.

本文中网络的搭建使用的是docker compose, 其他方式搭建太累.

假设读者已经安装了如下依赖:

* Golang 1.10 +
* NodeJS 8.11.2 +
* Docker 18.03 +
* Docker Compose 1.21.1 +

现在我们需要搭建一个fabric网络, 它比官方文档的示例网络更全面, 有更多的org, 更多的channel以及fabric-ca与couchdb的集成. 网络节点信息如下:

* 4个Org, 其中rca-\* 表示Root CA, ica-\*表示Intermediate CA. 每个peer会连一个couchdb.
  * 1个Orderer Org, 采用solo模式
    * themis: orderer1-themis, rca-themis, ica-themis
  * 3个Peer Org
    * google: peer1-google, peer2-google, rca-google, ica-google, couchdb-peer1-google, couchdb-peer2-google
    * baidu: peer1-baidu, peer2-baidu, rca-baidu, ica-baidu, couchdb-peer1-baidu, couchdb-peer2-baidu
    * bing: peer1-bing, peer2-bing, rca-bing, ica-bing, couchdb-peer1-bing, couchdb-peer2-bing
* 4个Channel
  * google-baidu-channel   google与baidu的通道
  * google-bing-channel    google与bing的通道
  * bing-baidu-channel    google与baidu的通道
  * all-in-one-channel  所有Org都加入的通道

网络拓扑图如下: 后面再补充

废话了一堆, 开始吧!

## 创建文件夹, 存放后面需要的脚本以及生成的配置和证书

文件夹名字就叫**bootstrap**好了

## 准备fabric-ca服务

官方说不建议在生产环境下使用cryptogen工具生成组织的msp, 所以我们还是用fabric-ca来做这些事情吧. 反正以后上生产也不能避免fabric-ca.

针对每个组织, 我们都有一个root ca以及一个intermediate ca.

下面就讲讲themis这个组织的docker-compose文件中内容以及fabric-ca-server启动的shell脚本

### rca容器

{% code-tabs %}
{% code-tabs-item title="docker-compose-rca-themis.yaml" %}
```yaml
services:
    rca-themis:    # 其他组织rca的服务名称请自行修改rca-google, rca-baidu, rca-bing
        container_name: rca-themis
        image: hyperledger/fabric-ca:x86_64-1.1.0 # 使用1.1.0版本
        environment:
            - FABRIC_CA_SERVER_HOME=/etc/hyperledger/fabric-ca
            - FABRIC_CA_SERVER_TLS_ENABLED=true # 启用TLS
            - FABRIC_CA_SERVER_CSR_CN=rca-themis
            - FABRIC_CA_SERVER_CSR_HOSTS=[rca-themis, localhost]
            - BOOTSTRAP_USER_PASS=rca-themis-admin:rca-themis-adminpw # ca服务启动的账号密码, start-root-ca.sh会使用
            - TARGET_CERTFILE=/data/themis-ca-cert.pem # 生成的ca根证书, start-root-ca.sh会使用
            - FABRIC_ORGS=themis google bing baidu # 用于生成affiliation, start-root-ca.sh会使用
        volumes:
            - ./scripts:/scripts # fabric-ca-server启动的脚本放这儿
            - ./data:/data  # fabric-ca-server生成的证书以及log放这儿
        command: /bin/bash -c '/scripts/start-root-ca.sh 2>&1 | tee /data/logs/rca-themis.log'
```
{% endcode-tabs-item %}

{% code-tabs-item title="start-root-ca.sh" %}
```bash
set -e
# 初始化rca, 账号:密码模板为 rca-themis-admin:rca-themis-adminpw
fabric-ca-server init -b $BOOTSTRAP_USER_PASS

# 将root cert复制到 /data/rca-themis-cert.pem, 后面的ica容器会用到这个证书
cp $FABRIC_CA_SERVER_HOME/ca-cert.pem $TARGET_CERTFILE

# 启动rca服务端
fabric-ca-server start
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### ica容器

{% code-tabs %}
{% code-tabs-item title="docker-compose-ica-themis.yaml" %}
```yaml
services:
    ica-themis:
      container_name: ica-themis # 其他组织ica的服务名称请自行修改ica-google, ica-baidu, ica-bing
      image: hyperledger/fabric-ca:x86_64-1.1.0
      environment:
        - FABRIC_CA_SERVER_HOME=/etc/hyperledger/fabric-ca
        - FABRIC_CA_SERVER_CA_NAME=ica-themis
        - FABRIC_CA_SERVER_INTERMEDIATE_TLS_CERTFILES=/data/themis-ca-cert.pem # 使用之前rca-themis生成的根证书
        - FABRIC_CA_SERVER_CSR_HOSTS=[ica-themis, localhost]
        - FABRIC_CA_SERVER_TLS_ENABLED=true # 启用TLS
        - BOOTSTRAP_USER_PASS=ica-themis-admin:ica-themis-adminpw # ca服务启动的账号密码, start-intermediate-ca.sh会使用
        - PARENT_URL=https://rca-themis-admin:rca-themis-adminpw@rca-themis:7054 # Root CA的连接信息, start-intermediate-ca.sh会使用
        - TARGET_CHAINFILE=/data/themis-ca-chain.pem # 生成的intermediate证书, 后面的peer节点需要使用到它
        - ORG=themis
        - FABRIC_ORGS=themis google bing baidu
      volumes:
        - ./scripts:/scripts
        - ./data:/data
      ports:
        - 7054:7054
      networks:
        - xyd-themis
      depends_on:
        - rca-themis # 需要rca先启动, 因为要拿到rca服务生成的根证书
      command: /bin/bash -c '/scripts/start-intermediate-ca.sh themis 2>&1 | tee /data/logs/ica-themis.log'

```
{% endcode-tabs-item %}

{% code-tabs-item title="start-intermediate-ca.sh" %}
```bash
set -e

# 初始化ica, 账号:密码模板为: ica-themis-admin:ica-themis-adminpw
# 同时需要制定此ica对应的rca连接信息
fabric-ca-server init -b $BOOTSTRAP_USER_PASS -u $PARENT_URL

# 将intermediate cert复制到 /data/ica-themis-chain.pem, 后面的peer会使用到这个证书
cp $FABRIC_CA_SERVER_HOME/ca-chain.pem $TARGET_CHAINFILE

# 启动ica服务端
fabric-ca-server start

```
{% endcode-tabs-item %}
{% endcode-tabs %}

## 准备fabric-ca-tools容器

hyperledger/fabric-ca-tools镜像中包含了如cryptogen, configtxgen以及fabric-ca-client等工具. 我们可以用它来完成注册组织的管理员, 生成组织msp以及生成fabric网络的配置等工作. 

