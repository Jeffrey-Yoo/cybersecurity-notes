# 🐧 Linux 시스템 관리

---

## 기본 명령어

| 명령어 | 설명 |
|---|---|
| `cd` | Change Directory. `..` (상위), `/` (최상위) |
| `pwd` | 현재 위치 출력 |
| `ls -al` | 목록 확인. `-a` 숨김파일, `-l` 상세정보 |
| `cat` | 파일 읽기 |
| `tail` | 파일 하단부 읽기. `-f` 옵션으로 실시간 업데이트 확인 |
| `cp` | 파일 복사. `cp 경로 파일` |
| `mv` | 파일 이동 및 이름 변경 |
| `rm` | 파일 삭제. `rmdir` = 디렉터리 삭제 |
| `mkdir` | 디렉터리 생성. `touch` = 파일 생성 |
| `find` | 파일 찾기. `find 경로 명령어 파일` |
| `grep` | 특정 단어 찾기 |
| `wc` | word count. 줄, 단어, 용량 수 확인 |
| `hostname` | 호스트 네임 확인. `hostnamectl set-hostname 이름` 으로 변경 |
| `df -h` | 파티션 정보 확인. `-h` = 읽기 편하게 출력 |
| `top` | 프로세스 현황 |
| `chown` | 오너 체인지 (권한 변경) |
| `chmod` | Change mode (변경 모드) |
| `\|` | 파이프. 이전 입력 값에 추가로 넣을 명령어 |

---

## 네트워크 명령어

```bash
ip a                        # IP 주소 확인
ping IP -t                  # 패킷 송수신 확인 (Windows)
netstat -an                 # 네트워크 상태 확인
  # EST 개수 = 현재 연결 수 (식당 식탁 수처럼 OS에 설정된 Socket 연결 값까지 가능)
  # Web Server 기준 약 3000개
nslookup 도메인             # 도메인 IP 조회
dig @DNS서버IP 도메인       # DNS 질의 테스트
```

---

## 계정 및 권한 관리

```bash
useradd -m [계정명]          # /home 아래 홈 디렉토리 포함 계정 생성
passwd [계정명]              # 계정 패스워드 설정
passwd -S [계정명]           # 계정 패스워드 상태 확인
cat /etc/passwd              # 시스템 전체 계정 정보 확인
usermod -aG 그룹 유저        # 유저를 그룹에 추가 (그룹 권한 상속)
chown 소유자:그룹 경로        # 폴더 소유권 변경
```

### 패스워드 정책 설정

```bash
vi /etc/login.defs          # 155번째 줄 근처
```

| 항목 | 설명 |
|---|---|
| PASS_MAX_DAYS | 패스워드 유효기간 |
| PASS_MIN_DAYS | 패스워드 재설정 최소 간격 |
| PASS_MIN_LEN | 패스워드 최소 길이 |
| PASS_WARN_AGE | 만료 전 경고 표시 일수 |

> ⚠️ 실무 메인 서버에서는 만료 정책을 적용하지 않는 경우가 많음
> 패스워드 만료 알림이 WAS 서버와 로그인 서버 사이 인증 연결에 장애를 일으킬 수 있기 때문

---

## 기본 보안 원칙

```
1. root 계정 사용 X
2. 비밀번호 설정 확인
3. Port 번호 변경
   → 기본 포트는 누구에게나 공개됨
   → 포트 변경만으로 자동화 스캔 감지 우회 가능
```

---

## SSH 포트 변경

```bash
vi /etc/ssh/sshd_config     # 주석(#) 제거 후 포트 번호 변경
systemctl restart ssh       # 서비스 재시작
netstat -an                 # 포트 변경 확인
```

### ⚠️ Ubuntu 22.04+ — `ssh.socket`이 sshd_config를 덮어쓴다

최신 우분투는 SSH가 **소켓 액티베이션(`ssh.socket`)**으로 뜬다. 포트를 소켓이 관리하므로 **sshd_config의 Port를 바꿔도 무시된다.**

```bash
# 방법 A) 소켓 오버라이드
systemctl edit ssh.socket
#   [Socket]
#   ListenStream=          ← 기존값 비우고
#   ListenStream=원하는포트  ← 새 포트
systemctl daemon-reload     # ⚠️ demon 아님 — daemon 철자
systemctl restart ssh.socket

# 방법 B) 소켓을 끄고 전통 방식으로 (이러면 sshd_config의 Port가 먹는다)
systemctl disable --now ssh.socket
systemctl enable --now ssh

ss -tlnp | grep ssh         # 실제 리스닝 포트 확인 (누가 그 포트를 잡았나)
```

> "분명 sshd_config 고쳤는데 포트가 안 바뀐다"의 범인이 `ssh.socket`. 포트가 안 먹으면 `ss -tlnp`부터 본다.

### vi 편집기 명령어

| 명령어 | 설명 |
|---|---|
| ESC | 명령 모드 전환 |
| i / a | 입력 모드 (i: 현재 위치, a: 한 칸 오른쪽) |
| o | 줄 추가 |
| x | 문자 삭제 |
| dd | 줄 삭제 |
| u | 이전 실행 취소 |
| Ctrl+R | 재실행 |
| :wq! | 강제 저장 & 종료 |
| :q! | 강제 종료 |

---

## SSH 원격 접속

```bash
# Linux → Linux
ssh -i 키파일.pem 유저@IP

# Windows → Linux (PuTTY)
# 실제 서버엔 모니터가 없어서 원격 접속이 기본
```

### SFTP (FileZilla)

```
Filezilla 다운로드 → 사이트 관리자 → 서버 정보 입력 → 연결 → 파일 송수신
```

> ⚠️ root 계정으로 로그인하면 모든 권한을 가져옴
> 실무에서는 반드시 일반 user 계정으로 접속

---

## 패키지 관리

```bash
apt update                  # 패키지 목록 최신화
apt install 패키지명         # 패키지 설치
systemctl status 서비스명    # 서비스 상태 확인
systemctl restart 서비스명   # 서비스 재시작
systemctl enable 서비스명    # 부팅 시 자동 시작 등록
```

---

## 웹 해킹 테스트 서버 구축

```bash
apt update
apt install apache2
apt install mariadb-server
apt install php php-mysql

# 웹소스 배포
cd /var/www/html
wget -O webhack_test.zip https://bit.ly/43Dxh9g
unzip webhack_test.zip

# DB 세팅
mysql -u root < webhack.sql

# 웹소스 권한 변경
chown www-data:www-data -R /var/www/html/board/
```

전체 흐름:
```
브라우저 요청 → 아파치(수신) → PHP(처리) → php-mysql(연결) → MariaDB(데이터) → 화면 출력
```

---

## Samba 서버 구축 (파일 공유)

```bash
# 서버 측
apt install samba samba-client
groupadd samba
useradd -M -s /sbin/nologin sambauser   # -M 홈없음, 로그인셸 없음(파일공유 전용 계정)
usermod -aG samba sambauser
smbpasswd -a sambauser              # 삼바 전용 패스워드 (Linux 로그인 비번과 별개)
smbpasswd -e sambauser              # 계정 활성화

mkdir -p /samba
chown sambauser:samba /samba
chmod 770 /samba

vi /etc/samba/smb.conf
# 맨 아래 추가
[samba]
   path = /samba
   browseable = yes                 # ⚠️ browsable 아님 (e 들어감)
   writeable = yes
   guest ok = no
   valid users = @samba             # ⚠️ valid user 아님 — 복수형

testparm                            # ★ 적용 전 문법검사 (Unknown parameter 안 뜨면 OK)
systemctl restart smbd && systemctl enable smbd
smbclient //localhost/samba -U sambauser   # 서버 자체 접속 테스트
```

> ⚠️ **smb.conf 오타 2종** — `browsable`→`browseable`, `valid user`→`valid users`. 오타가 나도 서비스는 떠서 동작만 이상해진다. **`testparm`으로 거른 뒤 restart** (netplan generate, named-checkconf와 같은 "적용 전 문법검사" 습관).

```bash
# 클라이언트 측 (마운트)
apt install cifs-utils              # 마운트 전용 (탐색용 samba-client와 다름)
mkdir -p /mnt/samba
mount -t cifs //서버IP/samba /mnt/samba -o username=sambauser

# 비번을 명령행에 안 남기게 credentials 파일로 분리
vi /etc/samba/credentials
#   username=sambauser
#   password=비번
chmod 600 /etc/samba/credentials    # 비번 파일이니 600 필수

# 재부팅에도 살아남게 fstab 자동 마운트
vi /etc/fstab
#   //서버IP/samba  /mnt/samba  cifs  credentials=/etc/samba/credentials,uid=1000  0  0
mount -a                            # ⚠️ 재부팅 전 fstab 검증 필수
df -h | grep samba                  # 마운트 확인
```

> ⚠️ **fstab엔 `-o`를 안 쓴다** — 필드 사이는 공백, 옵션끼리는 콤마. `mount`의 `-o username=...`이 fstab에선 옵션 필드(콤마 구분)로 들어간다.

> 🔗 설정파일 들여쓰기 규칙 비교: netplan(YAML)=들여쓰기가 문법 / smb.conf(INI)=들여쓰기는 가독성용 / fstab=공백으로 필드 구분(칸 수 무관). 파서가 다르니 규칙이 다르다.

> Samba 주요 포트: 445 (SMB Direct), 139 (NetBIOS 구버전 호환)

---

## DNS 서버 구축 (BIND9)

내부 서버끼리 IP 대신 이름으로 부르게 하는 **사내 전화번호부**.

```bash
apt install bind9 bind9utils bind9-dnsutils
```

### named.conf.options — 오픈 리졸버를 막는 게 보안 포인트

```
options {
    directory "/var/cache/bind";
    allow-query     { any; };                      # 누구나 질의는 허용
    allow-recursion { 10.10.0.0/16; localhost; };  # ★ 재귀는 내부망만 (오픈 리졸버 차단)
    forwarders { 8.8.8.8; 1.1.1.1; };              # 모르는 도메인은 구글/클플로 넘김
    forward only;
    dnssec-validation no;
    listen-on { 10.10.40.10; 127.0.0.1; };
    listen-on-v6 { none; };
};
```

> ⚠️ **`allow-recursion`을 any로 열면 "오픈 리졸버"** — 남 대신 외부까지 찾아주는 서버가 되어 DNS 증폭 DDoS의 발판으로 악용된다. 질의(allow-query)는 열어도 **재귀는 내부망만**. (방화벽 "최소허용·단방향" 원칙의 DNS판)

### 존 선언 + 존 파일

```bash
vi /etc/bind/named.conf.local
# zone "lab.internal"   { type master; file "/etc/bind/db.lab.internal";  allow-update { none; }; };
# zone "maxoverpro.org" { type master; file "/etc/bind/db.maxoverpro.org"; allow-update { none; }; };
```

존 파일 예 (`/etc/bind/db.maxoverpro.org`):
```
$TTL 86400
@   IN  SOA  ns.maxoverpro.org. root.maxoverpro.org. (
    2026051101  ; Serial  ← 수정 때마다 +1 필수
    3600        ; Refresh
    1800        ; Retry
    604800      ; Expire
    86400 )     ; Negative Cache TTL

    @       IN  NS   ns.maxoverpro.org.
    ns      IN  A    10.10.40.10
    www     IN  A    10.10.10.10
```

```bash
named-checkconf                                          # 설정 문법
named-checkzone maxoverpro.org /etc/bind/db.maxoverpro.org   # 존 파일 문법
rndc reload                                              # ★ 무중단 존 재로드 (restart보다 권장)
dig @localhost www.maxoverpro.org                        # 자체 테스트
```

> ⚠️ **존 파일 두 함정**
> 1. **Serial은 수정 때마다 +1** — 안 올리면 변경이 반영 안 됨(캐시/슬레이브가 "안 바뀐 줄" 앎).
> 2. **FQDN 끝의 점(.)** — 점 있으면 완전한 이름, 없으면 뒤에 존 이름이 자동으로 붙는다. `ns.maxoverpro.org`에 점을 빼먹으면 `ns.maxoverpro.org.maxoverpro.org`가 된다.

> 🧠 DNS = 이름→IP 변환(전화번호부)**일 뿐**, 통화허가(방화벽)와 별개. **이름이 풀려도(dig 성공) 방화벽이 막으면 접속 안 됨** — 트러블슈팅 때 둘을 분리해서 본다.

---

## 부팅 순서와 보안

```
BIOS → 하드웨어 인식 및 초기화
    ↓
부트로더(GRUB) → RAM에 로드 및 실행
    ↓
커널 실행
    ↓
서비스 로드
```

### GRUB 보안 이슈

- GRUB 화면에서 단일 사용자 모드 진입 시 root 패스워드 초기화 가능
- 막으려면 GRUB 자체에 패스워드 설정 가능
- 실무에서는 잘 적용 안 함 (비번 분실 시 부팅 불가 리스크)

---

## 네트워크 IP 설정 (netplan)

```yaml
# /etc/netplan/50-cloud-init.yaml
network:
  ethernets:
    enp0s8:
      addresses:
        - 192.168.30.1/24
  version: 2
```

```bash
netplan apply
```
