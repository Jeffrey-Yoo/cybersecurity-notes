# 웹 해킹 테스트 서버 구축 및 리눅스 명령어 정리

## 📡 네트워크 구조 이해

### 집 네트워크 흐름
```
SKB 통신사 모뎀 → 공유기(L2+L3 복합) → 각 디바이스
```

- **공유기의 두 가지 역할**
  - L3 역할: WAN(인터넷) ↔ LAN(내부) 라우팅 + NAT
  - L2 역할: 같은 네트워크 안 디바이스끼리 MAC 주소 기반 직접 통신

### VM 네트워크 구조
```
공유기 (192.168.x.0/24)
    ↓
노트북 (192.168.x.xxx)
    ↓ VirtualBox 설치
    ├── NAT 어댑터      → 가상 게이트웨이를 통해 인터넷 연결
    ├── 브릿지 어댑터   → 공유기와 같은 대역 사용 (같은 네트워크로 편입)
    └── Host-Only      → 호스트와 VM만 연결되는 격리된 가상 L2 스위치
```

> **브릿지 어댑터 핵심**  
> VM이 공유기와 같은 네트워크 대역을 가지게 되어  
> 같은 공유기 하의 모든 디바이스에서 VM에 원격 접속 가능

> **Host-Only 핵심**  
> VirtualBox가 가상 L2 스위치를 생성  
> 호스트(노트북)와 VM만 같은 세그먼트에 묶임  
> 인터넷 연결 없이 호스트-VM 간 직접 통신 가능

### IP 설정
- `ping IP -t` 로 패킷 송수신 확인 (Windows)
- 강의 환경 통일을 위해 DHCP → 고정 IP(x.x.x.101)로 변경
  - 서브넷 마스크: 255.255.255.0 (C클래스)
  - 게이트웨이: 공유기 IP와 동일하게 설정

### 공인 IP vs 사설 IP
| 구분 | 공인 IP | 사설 IP |
|---|---|---|
| 발급 | ISP (KT, SKB 등) | 공유기/관리자 |
| 인터넷 직접 통신 | ✅ | ❌ (NAT 필요) |
| 중복 | ❌ 전 세계 유일 | ✅ 내부끼리 중복 가능 |

**사설 IP 3대역 (필수 암기)**
```
10.0.0.0    ~ 10.255.255.255   (/8)
172.16.0.0  ~ 172.31.255.255   (/12)
192.168.0.0 ~ 192.168.255.255  (/16)
```

---

## 🐧 리눅스 명령어 정리

### 디렉터리 관련
| 명령어 | 설명 |
|---|---|
| `pwd` | 현재 위치 절대경로 출력 (Print Working Directory) |
| `cd 경로` | 디렉터리 이동 (cd .. 상위 / cd / 최상위 / cd ~ 홈) |
| `mkdir 폴더명` | 디렉터리 생성 (-p 옵션: 중간 경로 없어도 한번에 생성) |
| `rmdir 폴더명` | 빈 디렉터리 삭제 (내용 있으면 rm -r 사용) |

### 파일 관련
| 명령어 | 설명 |
|---|---|
| `ls -al` | 목록 출력 (-a 숨김파일 포함, -l 상세정보, -h 읽기 쉬운 단위) |
| `cp 원본 복사본` | 파일 복사 (-r 옵션: 디렉터리 복사) |
| `mv 원본 대상` | 파일 이동 또는 이름 변경 |
| `rm 파일명` | 파일 삭제 (-r 디렉터리, -f 강제) |
| `touch 파일명` | 빈 파일 생성 또는 타임스탬프 갱신 |
| `find 경로 -name "파일명"` | 파일 찾기 |
| `cat 파일명` | 파일 전체 내용 출력 |
| `tail 파일명` | 파일 하단 출력 (-f: 실시간 모니터링, 로그 분석에 활용) |
| `more / less` | 페이지 단위로 파일 읽기 (less가 더 기능 많음, /검색 가능) |
| `grep "단어" 파일` | 파일에서 특정 단어 검색 (-i 대소문자무시, -n 줄번호, -r 재귀) |
| `wc 파일명` | 줄/단어/바이트 수 출력 (-l 줄수만) |
| `chown 소유자:그룹 파일` | 파일 소유자 변경 |
| `chmod` | 파일 권한 변경 |

### 시스템 관련
| 명령어 | 설명 |
|---|---|
| `hostname` | 호스트명 확인 (hostnamectl set-hostname 이름: 변경) |
| `df -h` | 파티션 사용량 확인 (-h 읽기 쉬운 단위, 상세 모니터링은 df만) |
| `top` | 실시간 프로세스 현황 |

### 기타
```bash
|   # 파이프: 앞 명령어 출력을 다음 명령어 입력으로 연결
    # 예: grep "Failed" /var/log/auth.log | wc -l

>   # 출력을 파일로 저장 (덮어쓰기)
>>  # 출력을 파일에 이어쓰기
<   # 파일을 명령어 입력으로 사용
```

---

## 🌐 웹 해킹 테스트 서버 구축 (LAMP Stack)

### LAMP란?
**Linux + Apache + MariaDB + PHP** 조합으로 웹서버를 구축하는 방식

### 각 구성요소 역할
```
아파치(Apache)  → 브라우저 요청을 받아주는 웹서버
PHP            → .php 파일을 실행하는 프로그램
php-mysql      → PHP와 MariaDB를 연결해주는 통역사 (mysqli.so 모듈)
MariaDB        → 데이터를 저장하는 DB 엔진
```

> **MariaDB vs MySQL**  
> MySQL 개발자들이 오라클 인수 후 독립해서 만든 것이 MariaDB  
> 명령어/문법이 거의 동일해서 `mysql` 명령어로 MariaDB에 접속 가능

### 설치 순서 및 명령어

**1. 패키지 설치**
```bash
apt update                          # 패키지 목록 최신화
apt install apache2                 # 웹서버 설치
apt install mariadb-server          # DB 엔진 설치
apt install php php-mysql           # PHP + DB 연결 모듈 설치
```
> apache2 먼저 설치 후 php 설치해야 자동으로 아파치 모듈로 등록됨  
> (`/etc/apache2/mods-enabled/php7.4.load` 자동 생성)

**2. 설치 확인**
```bash
apt show apache2
systemctl status apache2
systemctl status mariadb
```

**3. 웹소스 다운로드 및 배포**
```bash
cd /var/www/html                              # 아파치 웹루트 폴더로 이동
wget -O webhack_test.zip https://bit.ly/43Dxh9g  # 파일 다운로드 (-O: 저장 파일명 지정)
unzip webhack_test.zip                        # 압축 해제
cd board/
ls -al
```
> `/var/www/html` 이 웹루트 폴더  
> 여기 있는 파일이 브라우저에서 접속 가능

**4. DB 세팅**
```bash
mysql -u root < webhack.sql
```
> `<` 리다이렉션으로 webhack.sql 파일을 MariaDB에 실행  
> 파일 안에 CREATE DATABASE, CREATE TABLE, INSERT 등이 들어있어  
> webhack DB와 member/board 테이블 자동 생성됨  
> 실제 데이터는 `/var/lib/mysql/webhack/` 폴더에 파일로 저장됨

**5. MariaDB root 패스워드 변경**
```bash
mysql -u root -p
```
```sql
show databases;
use mysql;
ALTER USER 'root'@'localhost' IDENTIFIED BY 'admin';
-- MariaDB 10.4 이상은: IDENTIFIED VIA mysql_native_password USING PASSWORD('admin');
FLUSH PRIVILEGES;
exit
```
> `dbconfig.php`에 패스워드가 'admin'으로 고정돼있어서 맞춰주는 것  
> 리눅스 root 비번 ≠ MariaDB root 비번 (완전히 별개!)  
> MariaDB 최초 설치 후에는 패스워드 없이 접속 가능 (초기 상태)

**6. 웹소스 권한 변경**
```bash
chown www-data:www-data -R /var/www/html/board/
ls -al
```
> 압축 해제 시 소유자가 root로 설정됨  
> 아파치 실행 계정(www-data)이 파일을 읽으려면 소유자 변경 필요  
> `-R`: 하위 모든 파일에 재귀 적용

**7. 접속 테스트**
```
브라우저: http://[우분투IP]/board/
ID/PW: DB의 member 테이블에서 확인
```

### 전체 동작 흐름
```
브라우저 요청 (http://IP/board/)
        ↓
아파치 수신 → PHP 파일 찾음
        ↓
PHP 실행 (libphp7.4.so 모듈)
        ↓
dbconfig.php 읽음 (DB 접속 정보)
        ↓
php-mysql(mysqli.so)으로 MariaDB 연결
        ↓
MariaDB가 /var/lib/mysql/webhack/ 에서 데이터 확인
        ↓
PHP가 결과를 HTML로 만들어 반환
        ↓
브라우저에 화면 출력
```

### 강사님 제공 파일 구조
```
board/
 ├── index.php         → 메인 페이지
 ├── login.php         → 로그인 화면
 ├── login_chk.php     → 로그인 처리 (DB 대조)
 ├── dbconfig.php      → DB 접속 정보 (host/user/pw/dbname)
 ├── board.php         → 게시판 목록
 ├── board_write.php   → 게시글 작성
 ├── board_show.php    → 게시글 상세
 ├── webhack.sql       → DB/테이블 생성 SQL
 ├── member.sql        → 회원 초기 데이터
 ├── stylesheet.css    → 화면 디자인
 └── webshell.php      → 웹쉘 (해킹 실습용)
```

### MariaDB 데이터 저장 위치
```
/var/lib/mysql/webhack/
 ├── member.frm   → 테이블 구조
 ├── member.MYD   → 실제 데이터 (아이디/비번 등)
 └── member.MYI   → 인덱스 (빠른 검색용)
```
> 직접 cat으로 읽으면 바이너리라 깨짐  
> 반드시 MariaDB 통해서 읽고 편집해야 함

### 주요 SQL 명령어
```sql
show databases;                          -- DB 목록
use webhack;                             -- DB 선택
show tables;                             -- 테이블 목록
select * from member;                    -- 전체 조회
DESC member;                             -- 테이블 구조 확인

INSERT INTO member VALUES (2, 'user1', '1234', 'user1', 1, NOW());
-- no, id, pw, name, level, regdate 순서

UPDATE member SET pw='newpw' WHERE id='admin';   -- 수정
DELETE FROM member WHERE id='user1';             -- 삭제
```
> level 9 = 관리자 / level 1 = 일반 유저  
> 웹해킹 실습에서 level 조작을 통한 권한 상승 실습 예정

### 트러블슈팅
| 문제 | 원인 | 해결 |
|---|---|---|
| ALTER USER 문법 오류 | MariaDB 버전 10.3 (강사님은 10.4↑) | `IDENTIFIED BY 'admin'` 문법 사용 |
| MariaDB 재시작 실패 | --skip-grant-tables 후 ib_logfile 충돌 | ib_logfile0, ib_logfile1 삭제 후 재시작 |
| 로그인 실패 | member 테이블 데이터 없음 | member.sql 임포트 |
