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
sudo apt install samba samba-client
sudo useradd sambauser
sudo usermod -aG samba sambauser
sudo smbpasswd -a sambauser         # 삼바 전용 패스워드 (Linux 패스워드와 별개)
sudo smbpasswd -e sambauser         # 계정 활성화

sudo mkdir -p /samba
sudo chown sambauser:samba /samba
sudo chmod 770 /samba

sudo vi /etc/samba/smb.conf
# 맨 아래 추가
[samba]
   path = /samba
   browsable = yes
   writeable = yes
   guest ok = no
   valid user = @samba

sudo systemctl restart smbd
sudo systemctl enable smbd

# 접속 테스트
smbclient //IP/samba -U sambauser
```

```bash
# 클라이언트 측 (마운트)
sudo apt install cifs-utils -y
sudo mkdir -p /mnt/samba
sudo mount -t cifs //서버IP/samba /mnt/samba -o username=sambauser,password=패스워드
df -h | grep samba                  # 마운트 확인
```

> Samba 주요 포트: 445 (SMB Direct), 139 (NetBIOS 구버전 호환)

---

## DNS 서버 구축 (BIND9)

```bash
apt update
apt install bind9 bind9utils

# named.conf 수정
vi /etc/bind/named.conf
# 맨 아래에 추가
include "/etc/bind/named.conf.external-zones";

# 존 파일 생성 및 설정
vi /etc/bind/maxoverpro.org.zone
```

존 파일 내용:
```
$TTL 86400
@   IN  SOA  ns.maxoverpro.org. root.maxoverpro.org. (
    2026051101  ; Serial
    3600        ; Refresh
    1800        ; Retry
    604800      ; Expire
    86400 )     ; Negative Cache TTL

    @       IN  NS   ns.maxoverpro.org.
    ns      IN  A    192.168.1.90
    web01   IN  A    192.168.1.50
    web02   IN  A    192.168.1.51
```

```bash
# 검증 및 재시작
named-checkconf
named-checkzone maxoverpro.org /etc/bind/maxoverpro.org.zone
systemctl restart bind9

# 테스트
dig @192.168.1.90 web01.maxoverpro.org
```

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
