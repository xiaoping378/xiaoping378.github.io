---
tags: ["linux"]
title: "Centos7 配置记录"
linkTitle: "Centos7 配置记录"
date: 2017-01-05
description: >
  记录centos7的日常操作记录
---

* **sshd** 禁用DNS反向解析和GSSAPI认证，以完成快速登陆

```bash
sudo sed -i 's/#UseDNS yes/UseDNS no/g' /etc/ssh/sshd_config
sudo sed -i 's/GSSAPIAuthentication yes/GSSAPIAuthentication no/g' /etc/ssh/sshd_config
sudo systemctl restart sshd
```

* **添加本地dvd源**

```bash
mkdir -p /media/dvd && mount -t auto /dev/cdrom /media/dvd

cat <<EOF >/etc/yum.repos.d/CentOS-Base.repo
[c7-dvd]
name=Centos-7
baseurl=file:///media/dvd
enabled=1
gpgcheck=1
gpgkey=file:///media/dvd/RPM-GPG-KEY-CentOS-7
EOF

yum clean all
```

* **安装docker**

  1. 添加yum源

```bash
sudo tee /etc/yum.repos.d/docker.repo <<-'EOF'
[dockerrepo]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/main/centos/7/
enabled=1
gpgcheck=1
gpgkey=https://yum.dockerproject.org/gpg
EOF
```

  2. 安装docker-engine

```bash
sudo yum install -y docker
```

  或者下载离线rpm包（供网络环境差的环境使用）

```bash
  sudo yum install docker --downloadonly --downloaddir=./
  sudo yum localinstall ./*.rpm
```

  ~~3. 修改必要的docker daemon配置参数~~ 现在不需要了，主要是devicemapper需要改下

```
sudo mkdir /etc/systemd/system/docker.service.d
sudo cat <<EOF >>/etc/systemd/system/docker.service.d/docker.conf
[Service]
    ExecStart=
    ExecStart=/usr/bin/docker daemon -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock --dns 180.76.76.76  --insecure-registry registry.cecf.com -g /home/Docker/docker -s overlay --selinux-enabled=false
EOF
```

   4. 启动docker

```bash
  sudo systemctl enable docker
  sudo systemctl start docker
  # 普通用户放进docker组里，快速CLI
  sudo usermod -aG docker [your_username]
```

* **firewalld**

  * 目前docker与firewalld存在兼容性问题
  * 先选择关闭firewalld吧

* **安装ntp**

  ```bash
  yum install -y ntp
  systemctl start ntpd
  systemctl enable ntpd
  ntpdate -u cn.pool.ntp.org
  ```

* **必要组件**

  ```bash
  yum install net-tools bind-utils tcpdump lsof
  ```

* **查询本机的外网IP**

  ```bash
  curl ipinfo.io
  ```