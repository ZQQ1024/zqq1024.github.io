---
title: "安装网易云英语"
---

1. Download deb package:
   ```
   $ wget "http://d1.music.126.net/dmusic/netease-cloud-music_1.1.0_amd64_ubuntu.deb"
   ```
2. Install package:
   ```
   $ dpkg -i netease-cloud-music_1.1.0_amd64_ubuntu.deb
   ```
3. If errors occur, install the dependencies:
   ```
   sudo apt-get install -f
   ```
4. Run(Command Line and Desktop):
   ```
   $ sh -c "unset SESSION_MANAGER && netease-cloud-music"
   $ sudo vi /usr/share/applications/netease-cloud-music.desktop
   # Update Exec=sh -c "unset SESSION_MANAGER && netease-cloud-music %U"

> 参考地址：  
[https://music.163.com/#/download](https://music.163.com/#/download)  
[https://www.zhihu.com/question/277330447/answer/478510195](https://www.zhihu.com/question/277330447/answer/478510195)