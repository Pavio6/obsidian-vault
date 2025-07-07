# 使用Docker Compose启动bitnami/Elasticsearch和bitnami/Kibana并配置ik中文分词器
## 1. 创建项目目录
```Shell
# 创建项目目录
mkdir -p ~/elastic-ik-stack/{es-data,kibana-data,plugins}

# 进入项目目录
cd ~/elastic-ik-stack
```
## 2. 准备 IK 分词器插件

```Shell
# 从主机将压缩包直接放到plugins文件夹中，并进行解压
# 解压 IK 分词器（确保解压到 analysis-ik 目录）
unzip elasticsearch-analysis-ik-8.14.3.zip -d analysis-ik
# 删除原始压缩包
rm elasticsearch-analysis-ik-8.14.3.zip
# 返回项目根目录
cd ..
```
## 3. 创建 Docker Compose 文件
```Shell
cat > docker-compose.yml <<EOF
version: '3.8'

services:
  elasticsearch:
    image: bitnami/elasticsearch:8.14.3
    container_name: elasticsearch
    environment:
      # 禁用安全特性简化测试
      - ELASTICSEARCH_ENABLE_SECURITY=no
      - ELASTICSEARCH_SKIP_TRANSPORT_TLS=yes
      # 配置 Java 堆大小
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
      # 禁用磁盘阈值检查
      - cluster.routing.allocation.disk.threshold_enabled=false
    volumes:
      # 挂载数据目录
      - ./es-data:/bitnami/elasticsearch/data
      # 挂载 IK 分词器插件
      - ./plugins/analysis-ik:/opt/bitnami/elasticsearch/plugins/analysis-ik
    ports:
      - "9200:9200"
      - "9300:9300"
    networks:
      - elk-network
    restart: unless-stopped
    privileged: true  # 授予特权解决权限问题

  kibana:
    image: bitnami/kibana:8.14.3
    container_name: kibana
    depends_on:
      - elasticsearch
    environment:
      # 禁用安全特性
      - KIBANA_ENABLE_SECURITY=no
      # 配置 Elasticsearch 连接
      - KIBANA_ELASTICSEARCH_URL=http://elasticsearch:9200
    volumes:
      # 挂载 Kibana 数据目录
      - ./kibana-data:/bitnami/kibana
    ports:
      - "5601:5601"
    networks:
      - elk-network
    restart: unless-stopped

networks:
  elk-network:
    driver: bridge
EOF
```
## 4. 设置目录权限
```Shell
# 设置所有目录为可读可写
sudo chmod -R 777 ~/elastic-ik-stack
```
## 5. 启动服务
```Shell
# 启动容器
docker compose up -d
# 查看容器状态
docker ps
```