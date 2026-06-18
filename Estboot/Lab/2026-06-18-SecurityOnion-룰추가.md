# 2026-06-18 — Security Onion Detections 커스텀 룰 추가 + 동기화

오늘은 이론(NGIPS 요건·Snort/Suricata 룰 옵션 분석)이 메인이라 핸즈온은 가벼웠다.
랩에서 한 일은 하나 — 어제 도입한 **Security Onion(미러 NSM)**의 SOC 콘솔에서 **Detections에 커스텀 탐지 룰을 직접 넣고 동기화**해본 것.

> 이론 정리는 `../IT-Infra-Secu/Security-Solutions.md`의
> "Snort/Suricata 룰 옵션 심화", "NGIPS 요건", "Security Onion — Detections에 커스텀 룰 직접 넣기" 참고.

---

## 한 일

```
SOC 콘솔 좌측 Detections 메뉴 진입
 → 룰(시그니처) 본문 입력
 → 톱니바퀴(⚙ 동기화/업데이트) 클릭
 → 센서에 배포·반영될 때까지 최대 ~15분 대기
 → 그 후 탐지에 적용됨
PCAP 메뉴에서 룰에 걸린 패킷의 실제 이동(흐름) 확인
```

- 6-15~16일 pfSense Snort에 커스텀 룰 넣던 것과 **문법은 동일**(content·sid·hex/hash). 넣는 위치만 pfSense GUI → Security Onion Detections로 바뀐 것.
- 룰 본문은 Snort/Suricata 룰 한 줄 형식 그대로 — 헤더(액션·프로토콜·출발지/목적지) + 옵션(msg·content·검색키워드·sid/rev).

## ⚠️ 삽질·주의

- **룰 넣자마자 안 잡힌다** → 동기화(톱니바퀴)를 누르고 **최대 15분**까지 기다려야 센서에 반영된다.
  "룰 넣었는데 왜 alert가 안 떠?"는 대개 룰 오류가 아니라 **아직 동기화 전**.
  → pfSense Snort는 룰 리로드가 비교적 빨랐는데, Security Onion은 **분산 센서 구조**라 매니저→센서 배포 지연이 더 크다.
- 적용 확인은 **PCAP**으로 — 룰에 걸린 게 진짜인지 실제 패킷 흐름까지 봐야 한다.
  (Alerts → Hunt → **PCAP** → Cases 파이프라인의 증거 단계)
- ⚠️ 어제 실설치가 매끄럽지 않아 **실설치 재시도는 백로그에 그대로 둔다** — 오늘은 콘솔 동작·룰 반영 흐름 확인 수준.

## 다음

- [ ] Security Onion 실설치 재시도(ISO 재다운로드) → `so-status`(컨테이너 running)·`ifconfig` 모니터링 NIC RX 증가 확인 → SPAN 물려 미러 검증
- [ ] Kali 공격 → **막는 쪽(Snort 인라인 drop) / 보는 쪽(Security Onion Alerts)** 동시 대조
- [ ] Detections 커스텀 룰을 Kali 트래픽으로 실제 트리거시켜 PCAP까지 확인
