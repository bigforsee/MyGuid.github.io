
获取Harbor安装包 https://github.com/goharbor/harbor/releases

    wget https://github.com/goharbor/harbor/releases/download/v2.1.2/harbor-offline-installer-v2.1.2.tgz
    
    yum -y install lrzsz

下载的docker-compose文件

    curl -L https://github.com/docker/compose/releases/download/1.23.2/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose

添加可执行权限

    chmod +x /usr/local/bin/docker-compose

解压压缩包 

    tar -xzvf harbor-offline-installer-v2.1.2.tgz

进入harbor目录修改配置文件名字

    cp harbor.yml.tmpl harbor.yml

编辑docker-compose，修改主机名和端口号

修改配置文件

    vi /etc/docker/daemon.json
    {
    "registry-mirrors": ["https://zydiol88.mirror.aliyuncs.com"],
    "insecure-registries": ["10.30.30.171:10880"]
    }
重启docker

    systemctl daemon-reload
    systemctl restart docker

启动harbor

    ./install.sh

启动成功后自动运行服务

    10.30.30.171:443

harbor服务命令
在harbor目录下执行 。  
启动

    docker-compose up -d

重启

    docker-compose restart

停止

    docker-compose down -v

登录

    docker login 10.30.30.171:10880

登出

    docker logout 10.30.30.171:10880

推送镜像

    docker tag mysql:5.7 10.30.30.171:10880/library/mysql:5.7
     docker tag fwoa:v2 10.30.30.171:10880/fwoa/fwoa:v2
    **/fwoa/fwoa:v2 这里是服务地址后面更项目及文件名 
    
    docker push 10.30.30.171:10880/library/mysql:5.7
    docker push 10.30.30.171:10880/fwoa/fwoa:v2
    **fwoa/fwoa:v2 解释同上

docker run -itd --mac-address 00:50:56:b5:69:7a -p 90:80 test:v1 
# 配置https访问
参考文档：
https://goharbor.io/docs/1.10/install-config/configure-https/
https://goharbor.io/docs/1.10/install-config/troubleshoot-installation/#https
https://www.cnblogs.com/cjwnb/p/13441071.html
## 1. 生成证书颁发机构证书
在生产环境中，您应该从CA获得证书。在测试或开发环境中，您可以生成自己的CA。要生成CA证书，请运行以下命令。
## 1.1 生成CA证书私钥。

    openssl genrsa -out ca.key 4096

## 1.2 生成CA证书

    调整-subj选项中的值以反映您的组织。如果使用FQDN连接Harbor主机，则必须将其指定为通用名称（CN）属性。
    openssl req -x509 -new -nodes -sha512 -days 3650 \
     -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=www.xianhai.online" \
     -key ca.key \
     -out ca.crt

如果是ip访问， 将 `harbor.od.com` 改成 ip地址

## 2. 生成服务器证书


证书通常包含一个.crt文件和一个.key文件
## 2.1 生成私钥
    openssl genrsa -out www.xianhai.online.key 4096  
 ## 2.2 生成证书签名请求（CSR）
    openssl req -sha512 -new \
        -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=www.xianhai.online" \
        -key www.xianhai.online.key \
        -out www.xianhai.online.csr
如果是ip访问， 将 `harbor.od.com` 改成 ip地址
## 2.3 生成一个x509 v3扩展文件
无论您使用FQDN还是IP地址连接到Harbor主机，都必须创建此文件，以便可以为您的Harbor主机生成符合主题备用名称（SAN）和x509 v3的证书扩展要求。替换DNS条目以反映您的域

    cat > v3.ext <<-EOF
    authorityKeyIdentifier=keyid,issuer
    basicConstraints=CA:FALSE
    keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
    extendedKeyUsage = serverAuth
    subjectAltName = @alt_names
    
    [alt_names]
    DNS.1=www.xianhai.online
    DNS.2=mynas
    DNS.3=hostname
    EOF
  如果是ip访问

    cat > v3.ext <<-EOF
    authorityKeyIdentifier=keyid,issuer
    basicConstraints=CA:FALSE
    keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
    extendedKeyUsage = serverAuth
    subjectAltName = IP:192.168.31.200
    EOF

 ##  2.4 使用该`v3.ext`文件为您的Harbor主机生成证书
    openssl x509 -req -sha512 -days 3650 \
        -extfile v3.ext \
        -CA ca.crt -CAkey ca.key -CAcreateserial \
        -in www.xianhai.online.csr \
        -out www.xianhai.online.crt
如果是ip访问， 将 `harbor.od.com` 改成 ip地址

## 3. 提供证书给Harbor和Docker
生成后`ca.crt`，`harbor.od.com.crt`和`harbor.od.com.key`文件，必须将它们提供给`Harbor`和`docker`，重新配置它们
## 3.1 将服务器证书和密钥复制到Harbor主机上的`/data/cert/`文件夹中

    mkdir -p cert/
    cp www.xianhai.online.crt /volume1/docker/harbor/cert/
    cp www.xianhai.online.key /volume1/docker/harbor/cert/
    mkdir -p harbor-log
    mkdir data
    mkdir - p common/config

## 3.2 转换`harbor.od.com.crt`为`harbor.od.com.cert`，供Docker使用
Docker守护程序将`.crt`文件解释为CA证书，并将`.cert`文件解释为客户端证书

    openssl x509 -inform PEM -in www.xianhai.online.crt -out www.xianhai.online.cert

## 3.3 将服务器证书，密钥和CA文件复制到Harbor主机上的Docker证书文件夹中。您必须首先创建适当的文件夹

    mkdir -p /etc/docker/certs.d/www.xianhai.online.com/
    cd /etc/docker/certs.d/www.xianhai.online.com
    cp /volume1/docker/harbor/www.xianhai.online.cert ./
    cp /volume1/docker/harbor/www.xianhai.online.key ./
    cp /volume1/docker/harbor/ca.crt ./

如果将默认`nginx`端口443 映射到其他端口，请创建文件夹`/etc/docker/certs.d/yourdomain.com:port`或`/etc/docker/certs.d/harbor_IP:port`

## 3.4 重新启动Docker Engine

    systemctl restart docker

## 3.5 证书的目录结构

    /etc/docker/certs.d/
    └── yourdomain.com:port
       ├── yourdomain.com.cert  <-- Server certificate signed by CA
       ├── yourdomain.com.key   <-- Server key signed by CA
       └── ca.crt               <-- Certificate authority that signed the registry certificate

## 4. 部署或重新配置Harbor
`harbor.yml`

    # Configuration file of Harbor
    
    # The IP address or hostname to access admin UI and registry service.
    # DO NOT use localhost or 127.0.0.1, because Harbor needs to be accessed by external clients.
    hostname: www.xianhai.online
    
    # http related config
    http:
      # port for http, default is 80. If https enabled, this port will redirect to https port
      port: 10881
    
    # https related config
    https:
      # https port for harbor, default is 443
      port: 10880
      # The path of cert and key files for nginx
      certificate: /volume1/docker/harbor/www.xianhai.online.cert
      private_key: /volume1/docker/harbor/www.xianhai.online.key
    
    # # Uncomment following will enable tls communication between all harbor components
    # internal_tls:
    #   # set enabled to true means internal tls is enabled
    #   enabled: true
    #   # put your cert and key files on dir
    #   dir: /etc/harbor/tls/internal
    
    # Uncomment external_url if you want to enable external proxy
    # And when it enabled the hostname will no longer used
    # external_url: https://reg.mydomain.com:8433
    
    # The initial password of Harbor admin
    # It only works in first time to install harbor
    # Remember Change the admin password from UI after launching Harbor.
    harbor_admin_password: Harbor12345
    
    # Harbor DB configuration
    database:
      # The password for the root user of Harbor DB. Change this before any production use.
      password: root123
      # The maximum number of connections in the idle connection pool. If it <=0, no idle connections are retained.
      max_idle_conns: 100
      # The maximum number of open connections to the database. If it <= 0, then there is no limit on the number of open connections.
      # Note: the default number of connections is 1024 for postgres of harbor.
      max_open_conns: 900
      # The maximum amount of time a connection may be reused. Expired connections may be closed lazily before reuse. If it <= 0, connections are not closed due to a connection's age.
      # The value is a duration string. A duration string is a possibly signed sequence of decimal numbers, each with optional fraction and a unit suffix, such as "300ms", "-1.5h" or "2h45m". Valid time units are "ns", "us" (or "µs"), "ms", "s", "m", "h".
      conn_max_lifetime: 5m
      # The maximum amount of time a connection may be idle. Expired connections may be closed lazily before reuse. If it <= 0, connections are not closed due to a connection's idle time.
      # The value is a duration string. A duration string is a possibly signed sequence of decimal numbers, each with optional fraction and a unit suffix, such as "300ms", "-1.5h" or "2h45m". Valid time units are "ns", "us" (or "µs"), "ms", "s", "m", "h".
      conn_max_idle_time: 0
    
    # The default data volume
    data_volume: /volume1/docker/harbor/data
    
    # Harbor Storage settings by default is using /data dir on local filesystem
    # Uncomment storage_service setting If you want to using external storage
    # storage_service:
    #   # ca_bundle is the path to the custom root ca certificate, which will be injected into the truststore
    #   # of registry's containers.  This is usually needed when the user hosts a internal storage with self signed certificate.
    #   ca_bundle:
    
    #   # storage backend, default is filesystem, options include filesystem, azure, gcs, s3, swift and oss
    #   # for more info about this configuration please refer https://docs.docker.com/registry/configuration/
    #   filesystem:
    #     maxthreads: 100
    #   # set disable to true when you want to disable registry redirect
    #   redirect:
    #     disable: false
    
    # Trivy configuration
    #
    # Trivy DB contains vulnerability information from NVD, Red Hat, and many other upstream vulnerability databases.
    # It is downloaded by Trivy from the GitHub release page https://github.com/aquasecurity/trivy-db/releases and cached
    # in the local file system. In addition, the database contains the update timestamp so Trivy can detect whether it
    # should download a newer version from the Internet or use the cached one. Currently, the database is updated every
    # 12 hours and published as a new release to GitHub.
    trivy:
      # ignoreUnfixed The flag to display only fixed vulnerabilities
      ignore_unfixed: false
      # skipUpdate The flag to enable or disable Trivy DB downloads from GitHub
      #
      # You might want to enable this flag in test or CI/CD environments to avoid GitHub rate limiting issues.
      # If the flag is enabled you have to download the `trivy-offline.tar.gz` archive manually, extract `trivy.db` and
      # `metadata.json` files and mount them in the `/home/scanner/.cache/trivy/db` path.
      skip_update: false
      #
      # The offline_scan option prevents Trivy from sending API requests to identify dependencies.
      # Scanning JAR files and pom.xml may require Internet access for better detection, but this option tries to avoid it.
      # For example, the offline mode will not try to resolve transitive dependencies in pom.xml when the dependency doesn't
      # exist in the local repositories. It means a number of detected vulnerabilities might be fewer in offline mode.
      # It would work if all the dependencies are in local.
      # This option doesn't affect DB download. You need to specify "skip-update" as well as "offline-scan" in an air-gapped environment.
      offline_scan: false
      #
      # Comma-separated list of what security issues to detect. Possible values are `vuln`, `config` and `secret`. Defaults to `vuln`.
      security_check: vuln
      #
      # insecure The flag to skip verifying registry certificate
      insecure: false
      # github_token The GitHub access token to download Trivy DB
      #
      # Anonymous downloads from GitHub are subject to the limit of 60 requests per hour. Normally such rate limit is enough
      # for production operations. If, for any reason, it's not enough, you could increase the rate limit to 5000
      # requests per hour by specifying the GitHub access token. For more details on GitHub rate limiting please consult
      # https://developer.github.com/v3/#rate-limiting
      #
      # You can create a GitHub token by following the instructions in
      # https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line
      #
      # github_token: xxx
    
    jobservice:
      # Maximum number of job workers in job service
      max_job_workers: 10
      # The jobLogger sweeper duration (ignored if `jobLogger` is `stdout`)
      logger_sweeper_duration: 1 #days
    
    notification:
      # Maximum retry count for webhook job
      webhook_job_max_retry: 3
      # HTTP client timeout for webhook job
      webhook_job_http_client_timeout: 3 #seconds
    
    # Log configurations
    log:
      # options are debug, info, warning, error, fatal
      level: info
      # configs for logs in local storage
      local:
        # Log files are rotated log_rotate_count times before being removed. If count is 0, old versions are removed rather than rotated.
        rotate_count: 50
        # Log files are rotated only if they grow bigger than log_rotate_size bytes. If size is followed by k, the size is assumed to be in kilobytes.
        # If the M is used, the size is in megabytes, and if G is used, the size is in gigabytes. So size 100, size 100k, size 100M and size 100G
        # are all valid.
        rotate_size: 200M
        # The directory on your host that store log
        location: /volume1/docker/harbor/harbor-log
    
      # Uncomment following lines to enable external syslog endpoint.
      # external_endpoint:
      #   # protocol used to transmit log to external endpoint, options is tcp or udp
      #   protocol: tcp
      #   # The host of external endpoint
      #   host: localhost
      #   # Port of external endpoint
      #   port: 5140
    
    #This attribute is for migrator to detect the version of the .cfg file, DO NOT MODIFY!
    _version: 2.8.0
    
    # Uncomment external_database if using external database.
    # external_database:
    #   harbor:
    #     host: harbor_db_host
    #     port: harbor_db_port
    #     db_name: harbor_db_name
    #     username: harbor_db_username
    #     password: harbor_db_password
    #     ssl_mode: disable
    #     max_idle_conns: 2
    #     max_open_conns: 0
    #   notary_signer:
    #     host: notary_signer_db_host
    #     port: notary_signer_db_port
    #     db_name: notary_signer_db_name
    #     username: notary_signer_db_username
    #     password: notary_signer_db_password
    #     ssl_mode: disable
    #   notary_server:
    #     host: notary_server_db_host
    #     port: notary_server_db_port
    #     db_name: notary_server_db_name
    #     username: notary_server_db_username
    #     password: notary_server_db_password
    #     ssl_mode: disable
    
    # Uncomment external_redis if using external Redis server
    # external_redis:
    #   # support redis, redis+sentinel
    #   # host for redis: <host_redis>:<port_redis>
    #   # host for redis+sentinel:
    #   #  <host_sentinel1>:<port_sentinel1>,<host_sentinel2>:<port_sentinel2>,<host_sentinel3>:<port_sentinel3>
    #   host: redis:6379
    #   password: 
    #   # Redis AUTH command was extended in Redis 6, it is possible to use it in the two-arguments AUTH <username> <password> form.
    #   # username:
    #   # sentinel_master_set must be set to support redis+sentinel
    #   #sentinel_master_set:
    #   # db_index 0 is for core, it's unchangeable
    #   registry_db_index: 1
    #   jobservice_db_index: 2
    #   trivy_db_index: 5
    #   idle_timeout_seconds: 30
    
    # Uncomment uaa for trusting the certificate of uaa instance that is hosted via self-signed cert.
    # uaa:
    #   ca_file: /path/to/ca
    
    # Global proxy
    # Config http proxy for components, e.g. http://my.proxy.com:3128
    # Components doesn't need to connect to each others via http proxy.
    # Remove component from `components` array if want disable proxy
    # for it. If you want use proxy for replication, MUST enable proxy
    # for core and jobservice, and set `http_proxy` and `https_proxy`.
    # Add domain to the `no_proxy` field, when you want disable proxy
    # for some special registry.
    proxy:
      http_proxy:
      https_proxy:
      no_proxy:
      components:
        - core
        - jobservice
        - trivy
    
    # metric:
    #   enabled: false
    #   port: 9090
    #   path: /metrics
    
    # Trace related config
    # only can enable one trace provider(jaeger or otel) at the same time,
    # and when using jaeger as provider, can only enable it with agent mode or collector mode.
    # if using jaeger collector mode, uncomment endpoint and uncomment username, password if needed
    # if using jaeger agetn mode uncomment agent_host and agent_port
    # trace:
    #   enabled: true
    #   # set sample_rate to 1 if you wanna sampling 100% of trace data; set 0.5 if you wanna sampling 50% of trace data, and so forth
    #   sample_rate: 1
    #   # # namespace used to differenciate different harbor services
    #   # namespace:
    #   # # attributes is a key value dict contains user defined attributes used to initialize trace provider
    #   # attributes:
    #   #   application: harbor
    #   # # jaeger should be 1.26 or newer.
    #   # jaeger:
    #   #   endpoint: http://hostname:14268/api/traces
    #   #   username:
    #   #   password:
    #   #   agent_host: hostname
    #   #   # export trace data by jaeger.thrift in compact mode
    #   #   agent_port: 6831
    #   # otel:
    #   #   endpoint: hostname:4318
    #   #   url_path: /v1/traces
    #   #   compression: false
    #   #   insecure: true
    #   #   timeout: 10s
    
    # Enable purge _upload directories
    upload_purging:
      enabled: true
      # remove files in _upload directories which exist for a period of time, default is one week.
      age: 168h
      # the interval of the purge operations
      interval: 24h
      dryrun: false
    
    # Cache layer configurations
    # If this feature enabled, harbor will cache the resource
    # `project/project_metadata/repository/artifact/manifest` in the redis
    # which can especially help to improve the performance of high concurrent
    # manifest pulling.
    # NOTICE
    # If you are deploying Harbor in HA mode, make sure that all the harbor
    # instances have the same behaviour, all with caching enabled or disabled,
    # otherwise it can lead to potential data inconsistency.
    cache:
      # not enabled by default
      enabled: false
      # keep cache for one day by default
      expire_hours: 24

## 4.1 运行`prepare`脚本以启用HTTPS
Harbor将`nginx`实例用作所有服务的反向代理。您可以使用`prepare`脚本来配置`nginx`为使用HTTPS

    ./prepare

## 4.2 如果Harbor正在运行，请停止并删除现有实例

镜像数据保留在文件系统中，因此不会丢失任何数据

    docker-compose down -v

## 4.3 重启docker

    docker-compose up -d

## 5. 验证HTTPS连接
![](img\1.png)
![](img\2.png)
![](img\3.png)

## 注意
然后登陆推送镜像测试， 如果服务器要推送代码到harbor， 必须在docker的配置文件的目录 `/etc/docker/certs.d/harbor.od.com/` 配置服务器证书（`harbor.od.com.cert`），密钥（`harbor.od.com.key`）和CA文件（`ca.crt`）





```shell
DOCKER_USERNAME="admin"
DOCKER_PASSWORD="30Harbor@30"
```



















