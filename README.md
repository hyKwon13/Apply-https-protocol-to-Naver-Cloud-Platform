# Apply HTTPS Protocol to Naver Cloud Platform

이 가이드는 네이버 클라우드 플랫폼에서 Uvicorn으로 배포 중인 웹 사이트에 도메인을 연결하고, SSL 인증서를 적용하여 HTTPS를 설정하는 방법을 다룹니다. 도메인과 SSL 인증서는 무료로 발급받을 수 있으며, Public IP도 아이디 생성 후 초기 3개월 동안은 Naver Cloud Platform의 크레딧을 통해 무료로 사용할 수 있습니다. 단, 3개월 이후부터는 Public IP에 대한 요금이 과금됩니다.


## 도메인 연결

먼저 네이버 클라우드 플랫폼 포털에서 다음 경로로 이동하여 Public IP를 발급받습니다:
`콘솔 > Services > Compute > Server > Public IP`

![Public IP 발급](https://github.com/hyKwon13/Apply-https-protocol-to-Naver-Cloud-Platform/assets/117807382/6cad1b54-1286-4cd3-a9d1-206b1e9d65ef)

그 다음, [내도메인.한국](https://xn--220b31d95hq8o.xn--3e0b707e/)에서 도메인을 발급받고 위에서 발급받은 Public IP를 추가합니다.

![도메인 설정](https://github.com/hyKwon13/Apply-https-protocol-to-Naver-Cloud-Platform/assets/117807382/2586b3e9-67b0-4227-b39f-80e0bede2e4d)

현재 3389 포트를 붙여야만 접속이 가능하기 때문에 외부에서 80번 포트로 요청이 오면 리눅스에서 3389번 포트로 포워딩하도록 설정해야 합니다. 이를 위해 네이버 클라우드 플랫폼의 ACG 규칙을 재설정합니다. ACG 규칙 설정 콘솔에서 다음 규칙들을 추가합니다:
- TCP 0.0.0.0/0 80
- TCP 0.0.0.0/0 443

![ACG 규칙 설정](https://github.com/hyKwon13/Apply-https-protocol-to-Naver-Cloud-Platform/assets/117807382/179e1fb0-0d79-4c14-a7b4-dd3b20881307)

이후 서버에 접속하여 아래 명령어를 입력합니다:
```bash
sudo iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 -j REDIRECT --to-port 3389
```

이 명령어는 80 포트로 들어온 요청을 3389 포트로 포워딩합니다.

## HTTPS 설정
### Nginx 설치 및 설정

Nginx를 사용하여 SSL 설정을 진행합니다. 먼저 Nginx를 설치합니다:

```bash
sudo apt-get install -y nginx nginx-common nginx-full
```
이전에 설정한 포트 포워딩 정책을 삭제합니다:

```bash
sudo iptables -t nat -L --line-numbers
sudo iptables -t nat -D PREROUTING 1
```

이제 Nginx 설정 파일을 수정합니다:
```bash
sudo vi /etc/nginx/nginx.conf
```

다음과 같이 자신의 IP와 웹사이트를 입력합니다:
```nginx
user www-data;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {}

http {
  upstream app {
    server xxx.xxx.xxx.xxx:3389;
  }

  server {
    listen 80;

    location / {
      proxy_pass http://mywebsite.kr;
    }
  }
}
```

Nginx를 재시작합니다:
```bash
sudo service nginx restart
```
이제 Naver Cloud Platform에서 발급받은 Public IP와 내도메인에서 발급받은 도메인이 연결된 것을 확인할 수 있습니다.

## SSL 설정
HTTPS 프로토콜은 SSL로 암호화된 HTTP입니다. SSL 인증서는 Certbot을 사용하여 Let's Encrypt 인증서를 발급받겠습니다.

### Certbot 설치
Certbot을 설치합니다:
```bash
sudo apt-get update -y
sudo apt-get install software-properties-common -y
sudo apt-get install certbot -y
```

### 인증서 발급

아래 명령어를 사용하여 SSL 인증서를 발급받습니다. xxx.kro.kr 자리에 자신의 도메인 이름을 입력합니다:

```bash
sudo certbot certonly -d xxx.kro.kr --manual --preferred-challenges dns
```
이메일을 입력하고 약관에 동의합니다. 그러면 인증 과정에서 _acme-challenge 값이 제공됩니다. 이때 터미널을 잠시 두고, "내 도메인" 사이트에서 _acme-challenge 값과 함께 제공된 DNS TXT 레코드를 입력해 주세요. 이 과정이 완료되어야 정상적으로 인증서가 발급됩니다.

"내 도메인" 사이트에서 TXT 값을 입력하고 수정 버튼을 누른 뒤, 약 1~2분 정도 기다렸다가 터미널에서 다음 단계를 진행하면 성공적으로 인증서를 받을 수 있습니다.

![다운로드 (1)](https://github.com/hyKwon13/Apply-https-protocol-to-Naver-Cloud-Platform/assets/117807382/a361fe6d-77dc-434b-a5f0-0d4150ba1fd8)
![다운로드 (2)](https://github.com/hyKwon13/Apply-https-protocol-to-Naver-Cloud-Platform/assets/117807382/320f28c7-ef9b-4db6-84f2-176858966b48)

### Nginx SSL 설정
SSL 인증서를 성공적으로 발급받았다면, Nginx 설정 파일을 수정합니다:

```bash
sudo vi /etc/nginx/nginx.conf
```
다음과 같이 수정합니다:

```nginx
user www-data;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
  worker_connections 768;
}

http {
  upstream app {
    server xxx.xxx.xxx.xxx:3389;
  }

  server {
    listen 80;
    server_name xxx.kro.kr;
    return 301 https://$host$request_uri;
  }

  server {
    listen 443 ssl;
    server_name xxx.kro.kr;

    ssl_certificate /etc/letsencrypt/live/xxx.kro.kr/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/xxx.kro.kr/privkey.pem;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;
    ssl_ciphers ECDH+AESGCM:ECDH+AES256:ECDH+AES128:DH+3DES:!ADH:!AECDH:!MD5;
    add_header Strict-Transport-Security "max-age=31536000" always;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    location / {
      proxy_pass http://xxx.xxx.xxx.xxx:3389;
    }
  }
}
```

Nginx를 재시작합니다:

```bash
sudo service nginx restart
```

이제 웹사이트에 접속하면 HTTPS가 적용된 것을 확인할 수 있습니다.


## 인증서 자동 갱신

Let's Encrypt 인증서는 3개월마다 만료되므로 자동 갱신을 설정해야 합니다. Crontab을 사용하여 인증서를 자동으로 갱신합니다.

Crontab 설정 파일을 엽니다:
```bash
sudo vi /etc/crontab
```

맨 아랫줄에 다음 명령어를 추가합니다:
```bash
15 3 * * * certbot renew --quiet --renew-hook "/etc/init.d/nginx reload"
```

이제 3개월에 한 번씩 자동으로 인증서를 갱신합니다.
