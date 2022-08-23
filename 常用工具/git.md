# git
## git安装与配置

## 日常命令
> 拉取项目
    
     1.git init 
     2.git remote add origin https://github.com/shuihuaxiang/kim-doc.git
     3.git pull origin master![](images/git/8bf83846.png)
     

***
## 异常记录
>  1. idea  push的时候遇到问题：unable to access 'https://github.com/shuihuaxiang/kim-doc.git/':
OpenSSL SSL_read: SSL_ERROR_SYSCALL, errno 10054
    
    解决方案：
         原因：  
            自2021年8月12日后，github不在使用以前的账号密码git push，所以需要申请token  
         处理：  
            1.点击idea中的terminal  
            2.git config --global --unset http.proxy 和 git config --global --unset https.proxy  
            3.重新关联自己的git：git remote add origin git (自己的git),  
               git remote add origin https://github.com/shuihuaxiang/kim-doc.git  
            4.git push -u origin master重新输入账号密码  
             