# Centos7 配置记录

* **sshd** 禁用DNS反向解析和GSSAPI认证，以完成快速登陆
  ```
  sudo sed -i 's/#UseDNS yes/UseDNS no/g' /etc/ssh/sshd_config
  sudo sed -i 's/GSSAPIAuthentication yes/GSSAPIAuthentication no/g' /etc/ssh/sshd_config
  sudo systemctl restart sshd
  ```

* **安装docker**

  添加yum源
  ```
sudo tee /etc/yum.repos.d/docker.repo <<-'EOF'
[dockerrepo]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/main/centos/7/
enabled=1
gpgcheck=1
gpgkey=https://yum.dockerproject.org/gpg
EOF
  ```
  安装docker-engine
  ```
  sudo yum install docker-engine
  ```

  或者下载离线rpm包（供网络环境差的环境使用）
  ```
  sudo yum install docker-engine --downloadonly --downloaddir=./
  sudo rpm -ivh ./*.rpm
  ```
  修改必要的docker daemon配置参数
  ```
sudo mkdir /etc/systemd/system/docker.service.d
sudo cat <<EOF >>/etc/systemd/system/docker.service.d/docker.conf
[Service]
    ExecStart=
    ExecStart=/usr/bin/docker daemon -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock --dns 180.76.76.76  --insecure-registry registry.cecf.com -g /home/Docker/docker
EOF
  ```

  启动docker
  ```
  sudo systemctl enable docker
  sudo systemctl start docker
  # 普通用户放进docker组里，快速CLI
  sudo usermod -aG docker [your_username]
  ```

* **firewalld**

  * 目前docker与firewalld存在兼容性问题
  * 先选择关闭firewalld吧


* **安装ntp**

  ```
  yum install -y ntp
  systemctl start ntpd
  systemctl enable ntpd
  ntpdate -u cn.pool.ntp.org
  ```
