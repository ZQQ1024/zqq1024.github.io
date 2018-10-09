---
title: "安装Chrome"
---

1. Add Public Key:
   ```
   $ wget -q -O - https://dl-ssl.google.com/linux/linux_signing_key.pub | sudo apt-key add -
   ```
2. Set repository:
   ```
   $ echo 'deb [arch=amd64] http://dl.google.com/linux/chrome/deb/ stable main' | sudo tee /etc/apt/sources.list.d/google-chrome.list
   ```
3. Install Package:
   ```
   $ sudo apt-get update 
   $ sudo apt-get install google-chrome-stable
   ```

**过程**：指定3rd的Repo地址，还有有公钥，用于验证Repo和Package的签名

> 参考地址：[https://askubuntu.com/questions/510056/how-to-install-google-chrome](https://askubuntu.com/questions/510056/how-to-install-google-chrome)