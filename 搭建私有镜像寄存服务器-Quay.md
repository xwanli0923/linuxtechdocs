---
title: 搭建私有镜像寄存服务器-Quay
created: '2022-05-31T14:28:30.646Z'
modified: '2022-05-31T15:32:43.401Z'
---

# 搭建私有镜像寄存服务器-Quay

---
> Operating System: Red Hat Enterprise Linux 8.6 (Ootpa)
> Kernel: Linux 4.18.0-372.9.1.el8.x86_64
> Architecture: x86-64
> Static hostname: registry.lab.example.com
> CPU: 2 vCPU
> Meomory: 8GiB
> 可以访问互联网,且有OpenShift Container Platform订阅
> IP Address: 192.168.100.105/24


# 准备工作
1. 创建一个可以免密码`sudo`的用户
```bash
# useradd contsvc
# echo <password> | passwd --stdin contsvc
# echo "contsvc ALL=(ALL)  NOPASSWD: ALL" > /etc/sudoers.d/contsvc
```
2. 使用`contsvc`用户登录系统，并安装所需软件
```bash
$ sudo yum install vim jq 
$ sudo yum module install container-tools
```
3. 登录 [OpenShift控制台Downloads页面下载](https://console.redhat.com/openshift/downloads#tool-mirror-registry) 下载所需的软件包：
- `mirror-registry.tar.gz`
- `openshift-client-linux.tar.gz`
> 下载的位置空间至少`10GiB`可用空间
```bash
$ ls -lh mirror-registry.tar.gz openshift-client-linux.tar.gz
-rw-rw-r--. 1 contsvc contsvc 633M May 31 21:55 mirror-registry.tar.gz
-rw-rw-r--. 1 contsvc contsvc  48M May 31 22:09 openshift-client-linux.tar.gz
```
4. 设置私有`registry`DNS记录
> 本环境使用了 `/etc/hosts`主机记录
```bash
$ sudo vim /etc/hosts
192.168.100.105 registry.lab.example.com registry
```
5. 下载个人账户下的`pull-secret`并转为`JSON`格式
```bash
$ cat pull-secret | jq . >  pull_secret.json
```
6. 按照`pull_secret.json`格式，输入私有`registry`的管理员账户和密码信息(需要和后面安装时的对应)
```bash
$ echo -n 'admin:RedHat321' | base64 -w0;echo
YWRtaW46UmVkSGF0MzIx
```
```json
{
  "auths": {
    "cloud.openshift.com": {
      "auth": "b3Blb...Vg==",
      "email": "natasha@demo.com"
    },
    "quay.io": {
      "auth": "b3Blb...Vg==",
      "email": "natasha@demo.com"
    },
    "registry.connect.redhat.com": {
      "auth": "fHWo...Yw==",
      "email": "natasha@demo.com"
    },
    "registry.redhat.io": {
      "auth": "fHWo...Yw==",
      "email": "natasha@demo"
    },
    "registry.lab.example.com": {
      "auth": "YWRtaW46UmVkSGF0MzIx",
      "email": "admin@lab.example.com"
    }
  }
}
```
7. 验证`podman`版本
> Podman 版本至少为 3.3
```bash
$ podman --version
podman version 4.0.2
```

## 为您的`registry`提供`SSL`保护(可选)
1. 生成根CA私钥
```bash
$ openssl genrsa -out rootCA.key 2048
```
2. 生成根CA证书
```bash
$ openssl req -x509 -new -nodes \
-key rootCA.key \
-sha256 \
-days 3650 \
-out rootCA.pem \
-subj "/C=CN/ST=ZJ/L=HZ/O=Red Hat Certification Training/OU=IT Department/CN=*.lab.example.com"
```
3.创建服务器私钥
```bash
$ openssl genrsa -out ssl.key 2048
```
4.生成证书请求
```bash
$ openssl req -new \
-key ssl.key \
-out ssl.csr \
-subj "/C=CN/ST=ZJ/L=HZ/O=Red Hat Certification Training/OU=IT Department/CN=registry.lab.example.com"
```
5. 创建一个`openssl.cnf`
```ini
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = registry.lab.example.com
IP.1 = 192.168.100.105
```
6.使用创建的`openssl.cnf`生成证书
```bash
$ openssl x509 -req \
-in ssl.csr \
-CA rootCA.pem \
-CAkey rootCA.key \
-CAcreateserial \
-out ssl.cert \
-days 3560 \
-extensions v3_req \
-extfile openssl.cnf
```

## 使用`mirror-registry`工具安装私有`registry`
1. 执行期间设置初始账户、初始账户密码、SSL证书私钥以及存储位置
```bash
$ sudo ./mirror-registry install \
--initUser admin \
--initPassword RedHat321 \
--quayHostname registry.lab.example.com \
--quayRoot /var/lib/registry-volume \
--sslCert ssl.cert \
--sslKey ssl.key  -v


   __   __
  /  \ /  \     ______   _    _     __   __   __
 / /\ / /\ \   /  __  \ | |  | |   /  \  \ \ / /
/ /  / /  \ \  | |  | | | |  | |  / /\ \  \   /
\ \  \ \  / /  | |__| | | |__| | / ____ \  | |
 \ \/ \ \/ /   \_  ___/  \____/ /_/    \_\ |_|
  \__/ \__/      \ \__
                  \___\ by Red Hat
 Build, Store, and Distribute your Containers

INFO[2022-05-31 23:20:34] Install has begun
DEBU[2022-05-31 23:20:34] Ansible Execution Environment Image: quay.io/quay/mirror-registry-ee:latest
DEBU[2022-05-31 23:20:34] Pause Image: registry.access.redhat.com/ubi8/pause:latest
DEBU[2022-05-31 23:20:34] Quay Image: registry.redhat.io/quay/quay-rhel8:v3.6.4
.....
PLAY RECAP *************************************************************************************************************
root@registry.lab.example.com : ok=40   changed=22   unreachable=0    failed=0    skipped=19   rescued=0    ignored=0

INFO[2022-05-31 23:23:45] Quay installed successfully, permanent data is stored in /var/lib/registry-volume
INFO[2022-05-31 23:23:45] Quay is available at https://registry.lab.example.com:8443 with credentials (admin, RedHat321)
```
> 卸载
> ```bash
> $ sudo ./mirror-registry uninstall -v \ 
>   --quayRoot /var/lib/registry-volume 
> ```

2. 配置`podman`信任CA
```bash
$ sudo mkdir /etc/containers/certs.d/registry.lab.example.com
$ sudo cp rootCA.pem /etc/containers/certs.d/registry.lab.example.com/ca.crt
```
3.配置系统信任CA
```bash
$ sudo cp rootCA.pem /etc/pki/ca-trust/source/anchors/
$ sudo update-ca-trust extract
```

## 测试
1. 使用 `podman` 登录`registry.lab.example.com`
```bash
$ podman login registry.lab.example.com:8443
Username: admin
Password: [RedHat321]
Login Succeeded!
```

