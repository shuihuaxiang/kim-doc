# springcache与redis
一般缓存直接用springcache，有时候也用mybatis二级缓存对接redis，特殊要求的缓存单独写redis。

## 缓存读写问题
1.写问题
 数据库和缓存一致性，springcache没有管
2.读问题
- 穿透：配置文件增加cache-null-values: true，缓存空值
- 雪崩：配置文件增加time-to-live: 3600s，增加过期时间，与随机过期时间差不多（随机时间也有可能增加大量失效的概率），
- 击穿： @Cacheable(value = "user",key = "'getUser'",sync = true) 增加sync = true，本地同步锁

## 使用
### 依赖
        <!--redis-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        <!--Spring Cache，使用注解简化开发-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-cache</artifactId>
        </dependency>
        
### 配置

    spring:
      redis:
        host: 192.168.171.132
        port: 6379
        password: 123456
        lettuce:
          pool:
            max-active: 10
            max-idle: 10
            min-idle: 1
            time-between-eviction-runs: 10s
      cache:
        type: redis # 使用redis作为缓存
        redis:
          time-to-live: 3600s # 过期时间
          #如果指定了前缀就用我们指定的前缀CACHE_，没有默认就用缓存的名字作为前缀，会导致自己在@Cacheable里设置的名字失效，所以这里不指定
          key-prefix: CACHE_ 
          # key值加前缀
          use-key-prefix: true
          #是否缓存空值，防止缓存穿透
          cache-null-values: true
### 添加配置类

要求springboot版本2.3以后。修改版本

     <parent>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-parent</artifactId>
            <version>2.3.2.RELEASE</version>
            <relativePath/> <!-- lookup parent from repository -->
        </parent>
        
      <properties>
              <java.version>1.8</java.version>
              <spring-cloud.version>Hoxton.SR9</spring-cloud.version>
              <com.alibaba.cloud>2.2.6.RELEASE</com.alibaba.cloud>
          </properties>
    
### 代码中使用
    //查询时    
    @Cacheable(value = "user",key = "'getUser'",sync = true)
    //删除
    @CacheEvict(value = "category", key = "'level1Categorys'")
    //新增
    @CachePut(value = "category", key = "'level1Categorys'")
    
