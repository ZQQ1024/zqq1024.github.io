---
title: "安装Repo"
classes: wide
toc: false
---

1. create a `bin/` directory in your home directoryand that it is included in your path:
   ```
   $ mkdir ~/bin
   $ PATH=~/bin:$PATH
   ```
2. Download the Repo tool and ensure that it is executable:
   ```
   $ curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
   $ chmod a+x ~/bin/repo
   ```
3. Verify the SHA-1 or SHA-256 checksum:
   ```
   version 1.21(SHA-1): b8bd1804f432ecf1bab730949c82b93b0fc5fede
   version 1.22(SHA-1): da0514e484f74648a890c0467d61ca415379f791
   version 1.23(SHA-256): e147f0392686c40cfd7d5e6f332c6ee74c4eab4d24e2694b3b0a0c037bf51dc5
   ```

> 参考链接：  
> [https://source.android.com/setup/build/downloading](https://source.android.com/setup/build/downloading)