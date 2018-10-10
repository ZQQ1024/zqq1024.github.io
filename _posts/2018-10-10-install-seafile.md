---
title: "安装Seafile同步盘"
classes: wide
toc: false
---

1. Run the seafile server container:
   ```
   $ docker run -d --name seafile \
   -e SEAFILE_SERVER_HOSTNAME=seafile.example.com \
   -v /opt/seafile-data:/shared \
   -p 80:80 \
   seafileltd/seafile:latest
   ```
2. Custom Admin Username and Password:
   ```
   $ docker run -d --name seafile \
   -e SEAFILE_SERVER_HOSTNAME=seafile.example.com \
   -e SEAFILE_ADMIN_EMAIL=me@example.com \
   -e SEAFILE_ADMIN_PASSWORD=a_very_secret_password \
   -v /opt/seafile-data:/shared \
   -p 80:80 \
   seafileltd/seafile:latest
   ```
   The default admin account is `me@example.com` and the password is `asecret`.
   If you forget the admin password, you can add a new admin account:
   ```
   $ docker exec -it seafile /opt/seafile/seafile-server-latest/reset-admin.sh
   ```
3. Email configuration:
   ```
   # append following to /opt/seafile-data(shared)/seafile/conf directory, seahub_settings.py file
   # QQ mail example
   EMAIL_USE_SSL = True
   EMAIL_HOST = 'smtp.qq.com'
   EMAIL_HOST_USER = 'username@domain.com'
   EMAIL_HOST_PASSWORD = 'Auth_Code not QQ password'
   EMAIL_PORT = '465'
   DEFAULT_FROM_EMAIL = EMAIL_HOST_USER
   SERVER_EMAIL = EMAIL_HOST_USER
   ```
4. Enable Onlyoffice online view:
   ```
   $ docker run -dit -p 88:80 --restart always --name oods onlyoffice/documentserver

   # append following to /opt/seafile-data(shared)/seafile/conf directory, seahub_settings.py file
   # Enable Only Office

   ENABLE_ONLYOFFICE = True
   VERIFY_ONLYOFFICE_CERTIFICATE = False
   ONLYOFFICE_APIJS_URL = 'http{s}://{your OnlyOffice server's domain or IP}/web-apps/apps/api/documents/api.js'
   ONLYOFFICE_FILE_EXTENSION = ('doc', 'docx', 'ppt', 'pptx', 'xls', 'xlsx', 'odt', 'fodt', 'odp', 'fodp', 'ods', 'fods')
   ONLYOFFICE_EDIT_FILE_EXTENSION = ('docx', 'pptx', 'xlsx')

   # container restart
   ```
5. Share file will get url like `http://ip:8000/f/4d87c77e7c974000a780/`, because use `docker -p`, 8000 port doesn't work, you need change `SERVICE_URL` in `/opt/seafile-data/seafile/conf/ccnet.conf` by deleting `8000` port.

> 参考链接：  
> Docker Deploy:  
> [https://manual.seafile.com/deploy/deploy_with_docker.html](https://manual-cn.seafile.com/config/sending_email.html)  
> Port:  
> [https://manual.seafile.com/deploy_windows/ports_used_by_seafile_windows_server.html](https://manual.seafile.com/deploy_windows/ports_used_by_seafile_windows_server.html)  
> Onlyoffice:  
> [https://manual.seafile.com/deploy/only_office.html](https://manual.seafile.com/deploy/only_office.html)  
> Email:  
[https://manual-cn.seafile.com/config/sending_email.html](https://manual-cn.seafile.com/config/sending_email.html)
