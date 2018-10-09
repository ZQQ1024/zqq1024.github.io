---
title: "安装Docker"
---

1. Uninstall old versions:
   ```
   $ sudo apt-get remove docker docker-engine docker.io
   ```
2. Update the `apt` package index:
   ```
   $ sudo apt-get update
   ```
3. Install packages to allow `apt` to use a repository over HTTPS:
   ```
   $ sudo apt-get install \
     apt-transport-https \
     ca-certificates \
     curl \
     software-properties-common
   ```
4. Add Docker’s official GPG key:
   ```
   $ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
   ```
   Verify that you now have the key with the fingerprint `9DC8 5822 9FC7 DD38 854A E2D8 8D81 803C 0EBF CD88`, by searching for the last 8 characters of the fingerprint:
   ```
   $ sudo apt-key fingerprint 0EBFCD88
   xxx
   ```
5. set up the `stable` repository:
   ```
   $ sudo add-apt-repository \
   "deb [arch=s390x] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
   ```
6. Update the `apt` package index, and Install the _latest_ version of Docker CE
   ```
   $ sudo apt-get update
   $ sudo apt-get install docker-ce
   ```
7. Use Docker as a non-root user:
   ```
   $ sudo usermod -aG docker <your-user>
   ```
   substitude`<your-user>` and log out.
8. Download the latest version of Docker Compose, and apply executable permissions:
   ```
   $ sudo curl -L "https://github.com/docker/compose/releases/download/1.22.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
   $ sudo chmod +x /usr/local/bin/docker-compose
   ```
只适用Ubuntu amd64

> 参考地址：  
> [https://docs.docker.com/install/linux/docker-ce/ubuntu/](https://docs.docker.com/install/linux/docker-ce/ubuntu/)  
> [https://docs.docker.com/compose/install/](https://docs.docker.com/compose/install/)