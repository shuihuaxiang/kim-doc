# Apache使用
## Apache反向代理
> 一、修改 httpd.conf 文件中：

      LoadModule proxy_module modules/mod_proxy.so
      LoadModule proxy_connect_module modules/mod_proxy_connect.so
      LoadModule proxy_http_module modules/mod_proxy_http.so
      LoadModule proxy_ftp_module modules/mod_proxy_ftp.so
      
  
> 二、修改安装目录（D:\HwsApacheMaster\Apache2.4\conf\extra）下的 httpd-vhosts.conf
  
      <VirtualHost *:80>
          ServerName pay.ds-czzl.com
          ProxyPass / http://localhost:8080/
          ProxyPassReverse / http://localhost:8080/
      </VirtualHost>  
        