# 集成sentinel



## windows安装

版本：1.8.5

下载下来，运行jar

java -Dserver.port=8088 -Dcsp.sentinel.dashboard.server=localhost:8088 -Dproject.name=sentinel-dashboard -jar sentinel-dashboard-1.8.5.jar


默认用户名和密码都是 sentinel

## docker安装

版本：latest

    docker pull bladex/sentinel-dashboard
    
    docker run --name sentinel  -d -p 8858:8858 -d bladex/sentinel-dashboard
    
    docker update --restart=always sentinel
    
    
## 访问Sentinel监控平台

路径：http://192.168.171.132:8858/

账户：sentinel

密码：sentinel

## application.yml
    
    spring:
      cloud:  
         sentinel:
              log:
               # logs日志地址修改一下
                dir: D:/work/soft/online-match-logs/sentienl
              transport:
                # 应用开启端口，接收dashboard限流规则，如果被占用会默认+1
                port: 8719
                 # 控制台ip:port
                dashboard: 192.168.171.132:8858  