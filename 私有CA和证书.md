# 构建私有的 CA 机构(拓展)

=========================================================

CA中心申请证书的流程:

过程：							
1、web服务器，生成一对非对称加密密钥（web公钥，web私钥）
2、web服务器使用 web私钥生成 web服务器的证书请求文件，并将证书请求发给CA服务器
3、CA服务器使用 CA的私钥 对 web 服务器的证书请求 进行数字签名得到 web服务器的数字证书，并将web服务器的数字证书颁发给web服务器。



## 1、CA 介绍

CA（Certificate Authority）证书颁发机构主要负责证书的颁发、管理以及归档和吊销。证书内包含了拥有证书者的姓名、地址、电子邮件帐号、公钥、证书有效期、发放证书的CA、CA的数字签名等信息。**证书主要有三大功能：加密、签名、身份验证。**

## 2、构建私有 CA

##### 1、检查安装 openssl

```shell
[root@https-ca ~]# rpm -qa openssl
```

![img](assets/%E7%A7%81%E6%9C%89CA%E5%92%8C%E8%AF%81%E4%B9%A6/1653033380509-c247de98-2fe7-4a32-aeb5-b8546c9d147a.png)



如果未安装

```shell
[root@https-ca ~]# yum install openssl openssl-devel -y
```



##### 2、查看配置文件

openssl 配置`/etc/pki/tls/openssl.cnf`有关CA的配置。如果服务器为证书签署者的身份那么就会用到此配置文件，此配置文件对于证书申请者是无作用的。



```shell
[root@https-ca ~]# vim /etc/pki/tls/openssl.cnf
####################################################################
[ ca ]
default_ca      = CA_default            # 默认的CA配置；CA_default指向下面配置块

####################################################################
[ CA_default ]

dir             = /etc/pki/CA           # CA的默认工作目录
certs           = $dir/certs            # 认证证书的目录
crl_dir         = $dir/crl              # 证书吊销列表的路径
database        = $dir/index.txt        # 数据库的索引文件


new_certs_dir   = $dir/newcerts         # 新颁发证书的默认路径

certificate     = $dir/cacert.pem       # 此服务认证证书，如果此服务器为根CA那么这里为自颁发证书
serial          = $dir/serial           # 下一个证书的证书编号
crlnumber       = $dir/crlnumber        # 下一个吊销的证书编号
                                        
crl             = $dir/crl.pem          # The current CRL
private_key     = $dir/private/cakey.pem# CA的私钥
RANDFILE        = $dir/private/.rand    # 随机数文件

x509_extensions = usr_cert              # The extentions to add to the cert

name_opt        = ca_default            # 命名方式，以ca_default定义为准
cert_opt        = ca_default            # 证书参数，以ca_default定义为准


default_days    = 365                   # 证书默认有效期
default_crl_days= 30                    # CRl的有效期
default_md      = sha256                # 加密算法
preserve        = no                    # keep passed DN ordering


policy          = policy_match          #policy_match策略生效

# For the CA policy
[ policy_match ]
countryName             = match         #国家；match表示申请者的申请信息必须与此一致
stateOrProvinceName     = match         #州、省
organizationName        = match         #组织名、公司名
organizationalUnitName  = optional      #部门名称；optional表示申请者可以的信息与此可以不一致
commonName              = supplied
emailAddress            = optional

# For the 'anything' policy
# At this point in time, you must list all acceptable 'object'
# types.
[ policy_anything ]                     #由于定义了policy_match策略生效，所以此策略暂未生效
countryName             = optional
stateOrProvinceName     = optional
localityName            = optional
organizationName        = optional
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional
```



##### 3、根证书服务器目录



根CA服务器：因为只有 CA 服务器的角色，所以用到的目录只有/etc/pki/CA



网站服务器：只是证书申请者的角色，所以用到的目录只有`/etc/pki/tls`



**4、创建所需要的文件**



```shell
[root@https-ca ~]# cd /etc/pki/CA/
[root@https-ca CA]# ls
certs  crl  newcerts  private
[root@https-ca CA]# touch index.txt   #创建生成证书索引数据库文件
[root@https-ca CA]# ls
certs  crl  index.txt  newcerts  private
[root@https-ca CA]# echo 01 > serial   #指定第一个颁发证书的序列号
[root@https-ca CA]# ls
certs  crl  index.txt  newcerts  private  serial
[root@https-ca CA]#
```



##### 4、创建密钥



在根CA服务器上创建密钥，密钥的位置必须为`/etc/pki/CA/private/cakey.pem`，这个是openssl.cnf中中指定的路径，只要与配置文件中指定的匹配即可。



```shell
[root@https-ca CA]# (umask 066; openssl genrsa -out private/cakey.pem 2048)
Generating RSA private key, 2048 bit long modulus
...........+++
...............+++	
e is 65537 (0x10001)
```



##### 5、生成自签名证书



根CA自签名证书，根CA是最顶级的认证机构，没有人能够认证他，所以只能自己认证自己生成自签名证书。



```shell
[root@https-ca CA]# openssl req -new -x509 -key /etc/pki/CA/private/cakey.pem -days 7300 -out /etc/pki/CA/cacert.pem -days 7300
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:BEIJING
Locality Name (eg, city) [Default City]:BEIJING
Organization Name (eg, company) [Default Company Ltd]:CA
Organizational Unit Name (eg, section) []:OPT
Common Name (eg, your name or your server's hostname) []:ca.asuka.com
Email Address []:
[root@https-ca CA]# ls
cacert.pem  certs  crl  index.txt  newcerts  private  serial
```



```shell
-new: 	生成新证书签署请求
-x509: 	专用于CA生成自签证书
-key: 	生成请求时用到的私钥文件
-days ：	证书的有效期限
-out /PATH/TO/SOMECERTFILE: 	证书的保存路径
```



##### 6、下载安装证书



`/etc/pki/CA/cacert.pem`就是生成的自签名证书文件，使用 `SZ/xftp`工具将他导出到窗口机器中。然后双击安装此证书到受信任的根证书颁发机构



```shell
[root@https-ca CA]# yum install -y lrzsz
[root@https-ca CA]# sz cacert.pem
```



![img](assets/%E7%A7%81%E6%9C%89CA%E5%92%8C%E8%AF%81%E4%B9%A6/1653033403477-4d7f60f8-524b-48f3-bf7a-7a86577709ba.png)



![img](assets/%E7%A7%81%E6%9C%89CA%E5%92%8C%E8%AF%81%E4%B9%A6/1653033418697-cbe02ba9-ab71-496b-90c7-b4cc258abdb6.png)



![img](assets/%E7%A7%81%E6%9C%89CA%E5%92%8C%E8%AF%81%E4%B9%A6/1653033439265-84251f08-a099-42b9-86d8-4b4fb0a6005a.png)



![img](assets/%E7%A7%81%E6%9C%89CA%E5%92%8C%E8%AF%81%E4%B9%A6/1653033450458-86103101-4491-40d5-987d-f80dfc9d4cd4.png)



![img](assets/%E7%A7%81%E6%9C%89CA%E5%92%8C%E8%AF%81%E4%B9%A6/1582688603239.png)



![img](assets/%E7%A7%81%E6%9C%89CA%E5%92%8C%E8%AF%81%E4%B9%A6/1582688623883.png)



![img](assets/%E7%A7%81%E6%9C%89CA%E5%92%8C%E8%AF%81%E4%B9%A6/1653033467785-f8135664-1f71-4ea6-8863-3c93182cc635.png)



![img](assets/%E7%A7%81%E6%9C%89CA%E5%92%8C%E8%AF%81%E4%B9%A6/1653033475303-a41a6406-4131-4f9e-9142-fabe73eee7ba.png)



##### **7、客户端（web服务器）CA 证书申请及签名**



**1、检查安装 openssl**



```shell
[root@nginx-server ~]# rpm -qa openssl
```



如果未安装，安装 openssl



```shell
[root@nginx-server ~]# yum install openssl openssl-devel  -y
```



**2、客户端生成私钥文件**



```shell
[root@nginx-server ~]# (umask 066; openssl genrsa -out /etc/pki/tls/private/www.asuka.com.key 2048)
Generating RSA private key, 2048 bit long modulus
..............................+++
..........+++
e is 65537 (0x10001)
[root@nginx-server ~]# cd /etc/pki/tls/private/
[root@nginx-server private]# ls
www.asuka.com.key
[root@nginx-server private]#
```



**8、客户端用私钥加密生成证书请求**



```shell
[root@nginx-server private]# ls ../
cert.pem  certs  misc  openssl.cnf  private
[root@nginx-server private]# openssl req -new -key /etc/pki/tls/private/www.asuka.com.key -days 365 -out /etc/pki/tls/www.asuka.com.csr
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:BEIJING
Locality Name (eg, city) [Default City]:BEIJING
Organization Name (eg, company) [Default Company Ltd]:asuka
Organizational Unit Name (eg, section) []:OPT
Common Name (eg, your name or your server's hostname) []:www.asuka.com
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
[root@nginx-server private]# ls ../
cert.pem  certs  misc  openssl.cnf  private  www.asuka.com.csr
[root@nginx-server private]#
```



CSR(Certificate Signing Request)包含了公钥和名字信息。通常以.csr为后缀，是网站向CA发起认证请求的文件，是中间文件。



在这一命令执行的过程中，系统会要求填写如下信息：



![img](assets/%E7%A7%81%E6%9C%89CA%E5%92%8C%E8%AF%81%E4%B9%A6/1562059278073.png)



最后把生成的请求文件（`/etc/pki/tls/www.asuka.com.csr`）传输给CA ,这里我使用scp命令，通过ssh协议，将该文件传输到CA下的`/etc/pki/CA/private/`目录



```shell
[root@nginx-server private]# cd ../
[root@nginx-server tls]# scp www.asuka.com.csr 192.168.62.163:/etc/pki/CA/private
```



**4、CA 签署证书**（在ca服务器上面操作）



使用/etc/pki/tls/openssl.cnf，修改organizationName=supplied



修改 /etc/pki/tls/openssl.cnf



```shell
[root@https-ca ~]# vim /etc/pki/tls/openssl.cnf
policy          = policy_match
 82 
 83 # For the CA policy
 84 [ policy_match ]
 85 countryName             = match
 86 stateOrProvinceName     = match
 87 organizationName        = supplied
 88 organizationalUnitName  = optional
 89 commonName              = supplied
 90 emailAddress            = optional
```



CA 签署证书



```shell
[root@https-ca ~]# openssl ca -in /etc/pki/CA/private/www.asuka.com.csr -out /etc/pki/CA/certs/www.asuka.com.crt -days 365
Using configuration from /etc/pki/tls/openssl.cnf
Check that the request matches the signature
Signature ok
Certificate Details:
        Serial Number: 1 (0x1)
        Validity
            Not Before: Jul  3 10:12:23 2019 GMT
            Not After : Jul  2 10:12:23 2020 GMT
        Subject:
            countryName               = CN
            stateOrProvinceName       = BEIJING
            organizationName          = asuka
            organizationalUnitName    = OPT
            commonName                = www.asuka.com
        X509v3 extensions:
            X509v3 Basic Constraints: 
                CA:FALSE
            Netscape Comment: 
                OpenSSL Generated Certificate
            X509v3 Subject Key Identifier: 
                E3:AC:1A:55:2B:28:B9:80:DC:9C:C2:13:70:53:27:AD:3D:44:8F:D3
            X509v3 Authority Key Identifier: 
                keyid:5D:2A:81:B2:E7:8D:D8:88:E5:7B:94:CA:75:65:9C:82:2B:A9:B2:3C

Certificate is to be certified until Jul  2 10:12:23 2020 GMT (365 days)
Sign the certificate? [y/n]:y


1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Data Base Updated
```



证书通常以.crt为后缀，表示证书文件



**2、查看生成的证书的信息**



```shell
[root@https-ca ~]# openssl x509 -in /etc/pki/CA/certs/www.asuka.com.crt -noout -subject
subject= /C=CN/ST=BEIJING/O=asuka/OU=OPT/CN=www.asuka.com
```



**3、将生成的证书发放给请求客户端(web服务端)**



```shell
[root@https-ca ~]# cd /etc/pki/CA/certs/
[root@https-ca certs]# scp www.asuka.com.ctr 192.168.62.162:/etc/pki/CA/certs/
```



测试:



```shell
nginx-server(充当服务端)：
[root@nginx-server ~]# cd /etc/pki/CA/certs/
[root@nginx-server certs]# ls
www.asuka.com.crt
[root@nginx-server certs]# find / -name *.key
/etc/pki/tls/private/www.asuka.com.key
/usr/share/doc/openssh-7.4p1/PROTOCOL.key

还是在这台机器安装nginx并且配置证书:
root@nginx-server conf.d]# pwd
/etc/nginx/conf.d
[root@nginx-server conf.d]# vim nginx.conf
server {
        listen       443 ssl;
        server_name  www.asuka.com;

        ssl_certificate         /etc/pki/CA/certs/www.asuka.com.crt;  #指定证书路径
        ssl_certificate_key  /etc/pki/tls/private/www.asuka.com.key;  #指定私钥路径
        ssl_session_timeout  5m;   #配置用于SSL会话的缓存
        ssl_protocols  SSLv2 SSLv3 TLSv1;   #指定使用的协议
        ssl_ciphers  ALL:!ADH:!EXPORT56:RC4+RSA:+HIGH:+MEDIUM:+LOW:+SSLv2:+EXP; # //密码指定为OpenSSL支持的格式
        ssl_prefer_server_ciphers   on;  #设置协商加密算法时，优先使用服务端的加密，而不是客户端浏览器的。

        location / {
                root /usr/share/nginx/html;
                index index.html index.htm;
                }
}
保存重启
[root@nginx-server conf.d]# nginx -t 
[root@nginx-server conf.d]# nginx -s reload
```



windows首先配置hosts文件解析



浏览器测试访问:



![img](assets/%E7%A7%81%E6%9C%89CA%E5%92%8C%E8%AF%81%E4%B9%A6/1586852136226.png)