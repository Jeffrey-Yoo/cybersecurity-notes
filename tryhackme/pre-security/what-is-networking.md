# What is Networking? — TryHackMe

**날짜:** 2026-04-04
**Room:** https://tryhackme.com/room/whatisnetworking

## 핵심 개념

### 네트워크란?
- 기기 2개 이상이 연결되면 네트워크
- 인터넷 = 작은 네트워크들이 모인 거대한 네트워크

### IP 주소 (집 주소처럼 바뀔 수 있음)
- 4개의 옥텟으로 구성 (0~255)
- 사설 IP: 내부용 (예: 192.168.1.x)
- 공인 IP: 인터넷용, 통신사(ISP)가 부여
- IPv4: 약 43억 개 한계 → IPv6로 전환 중

### MAC 주소 (주민번호처럼 고정)
- 공장에서 부여되는 고유 번호
- 앞 6자리 = 제조사 식별
- 스푸핑(위장) 가능 → 보안 취약점

### Ping
- 명령어: ping [IP 또는 URL]
- ICMP 프로토콜로 연결 상태 확인
- 응답 시간(ms)으로 네트워크 상태 파악

## 실습 메모
- 호텔 Wi-Fi에서 MAC 스푸핑으로 방화벽 우회하는 실습
- MAC 주소만으로 인증하면 위험하다는 걸 배움

## 새로 알게 된 용어
- Octet: IP 주소의 각 숫자 덩어리
- ICMP: Internet Control Message Protocol
- ISP: Internet Service Provider

## 궁금한 점
- MAC 스푸핑이 실무에서 얼마나 자주 발생할까?
