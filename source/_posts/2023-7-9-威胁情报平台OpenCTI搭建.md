---
title: 威胁情报平台OpenCTI搭建
author: jh2ng
cover: 'https://image.jh2ing.com/20230709201658.png'
tags:
  - 环境搭建
categories:
  - 网络安全
abbrlink: 10500
date: 2023-07-09 00:00:00
---
### 一、介绍
OpenCTI 即 Open Cyber Threat Intelligence Platform，开源网络威胁情报平台。它的创建是为了构建、存储、组织和可视化有关网络威胁的技术和非技术信息。它使用基于 STIX 2 标准的知识模式来执行数据的结构化。并被设计为现代 Web 应用程序，包括 GraphQL API 和面向 UX的前端。此外，OpenCTI 可以与其他工具和应用程序集成，如 MISP、TheHive、MITRE ATT&CK 等。
![20230709111233](https://image.jh2ing.com/20230709111233.png)

### 二、部署
工作原理及部署要求：`https://docs.opencti.io/5.8.X/deployment/overview/`

我的环境：
- CPU：16核
- 内存：32G
- 硬盘：150G
- 操作系统：Ubuntu 22.04.2 LTS
- IP地址：192.168.0.20/24
- docker和docker-compose（自行安装）

1. 创建OpenCTI工作目录

```bash
mkdir /data/opencti && cd /data/opencti
```
2. 从github上克隆opencti项目，并进入docker目录


```bash
git clone https://github.com/OpenCTI-Platform/docker.git &&cd docker
```
3. 设置环境变量.env

`OPENCTI_ADMIN_EMAI`设置账户名
`OPENCTI_ADMIN_PASSWORD`设置密码
```bash
(cat << EOF
OPENCTI_ADMIN_EMAIL=admin@opencti.io  
OPENCTI_ADMIN_PASSWORD=ChangeMePlease
OPENCTI_ADMIN_TOKEN=$(cat /proc/sys/kernel/random/uuid)
MINIO_ROOT_USER=$(cat /proc/sys/kernel/random/uuid)
MINIO_ROOT_PASSWORD=$(cat /proc/sys/kernel/random/uuid)
RABBITMQ_DEFAULT_USER=guest
RABBITMQ_DEFAULT_PASS=guest
ELASTIC_MEMORY_SIZE=4G
CONNECTOR_HISTORY_ID=$(cat /proc/sys/kernel/random/uuid)
CONNECTOR_EXPORT_FILE_STIX_ID=$(cat /proc/sys/kernel/random/uuid)
CONNECTOR_EXPORT_FILE_CSV_ID=$(cat /proc/sys/kernel/random/uuid)
CONNECTOR_IMPORT_FILE_STIX_ID=$(cat /proc/sys/kernel/random/uuid)
CONNECTOR_IMPORT_REPORT_ID=$(cat /proc/sys/kernel/random/uuid)
EOF
) > .env
```
4. 将.env参数导入环境变量

```bash
export $(cat .env | grep -v "#" | xargs)
```
5. 内存管理设置

> 由于OpenCTI依赖于ElasticSearch，ElasticSearch对内存要求较高，因此需要调整机器内存参数。将vm.max_map_count=1048575写入/etc/sysctl.conf的末尾

6. 使用 docker-compose部署opencti

```bash
docker-compose up -d
```

启动之后等待几分钟访问。
![20230709185842](https://image.jh2ing.com/20230709185842.png)

### 三、连接器配置导入数据
模板地址：`https://github.com/OpenCTI-Platform/connectors`

以最简单方便的CVE连接器配置为例子
在模板地址中找到cve模板，路径：`external-import/cve/docker-compose.yml`
只需要这部分内容：
```yaml
connector-cve:
  image: opencti/connector-cve:5.8.7
  environment:
    - OPENCTI_URL=http://localhost
    - OPENCTI_TOKEN=ChangeMe
    - CONNECTOR_ID=ChangeMe
    - CONNECTOR_TYPE=EXTERNAL_IMPORT
    - CONNECTOR_NAME=Common Vulnerabilities and Exposures
    - CONNECTOR_SCOPE=identity,vulnerability
    - CONNECTOR_CONFIDENCE_LEVEL=75 # From 0 (Unknown) to 100 (Fully trusted)
    - CONNECTOR_UPDATE_EXISTING_DATA=false
    - CONNECTOR_RUN_AND_TERMINATE=false
    - CONNECTOR_LOG_LEVEL=info
    - CVE_IMPORT_HISTORY=true # Import history at the first run (after only recent), reset the connector state if you want to re-import
    - CVE_NVD_DATA_FEED=https://nvd.nist.gov/feeds/json/cve/1.1/nvdcve-1.1-recent.json.gz
    - CVE_HISTORY_START_YEAR=2002
    - CVE_HISTORY_DATA_FEED=https://nvd.nist.gov/feeds/json/cve/1.1/
    - CVE_INTERVAL=7 # In days, must be strictly greater than 1
  restart: always
```
在之前OpenCTI的`docker-compose.yml`中可以看到
```yaml
 - OPENCTI_URL=http://opencti:8080
  - OPENCTI_TOKEN=${OPENCTI_ADMIN_TOKEN}
  - CONNECTOR_ID=${CONNECTOR_IMPORT_DOCUMENT_ID} # Valid UUIDv4
```
根据OpenCTI的`docker-compose.yml`中的配置对CVE连接器进行配置。
UUIDv4的值，可以在线生成`https://www.uuidtools.com/generate/v4`，也可以本地获取uuidv4`cat /proc/sys/kernel/random/uuid`
```yaml
connector-cve:
  image: opencti/connector-cve:5.8.7
  environment:
    - OPENCTI_URL=http://opencti:8080
    - OPENCTI_TOKEN=${OPENCTI_ADMIN_TOKEN}
    - CONNECTOR_ID=207530ac-6ae5-4c38-8bba-9932ddad01d8
    - CONNECTOR_TYPE=EXTERNAL_IMPORT
    - CONNECTOR_NAME=Common Vulnerabilities and Exposures
    - CONNECTOR_SCOPE=identity,vulnerability
    - CONNECTOR_CONFIDENCE_LEVEL=75 # From 0 (Unknown) to 100 (Fully trusted)
    - CONNECTOR_UPDATE_EXISTING_DATA=false
    - CONNECTOR_RUN_AND_TERMINATE=false
    - CONNECTOR_LOG_LEVEL=info
    - CVE_IMPORT_HISTORY=true # Import history at the first run (after only recent), reset the connector state if you want to re-import
    - CVE_NVD_DATA_FEED=https://nvd.nist.gov/feeds/json/cve/1.1/nvdcve-1.1-recent.json.gz
    - CVE_HISTORY_START_YEAR=2002
    - CVE_HISTORY_DATA_FEED=https://nvd.nist.gov/feeds/json/cve/1.1/
    - CVE_INTERVAL=7 # In days, must be strictly greater than 1
  restart: always
```
将配置写入`docker-compose.yml`中，位置如下
注意前后有
```yaml
depends_on:
  - opencti
```
![20230709201312](https://image.jh2ing.com/20230709201312.png)
然后执行docker-compose up -d进行镜像的更新和拉取。可以看到已经有实体数据。
![20230709201658](https://image.jh2ing.com/20230709201658.png)
其他连接器跟以上过程差不多，区别在于需要填写一些其他的参数如数据源的平台地址以及API等等。

**参考链接**
> https://docs.opencti.io/5.8.X/deployment/overview/