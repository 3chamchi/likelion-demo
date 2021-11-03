배포용 프로젝트
---

# 멋쟁이사자처럼 직장인 6기

## 목차
* [Ubuntu 파이썬 환경 구성](#Ubuntu_파이썬_환경_구성)
* [배포를 위한 장고 설정](#배포를_위한_장고_설정)
* [Nginx 구축](#Nginx_구축)
* [Gunicorn 구성](#Gunicorn_구성)

## Ubuntu 파이썬 환경 구성
```console
sudo apt-get update
sudo apt-get upgrade -y

# Install Python Dependencies
sudo apt-get install -y make build-essential libssl-dev zlib1g-dev libbz2-dev \
libreadline-dev libsqlite3-dev wget curl llvm libncurses5-dev libncursesw5-dev \
xz-utils tk-dev

# Install Pyenv
git clone https://github.com/pyenv/pyenv.git ~/.pyenv

echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bashrc

echo 'export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bashrc

echo -e 'if command -v pyenv 1>/dev/null 2>&1; then\n  eval "$(pyenv init --path)"\nfi' >> ~/.bashrc

source .bashrc

# Install Python 3.9.7 with Pyenv
pyenv install 3.9.7
pyenv global 3.9.7
```  

## 배포를 위한 장고 설정
### 패키지 관리
* (개발환경) 패키지 명세 : `pip freeze > requirements.txt`
* (배포환경) 패키지 설치 : `pip install -r requirements.txt`

### 환경설정
`settings.py` 수정
```python
# ... 생략
DEBUG = False  # 배포 시 디버그 False
ALLOWED_HOSTS = ['*']  # 사용자가 접속할 서버 아이피 또는 도메인, '*'인 경우 전체 허용
# ... 생략

# 배포시 STATIC_ROOT 설정, 개발 중에는 STATICFILES_DIRS 설정
STATIC_ROOT = os.path.join(BASE_DIR, 'static')
# STATICFILES_DIRS = [os.path.join(BASE_DIR, 'static')]
```

### Static 파일 취합합
* Static 파일 취합 : `python manage.py collectstatic`  

### Media 폴더 생성
* Media 폴더 생성 : `mkdir media`

## Gunicorn 구성
### Gunicorn 설치
```console
pip install guniucorn
```

### 소켓 생성
파일 생성 : `/etc/systemd/system/gunicorn.socket`
* `sudo vi /etc/systemd/system/gunicorn.socket`

```bash
[Unit]
Description=gunicorn socket

[Socket]
ListenStream=/run/gunicorn.sock

[Install]
WantedBy=sockets.target
```

### 서비스 생성
파일 생성 : `/etc/systemd/system/gunicorn.service`  
* `sudo vi /etc/systemd/system/gunicorn.service`
* `PROJECT_ROOT_PATH`: 장고 프로젝트 이름
```bash
[Unit]
Description=gunicorn daemon
Requires=gunicorn.socket
After=network.target

[Service]
User=ubuntu
Group=ubuntu
WorkingDirectory=/home/ubuntu/{PROJECT_ROOT_PATH}
ExecStart=/home/ubuntu/{PROJECT_ROOT_PATH}/venv/bin/gunicorn \
          --access-logfile - \
          --workers 3 \
          --bind unix:/run/gunicorn.sock \
          config.wsgi:application

[Install]
WantedBy=multi-user.target
```

### 명령어
```consle
# OS 시작 시 자동 실행
sudo systemctl enable gunicorn.service

# 실행
sudo systemctl start gunicorn.service

# 재실행 -> 코드 수정 시 반영
sudo systemctl restart gunicorn.service

# 상태
sudo systemctl status gunicorn.service

# 종료
sudo systemctl stop gunicorn.service
```
```consle
# OS 시작 시 자동 실행
sudo systemctl enable gunicorn.socket

# 실행
sudo systemctl start gunicorn.socket

# 재실행
sudo systemctl restart gunicorn.socket

# 상태
sudo systemctl status gunicorn.socket

# 종료
sudo systemctl stop gunicorn.socket
```


## Nginx 구축
### Nginx 설치
```console
sudo apt-get install -y nginx
```
### Nginx 설정 변경
`/etc/nginx/conf.d/likelion.conf` 내용  
* `SERVER_IP OR DOMAIN`: 사용자가 접속할 IP 또는 도메인  
* `PROJECT_ROOT_PATH`: 장고 프로젝트 이름
```bash
server {
    listen 80;
    server_name {SERVER_IP OR DOMAIN};

    location = /favicon.ico { access_log off; log_not_found off; }

    location /static {
        alias /home/ubuntu/{PROJECT_ROOT_PATH}/static;
    }

    location /media {
        alias /home/ubuntu/{PROJECT_ROOT_PATH}/media;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/run/gunicorn.sock;
    }
}
```

### Nginx 명령어
* Nginx 설정 문법 검사 : `nginx -t`
```console
# 실행
sudo service nginx start

# 재실행
sudo service nginx restart

# 종료
sudo service nginx stop

# 상태
sudo service nginx status
```


배포 1차 참고 코드
```console
sudo apt-get update
sudo apt-get upgrade -y

# Install Pyenv Dependencies
sudo apt-get install -y make build-essential libssl-dev zlib1g-dev libbz2-dev \
libreadline-dev libsqlite3-dev wget curl llvm libncurses5-dev libncursesw5-dev \
xz-utils tk-dev

# Install Pyenv
git clone https://github.com/pyenv/pyenv.git ~/.pyenv

echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bashrc

echo 'export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bashrc

echo -e 'if command -v pyenv 1>/dev/null 2>&1; then\n  eval "$(pyenv init --path)"\nfi' >> ~/.bashrc

source .bashrc

# Install Python 3.10 with Pyenv
pyenv install 3.9.7
pyenv global 3.9.7
```
