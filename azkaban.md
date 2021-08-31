azkaban提供单机模式和集群模式。单机模式数据库基于H2数据库，集群模式基于配置了主从的MySQL。

### 源码编译
```
# Build Azkaban
./gradlew build

# Clean the build
./gradlew clean

# Build and install distributions
./gradlew installDist

# Run tests
./gradlew test

# Build without running tests
./gradlew build -x test
```


## 单机模式
```
git clone https://github.com/azkaban/azkaban.git

cd azkaban; ./gradlew build installDist

cd azkaban-solo-server/build/install/azkaban-solo-server; bin/start-solo.sh
```

> http://localhost:8081/ 默认用户名密码azkaban
> 可在conf/azkaban-users.xml配置

### 配置SSL
单机模式默认不启用SSL，可以在azkaban.properties中配置jetty来设置SSL。

## 集群模式
```
mysql库、表配置

cd azkaban-db; ../gradlew build installDist

azkaban/azkaban-db/build/distributions/azkaban-db-<version> // create-all-sql-<version>.sql

修改azkaban.properties中的MySQL配置

cd azkaban-exec-server/build/install/azkaban-exec-server
./bin/start-exec.sh

cd azkaban-web-server/build/install/azkaban-web-server
./bin/start-web.sh
```

## 插件

