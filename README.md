# ft_server

42 본 과정의 첫 시스템 관리·인프라 과제. **Docker** 기술을 처음 접하면서 단일 컨테이너 안에 **Nginx + PHP-FPM + MariaDB + WordPress + phpMyAdmin** 풀스택을 자동으로 구성하는 웹 서버를 만든다.

> 원본 과제 명세: [`subject.pdf`](./subject.pdf) (영문)

## 과제 개요 / 요구사항

- **컨테이너**: Debian Buster 베이스, **단일 Docker 컨테이너**에 모든 서비스를 띄움 — `docker-compose` 사용 금지
- **서비스 스택**
  - **Nginx**: 웹 서버 (HTTPS 지원, URL 기반 라우팅)
  - **PHP-FPM**: PHP 처리 (FastCGI)
  - **MariaDB / MySQL**: SQL 데이터베이스
  - **WordPress**: 메인 사이트
  - **phpMyAdmin**: DB 관리 UI
- **요구 동작**
  - 모든 서비스가 컨테이너 시작 시 자동 기동
  - **SSL/TLS** 지원 (HTTPS)
  - URL에 따라 WordPress / phpMyAdmin으로 라우팅
  - **autoindex 활성화/비활성화**가 가능해야 함
- **구조 제약**
  - `Dockerfile`은 레포 루트
  - 모든 설정·자산 파일은 `srcs/` 폴더 안

## 레포 구조

```
.
├── Dockerfile
└── srcs/
    ├── init.sh                                  # 컨테이너 진입 스크립트
    ├── default                                  # nginx config (autoindex off)
    ├── default_a                                # nginx config (autoindex on)
    ├── wp-config.php                            # WordPress DB 연결 설정
    ├── config.inc.php                           # phpMyAdmin 설정
    ├── latest.tar.gz                            # WordPress 설치본
    └── phpMyAdmin-5.0.2-all-languages.tar.gz    # phpMyAdmin 설치본
```

## 풀이 노트

### Dockerfile — 빌드 단계는 최소, 런타임에 위임

```dockerfile
FROM debian:buster
RUN apt-get update && apt-get -y upgrade
RUN apt-get -y install vim nginx openssl php-fpm mariadb-server \
                       php-mysql php-mbstring php-curl
COPY ./srcs/init.sh /
COPY ./srcs/default /etc/nginx/sites-available/
COPY ./srcs/default_a /
COPY ./srcs/config.inc.php /
COPY ./srcs/wp-config.php /
COPY ./srcs/phpMyAdmin-5.0.2-all-languages.tar.gz /
COPY ./srcs/latest.tar.gz /
CMD bash init.sh
```

- 빌드 시점엔 **패키지 설치 + 자산 복사**만 하고, 압축 해제·DB 초기화·서비스 기동은 모두 `init.sh`로 미룸. 이미지 자체는 가볍게 두고 컨테이너 실행 시 1회 셋업.
- 정식 모범 사례는 빌드 시 가능한 모든 작업을 끝내는 것이지만, 본 과제 시점엔 학습 목적상 init 스크립트 패턴을 그대로 둠.

### init.sh — 컨테이너 시작 시퀀스

1. **autoindex 입력 받기** — `read AUTOINDEX`로 `on`/`off` 사용자 입력
2. **SSL 인증서 자체 발급** — `openssl req`로 4096-bit RSA self-signed cert (365일, `CN=localhost`)를 `/etc/ssl/certs|private`에 생성
3. **phpMyAdmin 설치** — tar 압축 해제 → `/var/www/html/phpmyadmin`로 이동, 설정 파일 복사
4. **DB 부트스트랩**
   - `service mysql start`
   - phpMyAdmin이 가져온 `create_tables.sql` 실행
   - `wordpress` DB 생성 + 사용자 `seojeong@localhost` 생성·전체 권한 부여
5. **WordPress 설치** — tar 압축 해제 → `/var/www/html/wordpress`로 이동, 소유자 `www-data:www-data`, `wp-config.php` 복사
6. **autoindex 분기**
   - `on`이면 `default_a`(autoindex on 버전)를 nginx 사이트 설정으로 덮어씀
   - `off`면 `default_a` 삭제 (`default`가 그대로 사용됨)
7. **서비스 기동** — `php7.3-fpm start` → `nginx start` → `mysql restart`
8. 마지막에 `bash`로 컨테이너 살려둠 — foreground process가 없으면 컨테이너가 즉시 종료되므로 의도적인 hold

### autoindex 토글 — config 파일 두 벌

`autoindex on/off`를 동적으로 토글하는 대신 **nginx config 파일을 두 벌(`default`, `default_a`)** 미리 두고, 시작 시 입력에 따라 어느 쪽을 `sites-available/default`로 둘지 결정. 단순하지만 init 스크립트가 끝나면 변경 불가 — autoindex를 바꾸려면 컨테이너 재기동.

### SSL — self-signed cert

```sh
openssl req -newkey rsa:4096 -x509 -days 365 -nodes \
  -subj "/C=KR/ST=GAEPO/L=Seoul/O=ECOLE42/OU=42SEOUL/CN=localhost" \
  -keyout /etc/ssl/private/localhost.dev.key \
  -out /etc/ssl/certs/localhost.dev.crt
```

`-nodes`로 키 비밀번호 없이 발급해 nginx가 비대화식으로 읽을 수 있게 한다. 브라우저는 자체 서명이라 경고를 띄우지만 학습 환경에선 무시하고 진행.

### Nginx 라우팅 — HTTP → HTTPS 강제

- `listen 80 default_server` → `return 301 https://$host$request_uri`로 모든 평문 요청을 HTTPS로 리다이렉트
- `listen 443` 블록은 `root /var/www/html`을 두고 `try_files`로 정적 파일 매칭
- `.php` 요청은 `fastcgi_pass unix:/run/php/php7.3-fpm.sock`로 PHP-FPM에 위임

## 빌드 / 사용법

```sh
# 이미지 빌드
docker build -t ft_server .

# 컨테이너 실행 (인터랙티브 — autoindex 입력을 받아야 함)
docker run -it -p 80:80 -p 443:443 ft_server

# 컨테이너 시작 후 프롬프트
# AUTOINDEX [on/off]?  → on 또는 off 입력
```

접속:

| URL | 도착지 |
|---|---|
| `http://localhost` | `https://localhost`로 301 리다이렉트 |
| `https://localhost` | WordPress |
| `https://localhost/phpmyadmin` | phpMyAdmin |

DB 자격 증명 (개발용):

| 항목 | 값 |
|---|---|
| DB 이름 | `wordpress` |
| 사용자 | `seojeong` |
| 비밀번호 | `seojeong` |
| 호스트 | `localhost` |

## 메모

- `srcs/latest.tar.gz`, `srcs/phpMyAdmin-5.0.2-all-languages.tar.gz`는 당시 받은 설치본을 그대로 커밋해둔 것. 현재 업스트림에서 받으면 더 최신 버전이지만, 본 빌드의 재현성을 위해 보존.
- `wp-config.php`의 평문 DB 비밀번호는 학습용 개발 환경 한정 — 실제 배포에서는 환경 변수·시크릿 매니저로 빼는 게 정석.
- `subject.pdf`는 [evgenkarlson/ALL_SCHOOL_42 — ft_server](https://github.com/evgenkarlson/ALL_SCHOOL_42/tree/master/00_Projects__(%D0%9E%D1%81%D0%BD%D0%BE%D0%B2%D0%BD%D0%BE%D0%B5_%D0%9E%D0%B1%D1%83%D1%87%D0%B5%D0%BD%D0%B8%D0%B5)/05_Infrastructure_and_Admin/ft_server)의 2020-03-17 영문 판본.
