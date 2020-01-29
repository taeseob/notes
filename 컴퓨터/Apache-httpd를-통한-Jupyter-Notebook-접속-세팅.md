# Apache httpd를 통한 Jupyter Notebook 접속 세팅

2020년 1월

언제 어디서나 주피터 노트북을 사용해서 금융 데이터 분석 및 퀀트 투자 시뮬레이션을 해 보고자 AWS EC2 인스턴스에 주피터 노트북을 설치하여 사용하고 있었다. 하지만 80/443 포트만 사용 가능한 경우(...ㅎㅎ)가 있어서 주피터 노트북의 기본 포트인 8888이 아닌 80 포트를 통해 접속 할 수 있도록 세팅하였다.

아파치 웹서버를 통해 세팅을 완료했다. 중간에 여러가지 시행착오를 겪고서는 주피터 노트북을 root 권한으로 수행하여 80포트를 직접 사용하도록 해 보기도 하였다. 하지만 노트북 파일이 root 권한으로 생성되는 등 여러가지로 좋지 않은 방법이어서 다시 아파치 웹서버를 사용하는 방법으로 돌아왔다.

HTTP 프로토콜의 ProxyPass 설정은 문제없이 동작했었는데, 문제는 웹소켓 프로토콜의 통신이었다. 여러 인터넷 문서에 웹소켓도 ProxyPass 설정이 잘 적용되는 것으로 나와있어서 시도 해 보았지만 실패했고, 결국 [JupyterHub 관련 문서](https://jupyterhub.readthedocs.io/en/stable/reference/config-proxy.html#apache)의 도움을 받아 해결하였다. RewriteEngine 설정으로 잘 동작하는것을 확인했다.

주석 처리 된 불필요한 설정은 추후에 혹시 참고할까 싶어서 남겨둔다.

```
(python3) [ec2-user@ip-xxx-xx-xx-xxx ~]$ sudo cat /etc/httpd/conf.d/vhost-jupyter-notebook.conf
<VirtualHost *:80>
	ServerName localhost
	#ProxyPreserveHost On
	#ProxyRequests off

	#<Proxy *>
	#	Require all granted
	#</Proxy>

	#Redirect permanent / http://xx.xxx.xx.xx/
	#Redirect permanent / http://localhost/
	#RequestHeader set Origin "http://xx.xxx.xx.xx:8888"
	#RequestHeader set Origin ""
	#RequestHeader set Host "xx.xxx.xx.xx:8888"

	#<Location "/api/kernels">
		#ProxyPass ws://localhost:8888/api/kernels retry=1 acquire=3000 timeout=600 Keepalive=On
		#ProxyPass ws://localhost:8888/api/kernels
		#ProxyPassReverse ws://localhost:8888/api/kernels
		#RequestHeader set Upgrade "WebSocket"
		#RequestHeader set Origin "http://xx.xxx.xx.xx:8888"
		#RequestHeader set Origin ""
		#RequestHeader set Host "xx.xxx.xx.xx:8888"
		#Header set Origin "http://xx.xxx.xx.xx:8888"
		#Header set Origin ""
		#LogLevel debug
	#</Location>

	RewriteEngine On
	RewriteCond %{HTTP:Connection} Upgrade [NC]
	RewriteCond %{HTTP:Upgrade} websocket [NC]
	RewriteRule /(.*) ws://localhost:8888/$1 [P,L]

	<Location "/">
		ProxyPreserveHost on
		ProxyPass http://localhost:8888/
		ProxyPassReverse http://localhost:8888/
		#ProxyPassReverseCookieDomain localhost xx.xxx.xx.xx
		#RequestHeader set Origin "http://xx.xxx.xx.xx:8888"
		#RequestHeader set Host "xx.xxx.xx.xx:8888"
	</Location>
</VirtualHost>

```

