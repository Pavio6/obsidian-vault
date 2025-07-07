### Docker Compose 是一个用于定义和运行多容器 Docker 应用的工具。
##### 基本命令
```shell
# 启动项目（-d 后台运行） 
docker compose up -d 
# -f 指定具体的compose文件
docker compose -f compose.yml up -d 
# 停止项目 
docker compose down 
# --rmi all 删除与服务关联的镜像
# -v 删除与服务关联的卷
docker compose -f compose.yml down --rmi all -v
# 查看服务状态 
docker compose ps 
# 查看服务日志 
docker compose logs -f 服务名 
# -f 持续输出日志 
# 重启服务 
docker compose restart 服务名 
# 重建并启动服务（修改 YAML 后使用） 
docker compose up -d --build 服务名
```

---
### Dockerfile 是用于构建 Docker 镜像的文本文件，其中包含了一系列命令和指令，用于定义如何创建镜像以及在容器中运行应用程序。
#### 制作镜像
通过执行 Dockerfile 中的指令，将应用程序、依赖项、配置等打包成一个独立的文件系统（镜像）。
#### 镜像分层机制 
通过将镜像拆分为多个独立的 “层”（Layer）来实现高效的存储、传输和复用。


**compose.yml的内容**
```yml
name: devsoft  
services:  
  redis:  
    image: bitnami/redis:latest  
    restart: always  
    container_name: redis  
    environment:  
      - REDIS_PASSWORD=123456  
    ports:  
      - '6379:6379'  
    volumes:  
      - redis-data:/bitnami/redis/data  
      - redis-conf:/opt/bitnami/redis/mounted-etc  
      - /etc/localtime:/etc/localtime:ro  
  
  mysql:  
    image: mysql:8.0.31  
    restart: always  
    container_name: mysql  
    environment:  
      - MYSQL_ROOT_PASSWORD=123456  
    ports:  
      - '3306:3306'  
      - '33060:33060'  
    volumes:  
      - mysql-conf:/etc/mysql/conf.d  
      - mysql-data:/var/lib/mysql  
      - /etc/localtime:/etc/localtime:ro  
  
  rabbit:  
    image: rabbitmq:3-management  
    restart: always  
    container_name: rabbitmq  
    ports:  
      - "5672:5672"  
      - "15672:15672"  
    environment:  
      - RABBITMQ_DEFAULT_USER=rabbit  
      - RABBITMQ_DEFAULT_PASS=rabbit  
      - RABBITMQ_DEFAULT_VHOST=dev  
    volumes:  
      - rabbit-data:/var/lib/rabbitmq  
      - rabbit-app:/etc/rabbitmq  
      - /etc/localtime:/etc/localtime:ro  
  opensearch-node1:  
    image: opensearchproject/opensearch:2.13.0  
    container_name: opensearch-node1  
    environment:  
      - cluster.name=opensearch-cluster # Name the cluster  
      - node.name=opensearch-node1 # Name the node that will run in this container  
      - discovery.seed_hosts=opensearch-node1,opensearch-node2 # Nodes to look for when discovering the cluster  
      - cluster.initial_cluster_manager_nodes=opensearch-node1,opensearch-node2 # Nodes eligibile to serve as cluster manager  
      - bootstrap.memory_lock=true # Disable JVM heap memory swapping  
      - "OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m" # Set min and max JVM heap sizes to at least 50% of system RAM  
      - "DISABLE_INSTALL_DEMO_CONFIG=true" # Prevents execution of bundled demo script which installs demo certificates and security configurations to OpenSearch  
      - "DISABLE_SECURITY_PLUGIN=true" # Disables Security plugin  
    ulimits:  
      memlock:  
        soft: -1 # Set memlock to unlimited (no soft or hard limit)  
        hard: -1  
      nofile:  
        soft: 65536 # Maximum number of open files for the opensearch user - set to at least 65536  
        hard: 65536  
    volumes:  
      - opensearch-data1:/usr/share/opensearch/data # Creates volume called opensearch-data1 and mounts it to the container  
      - /etc/localtime:/etc/localtime:ro  
    ports:  
      - 9200:9200 # REST API  
      - 9600:9600 # Performance Analyzer  
  
  opensearch-node2:  
    image: opensearchproject/opensearch:2.13.0  
    container_name: opensearch-node2  
    environment:  
      - cluster.name=opensearch-cluster # Name the cluster  
      - node.name=opensearch-node2 # Name the node that will run in this container  
      - discovery.seed_hosts=opensearch-node1,opensearch-node2 # Nodes to look for when discovering the cluster  
      - cluster.initial_cluster_manager_nodes=opensearch-node1,opensearch-node2 # Nodes eligibile to serve as cluster manager  
      - bootstrap.memory_lock=true # Disable JVM heap memory swapping  
      - "OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m" # Set min and max JVM heap sizes to at least 50% of system RAM  
      - "DISABLE_INSTALL_DEMO_CONFIG=true" # Prevents execution of bundled demo script which installs demo certificates and security configurations to OpenSearch  
      - "DISABLE_SECURITY_PLUGIN=true" # Disables Security plugin  
    ulimits:  
      memlock:  
        soft: -1 # Set memlock to unlimited (no soft or hard limit)  
        hard: -1  
      nofile:  
        soft: 65536 # Maximum number of open files for the opensearch user - set to at least 65536  
        hard: 65536  
    volumes:  
      - /etc/localtime:/etc/localtime:ro  
      - opensearch-data2:/usr/share/opensearch/data # Creates volume called opensearch-data2 and mounts it to the container  
  
  opensearch-dashboards:  
    image: opensearchproject/opensearch-dashboards:2.13.0  
    container_name: opensearch-dashboards  
    ports:  
      - 5601:5601 # Map host port 5601 to container port 5601  
    expose:  
      - "5601" # Expose port 5601 for web access to OpenSearch Dashboards  
    environment:  
      - 'OPENSEARCH_HOSTS=["http://opensearch-node1:9200","http://opensearch-node2:9200"]'  
      - "DISABLE_SECURITY_DASHBOARDS_PLUGIN=true" # disables security dashboards plugin in OpenSearch Dashboards  
    volumes:  
      - /etc/localtime:/etc/localtime:ro  
  zookeeper:  
    image: bitnami/zookeeper:3.9  
    container_name: zookeeper  
    restart: always  
    ports:  
      - "2181:2181"  
    volumes:  
      - "zookeeper_data:/bitnami"  
      - /etc/localtime:/etc/localtime:ro  
    environment:  
      - ALLOW_ANONYMOUS_LOGIN=yes  
  
  kafka:  
    image: 'bitnami/kafka:3.4'  
    container_name: kafka  
    restart: always  
    hostname: kafka  
    ports:  
      - '9092:9092'  
      - '9094:9094'  
    environment:  
      - KAFKA_CFG_NODE_ID=0  
      - KAFKA_CFG_PROCESS_ROLES=controller,broker  
      - KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093,EXTERNAL://0.0.0.0:9094  
      - KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092,EXTERNAL://119.45.147.122:9094  
      - KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,EXTERNAL:PLAINTEXT,PLAINTEXT:PLAINTEXT  
      - KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=0@kafka:9093  
      - KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER  
      - ALLOW_PLAINTEXT_LISTENER=yes  
      - "KAFKA_HEAP_OPTS=-Xmx512m -Xms512m"  
    volumes:  
      - kafka-conf:/bitnami/kafka/config  
      - kafka-data:/bitnami/kafka/data  
      - /etc/localtime:/etc/localtime:ro  
  kafka-ui:  
    container_name: kafka-ui  
    image: provectuslabs/kafka-ui:latest  
    restart: always  
    ports:  
      - 8080:8080  
    environment:  
      DYNAMIC_CONFIG_ENABLED: true  
      KAFKA_CLUSTERS_0_NAME: kafka-dev  
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092  
    volumes:  
      - kafkaui-app:/etc/kafkaui  
      - /etc/localtime:/etc/localtime:ro  
  
  nacos:  
    image: nacos/nacos-server:v2.3.1  
    container_name: nacos  
    ports:  
      - 8848:8848  
      - 9848:9848  
    environment:  
      - PREFER_HOST_MODE=hostname  
      - MODE=standalone  
      - JVM_XMX=512m  
      - JVM_XMS=512m  
      - SPRING_DATASOURCE_PLATFORM=mysql  
      - MYSQL_SERVICE_HOST=nacos-mysql  
      - MYSQL_SERVICE_DB_NAME=nacos_devtest  
      - MYSQL_SERVICE_PORT=3306  
      - MYSQL_SERVICE_USER=nacos  
      - MYSQL_SERVICE_PASSWORD=nacos  
      - MYSQL_SERVICE_DB_PARAM=characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=Asia/Shanghai&allowPublicKeyRetrieval=true  
      - NACOS_AUTH_IDENTITY_KEY=2222  
      - NACOS_AUTH_IDENTITY_VALUE=2xxx  
      - NACOS_AUTH_TOKEN=SecretKey012345678901234567890123456789012345678901234567890123456789  
      - NACOS_AUTH_ENABLE=true  
    volumes:  
      - /app/nacos/standalone-logs/:/home/nacos/logs  
      - /etc/localtime:/etc/localtime:ro  
    depends_on:  
      nacos-mysql:  
        condition: service_healthy  
  nacos-mysql:  
    container_name: nacos-mysql  
    build:  
      context: .  
      dockerfile_inline: |  
        FROM mysql:8.0.31  
        ADD https://raw.githubusercontent.com/alibaba/nacos/2.3.2/distribution/conf/mysql-schema.sql /docker-entrypoint-initdb.d/nacos-mysql.sql  
        RUN chown -R mysql:mysql /docker-entrypoint-initdb.d/nacos-mysql.sql  
        EXPOSE 3306  
        CMD ["mysqld", "--character-set-server=utf8mb4", "--collation-server=utf8mb4_unicode_ci"]  
    image: nacos/mysql:8.0.30  
    environment:  
      - MYSQL_ROOT_PASSWORD=root  
      - MYSQL_DATABASE=nacos_devtest  
      - MYSQL_USER=nacos  
      - MYSQL_PASSWORD=nacos  
      - LANG=C.UTF-8  
    volumes:  
      - nacos-mysqldata:/var/lib/mysql  
      - /etc/localtime:/etc/localtime:ro  
    ports:  
      - "13306:3306"  
    healthcheck:  
      test: [ "CMD", "mysqladmin" ,"ping", "-h", "localhost" ]  
      interval: 5s  
      timeout: 10s  
      retries: 10  
  prometheus:  
    image: prom/prometheus:v2.52.0  
    container_name: prometheus  
    restart: always  
    ports:  
      - 9090:9090  
    volumes:  
      - prometheus-data:/prometheus  
      - prometheus-conf:/etc/prometheus  
      - /etc/localtime:/etc/localtime:ro  
  
  grafana:  
    image: grafana/grafana:10.4.2  
    container_name: grafana  
    restart: always  
    ports:  
      - 3000:3000  
    volumes:  
      - grafana-data:/var/lib/grafana  
      - /etc/localtime:/etc/localtime:ro  
  
volumes:  
  redis-data:  
  redis-conf:  
  mysql-conf:  
  mysql-data:  
  rabbit-data:  
  rabbit-app:  
  opensearch-data1:  
  opensearch-data2:  
  nacos-mysqldata:  
  zookeeper_data:  
  kafka-conf:  
  kafka-data:  
  kafkaui-app:  
  prometheus-data:  
  prometheus-conf:  
  grafana-data:
```

`KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092,EXTERNAL://119.45.147.122:9094  `这里的地址119.45.147.122改为自己虚拟机或服务器地址
