# git
## git安装与配置

## 日常命令
> 拉取项目
    
     1.git init 
     2.git remote add origin https://github.com/shuihuaxiang/kim-doc.git
     3.git pull origin master

## 设置网络
1.打开DNS查询工具：http://tool.chinaz.com/dns  

2.选择TTL值最低的那个，如果当前列表没有符合要求的IP，重新检测。

![](images/c4299392.png)   
3.使用Win + R 组合键，在运行对话框里，复制并粘贴：C:\WINDOWS\system32\drivers\etc

4.打开以后，选择HOSTS文件。把刚才复制的IP地址，复制到这个文件里，格式如下：

![](images/6e201f1b.png)
  
***
## 异常记录
>  1. idea  push的时候遇到问题：unable to access 'https://github.com/shuihuaxiang/kim-doc.git/':
OpenSSL SSL_read: SSL_ERROR_SYSCALL, errno 10054
    
    解决方案1：
         原因：  
            自2021年8月12日后，github不在使用以前的账号密码git push，所以需要申请token  
         处理：  
            1.点击idea中的terminal  
            2.git config --global --unset http.proxy 和 git config --global --unset https.proxy  
            3.重新关联自己的git：git remote add origin git (自己的git),  
               git remote add origin https://github.com/shuihuaxiang/kim-doc.git  
            4.git push -u origin master重新输入账号密码  
       
     方案2：  
       因为服务器的SSL证书没有经过第三方机构的签署
        git config --global http.sslVerify "false"
       解除ssl验证后，再次git即可
     方案3：
     取消代理
      git config --global --unset http.proxy 
      
      git config --global --unset https.proxy