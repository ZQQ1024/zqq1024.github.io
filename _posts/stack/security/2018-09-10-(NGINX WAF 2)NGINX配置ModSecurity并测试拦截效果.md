---
title: "(NGINX WAF 2)NGINX配置ModSecurity并测试拦截效果"
categories:
  - security
tags:
  - Nginx
  - WAF
classes: wide

excerpt: "NGINX配置ModSecurity并测试拦截效果"
---

该文接着上文，上文主要是装了ModSecurity软件，并没有规则，这篇配置一个简单的规则和使用开源的OWASP CRS规则，并分别测试拦截效果


# 3 测试
## 3.1 简单规则
Configure a simple ModSecurity rule to block certain requests
to a demo application.NGINX acts as the reverse proxy in the example, but the
same configuration applies to load balancing.

1.Creating the Demo Web Application:  
the demo application is simply an NGINX virtual server that returns status code 200 and a text message.Create the file **/etc/nginx/conf.d/echo.conf** with the following content.
```
server {
	 listen localhost:8085;
	 location / {
	     default_type text/plain;
         return 200 "Thank you for requesting ${request_uri}\n";
	 }
}
```

2.Test the application by reloading the NGINX configuration and making a request:
```
$ sudo nginx -s reload
$ curl -D - http://localhost:8085
HTTP/1.1 200 OK
Server: nginx/1.15.3
Date: Mon, 10 Sep 2018 11:52:23 GMT
Content-Type: text/plain
Content-Length: 27
Connection: keep-alive

Thank you for requesting /
```
注意：这里只是用NGINX自身模拟了一个业务源站，真实的ModSecurity是配置在负载均衡器或者反向代理服务器上面的

3.Configuring NGINX as a reverse proxy:
Configure NGINX as a reverse proxy for the demo application.Create the file **/etc/nginx/conf.d/proxy.conf** with the following content.
```
server {
	 listen 80;
	 location / {
		
     proxy_pass http://localhost:8085;
	 proxy_set_header Host $host;
	}
}
```
注意：If any other virtual servers (server blocks) in your NGINX configuration listen on
port 80, you need to disable them for the reverse proxy to work correctly. For example, the
**/etc/nginx/conf.d/default.conf** file provided in the nginx package includes such a server block.
Comment out or remove the server block, but **do not remove or rename the default.conf file
itself** – if the file is missing during an upgrade, it is automatically restored, which can break the
reverse‐proxy configuration.（默认配置文件已经包含了这样一个`server`块了，将其删除或注释掉就行）

4.Reload the NGINX configuration, and verify that a request succeeds, which confirms that the proxy is working correctly:
```
$ sudo nginx -s reload

$ curl -D - http://localhost
HTTP/1.1 200 OK
Server: nginx/1.15.3
Date: Mon, 10 Sep 2018 11:54:23 GMT
Content-Type: text/plain
Content-Length: 27
Connection: keep-alive

Thank you for requesting /
```

5.Protecting the Demo Web Application:  

Configure ModSecurity 3.0 to protect the demo web application by blocking
certain requests.Create the folder **/etc/nginx/modsec** for storing ModSecurity configuration.  
```
$ sudo mkdir /etc/nginx/modsec
```
Download the file of recommended ModSecurity configuration
from the v3/master branch of the ModSecurity GitHub repo and
name it **modsecurity.conf**（或者用之前的ModSecurity仓库下面也有一个相同的）.
```
$ cd /etc/nginx/modsec
$ sudo wget https://raw.githubusercontent.com/SpiderLabs/ModSecurity/v3/master/modsecurity.conf-recommended
$ sudo mv modsecurity.conf-recommended modsecurity.conf
```
Enable execution of rules by commenting out the existing **SecRuleEngine** directive in **modsecurity.conf**.
```
# SecRuleEngine DetectionOnly
SecRuleEngine On
```
Create the main ModSecurity 3.0 configuration file, **/etc/nginx/modsec/main.conf**, and define a rule in it.
```
# Include the recommended configuration
Include /etc/nginx/modsec/modsecurity.conf

# A test rule
SecRule ARGS:testparam "@contains test" "id:1234,deny,log,status:403"
```
- **Include** – Includes the recommended configuration from the
**modsecurity.conf** file.
- **SecRule** – Creates a rule that protects the application by
blocking requests and returning status code **403** when the
testparam parameter in the query string contains the string test.

Change the reverse proxy configuration file (**/etc/nginx/conf.d/proxy.conf**) to enable the ModSecurity 3.0:
```
server {
	 listen 80;
	 
	 modsecurity on;
     modsecurity_rules_file /etc/nginx/modsec/main.conf;
	 location / {
		
     proxy_pass http://localhost:8085;
	 proxy_set_header Host $host;
	 }
}
```
- **modsecurity on** – Enables the ModSecurity 3.0.
- **modsecurity_rules_file** – Specifies the location of the ModSecurity 3.0 configuration file.

6.Reload the NGINX configuration, verify that the rule works correctly(状态码由200变成403):
```
$ curl -D - http://localhost/foo?testparam=thisisatestofmodsecurity
HTTP/1.1 200 OK
Server: nginx/1.15.3
Date: Mon, 10 Sep 2018 12:14:57 GMT
Content-Type: text/plain
Content-Length: 65
Connection: keep-alive

Thank you for requesting /foo?testparam=thisisatestofmodsecurity

$ sudo nginx -s reload

$ curl -D - http://localhost/foo?testparam=thisisatestofmodsecurity
HTTP/1.1 403 Forbidden
Server: nginx/1.15.3
Date: Mon, 10 Sep 2018 12:15:24 GMT
Content-Type: text/html
Content-Length: 169
Connection: keep-alive

<html>
<head><title>403 Forbidden</title></head>
<body bgcolor="white">
<center><h1>403 Forbidden</h1></center>
<hr><center>nginx/1.15.3</center>
</body>
</html>
```

## 3.2 OWASP CRS规则
After installing the ModSecurity 3.0 **software**, the next step is to install ModSecurity's **rule** set(s).

The rules define the attack patterns and determine what ModSecurity will block.The OWASP Core Rule Set (CRS) is the base rule set that all ModSecurity users should install. 

The CRS is community-maintained and contains rules to help stop common attack vectors, such as SQL injection, cross-site scripting, and many others.

1.Running the Nikto Scanning Tool:  
Many attackers run **vulnerability scanners** to identify security vulnerabilities in a target website or app. Once they learn what vulnerabilities are present, they can **launch the appropriate attacks**.

We’re using the Nikto scanning tool to generate malicious requests, including
probes for the presence of files known to be vulnerable, cross‐site scripting
(XSS), and other types of attack.

```
$ git clone https://github.com/sullo/nikto
Cloning into 'nikto'...
$ cd nikto
$ perl program/nikto.pl -h localhost

xxx
7678 requests: 0 error(s) and 612 item(s) reported on remote host
xxx
```
612 item(s) in the output are potential vulnerabilities.

2.Enabling the OWASP CRS:

Download the latest OWASP CRS from GitHub and extract the rules into
**/usr/local** or another location of your choice.（写的时候版本还是3.0.2）
```
$ wget https://github.com/SpiderLabs/owasp-modsecurity-crs/archive/v3.0.2.tar.gz
$ tar -xzvf v3.0.2.tar.gz
$ sudo mv owasp-modsecurity-crs-3.0.2 /usr/local
```

Create the **crs‐setup.conf** file as a copy of **crs‐setup.conf.example**.
```
$ cd /usr/local/owasp-modsecurity-crs-3.0.2
$ sudo cp crs-setup.conf.example crs-setup.conf
```

Add Include directives in the main ModSecurity configuration file (**/etc/nginx/modsec/main.conf**).
```
# Include the recommended configuration
Include /etc/nginx/modsec/modsecurity.conf

# OWASP CRS v3 rules
Include /usr/local/owasp-modsecurity-crs-3.0.2/crs-setup.conf
Include /usr/local/owasp-modsecurity-crs-3.0.2/rules/*.conf
```
Reload the NGINX configuration.
```
$ sudo nginx -s reload

$ perl program/nikto.pl -h localhost
xxx
26303 requests: 0 error(s) and 303 item(s) reported on remote host
xxx
```

3.Blocking Requests with XSS Attempts:  
OWASP CRS is not currently configured to block requests that contain XSS attempts in the request URL, such as:
```
$ curl 'http://localhost/<script>alert('Vulnerable')</script>'
Thank you for requesting /<script>alert(Vulnerable)</script>
```
To block requests with XSS attempts, edit **rules 941160 and 941320** in the CRS’s
XSS Application Attack rule set (**/usr/local/owasp-modsecurity-crs-3.0.2/rules/REQUEST-941-APPLICATION-ATTACK-XSS.conf**) by adding **REQUEST_URI** at the start of the variables list for each rule.
```
SecRule REQUEST_URI|REQUEST_COOKIES|!REQUEST_COOKIES:/__utm/ ...
```
Reload the NGINX configuration, verify block requests that contain XSS attempts in the request URL:
```
$ sudo nginx -s reload
$ curl 'http://localhost/<script>alert('Vulnerable')</script>'
<html>
<head><title>403 Forbidden</title></head>
<body bgcolor="white">
<center><h1>403 Forbidden</h1></center>
<hr><center>nginx/1.15.3</center>
</body>
</html>
```

注意：we configured our demo application to return status code 200 for every request, without actually ever delivering a file.Nikto is interpreting these 200 status codes to mean that the file it is requesting actually exists, which in the context of our application is a false positive.（由于demo sever只返回200，nikto有些检测把返回码200当成某些文件存在，这个对nikto而言是误报，多报了）

## 3.3 日志
1.Setting up logging:  
The error_log directive in the main **/etc/nginx/nginx.conf** file provided
with NGINX is configured to write messages to **/var/log/nginx/error.log**
at the **warn** level and higher. If you want to capture **all** messages from the
ModSecurity 3.0, you need to set the logging level to **info** instead.
```
error_log /var/log/nginx/error.log info;
```

2.Controlling audit and debug logging:  
ModSecurity has two types of logs:
- An audit log. For every transaction that’s blocked, ModSecurity provides detailed logs about the transaction and why it was blocked.
- A debug log. When turned on, this log keeps extensive information about everything that ModSecurity does.

ModSecurity also supports audit logging as configured with the **SecAudit*** set of directives. Audit logging is enabled in the recommended configuration that we downloaded to **/etc/nginx/modsec/modsecurity.conf**, but we recommend disabling it in production environments, both because audit logging affects ModSecurity performance and because the log file can grow large very quickly and exhaust disk space.To disable audit logging, change the value of the SecAuditEngine directive in **modsecurity.conf** to **off**.
```
SecAuditEngine off
```

If you experience problems with the ModSecurity 3.0, you can enable debug logging by changing the **SecDebugLog** and **SecDebugLogLevel** directives in **modsecurity.conf** to the following values. Like audit logging, debug logging generates a large volume of output and affects ModSecurity performance, so we recommend disabling it in production.
```
SecDebugLog /tmp/modsec_debug.log
SecDebugLogLevel 9
```