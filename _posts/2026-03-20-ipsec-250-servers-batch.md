---
title: "250대 서버 IPSec 정책을 혼자 바꿔야 했던 날 — 배치파일과 멀티 RDP"
date: 2026-03-20 19:30:00 +0900
categories: [인프라 실무]
tags: [windows, ipsec, rdp, 자동화, 배치파일, 트러블슈팅, 대규모작업]
---

## 상황

폐쇄망 환경에서 운영 중인 Windows 서버가 약 250대 있다.

외부 접근이 원천 차단된 환경이라 RDP 접속도 아무나 못 한다. 방화벽 + IPSec 필터 정책으로 **지정된 워크스테이션 IP에서만** RDP 접근이 허용되는 구조다.

그런데 노후 워크스테이션을 불용 처리하고 새 장비로 교체하는 과정에서 정보 공유가 제대로 안 됐다. 결과적으로 **RDP 전용 워크스테이션의 IP가 바뀌었고**, 250대 서버 전체의 IPSec 허용 IP를 갱신해야 하는 상황이 됐다.

---

## IPSec 필터 정책이 뭔데

Windows IPSec은 네트워크 레벨에서 특정 IP/포트 조합만 통신을 허용하거나 차단하는 정책이다. 방화벽보다 낮은 레이어에서 동작한다.

RDP(3389 포트) 접근을 이렇게 제어하고 있었다:

```
IPSec 필터 규칙:
- 출발지 IP: [워크스테이션 IP] / 목적지: any / 포트: 3389 → 허용
- 그 외 모든 3389 트래픽 → 차단
```

이 정책이 서버 250대에 각각 로컬로 적용돼 있었다. 워크스테이션 IP가 바뀌면 → 새 IP로 룰을 바꿔줘야 → RDP가 다시 된다.

---

## 수동으로 하면?

서버 한 대당 작업:

1. RDP 접속
2. `secpol.msc` 또는 `netsh` 실행
3. 기존 IP 룰 삭제
4. 신규 IP 룰 추가
5. 정책 적용 확인
6. 다음 서버로 이동

250대 × 5분 = **1,250분 = 약 20시간**

혼자 다 하면 며칠이 걸린다. 그것도 실수 없이.

---

## 배치파일로 해결

`netsh ipsec static` 명령어로 IPSec 정책을 커맨드라인에서 제어할 수 있다. 배치파일로 만들면 각 서버에서 실행만 하면 된다.

```batch
@echo off

REM ============================
REM  IPSec RDP 접근 IP 변경 스크립트
REM  구 IP → 신 IP 교체
REM ============================

SET OLD_IP=192.168.1.100
SET NEW_IP=192.168.1.200
SET POLICY_NAME=RDP_ACCESS_POLICY
SET FILTER_NAME=RDP_FILTER

REM 기존 필터 삭제
netsh ipsec static delete filter ^
    filterlist="%FILTER_NAME%" ^
    srcaddr=%OLD_IP% ^
    dstaddr=any ^
    protocol=TCP ^
    dstport=3389

REM 신규 필터 추가
netsh ipsec static add filter ^
    filterlist="%FILTER_NAME%" ^
    srcaddr=%NEW_IP% ^
    dstaddr=any ^
    protocol=TCP ^
    dstport=3389 ^
    description="RDP access from management workstation"

REM 정책 재적용
netsh ipsec static set store location=local
netsh ipsec static exportpolicy file=backup_%DATE%.ipsec

echo 완료: %COMPUTERNAME%
```

이걸 각 서버에 넣고 실행하면 된다.

---

## RDCMan으로 순차 실행

배치파일은 만들었는데, 250대에 어떻게 배포하고 실행할 건지가 문제였다.

폐쇄망이라 중앙 배포 솔루션(SCCM 같은 것)을 쓸 수 없는 환경이었다. 그래서 선택한 게 **Microsoft RDCMan(Remote Desktop Connection Manager)** — 오픈소스 멀티 RDP 관리 도구다.

사용 방법:
1. 250대 서버 목록을 RDCMan에 그룹으로 등록
2. 공통 자격증명 설정 (도메인 계정)
3. 배치파일을 각 서버에 미리 복사 (공유 폴더 경유)
4. RDCMan으로 서버 순차 접속 → 배치파일 실행 → 다음 서버

동시 실행 기능을 못 찾아서 결국 순차로 진행했는데, 그래도 접속-실행-확인-다음 사이클이 자동화되니까 속도가 달랐다.

---

## 테스트를 먼저 한 이유

사실 과거에 IPSec 정책 잘못 건드렸다가 **서버에 RDP가 아예 안 되는 상황**을 만든 적이 있다.

IPSec이 RDP를 차단해버리면 원격으로 복구할 방법이 없다. 콘솔 케이블 들고 서버실 가거나, 물리적으로 접근하거나. 둘 다 번거롭고 창피하다.

그래서 이번엔 **사무실에서 가장 가까운 서버 1대로 먼저 테스트**했다. 배치파일 실행 → 신규 IP로 RDP 접속 확인 → 이상 없으면 전체 진행. 만약 잘못됐으면 바로 서버실 달려가서 콘솔로 롤백할 수 있는 거리.

단순한 방법이지만, 고가용성이 요구되는 환경에서 변경 작업은 **항상 롤백 플랜이 있어야 한다**.

---

## 결과

- 예상 수동 작업 시간: 약 8~9시간 (현실적으로 실수 포함하면 더)
- 실제 소요 시간: **3시간 이내**

배치파일로 작업 표준화 + RDCMan으로 접속 효율화. 단순한 조합이었지만 효과는 명확했다.

---

## 기술적으로 더 보면

**왜 netsh ipsec static인가**

Windows IPSec은 GUI(`secpol.msc`)로도 설정 가능하지만, CLI로 다루면 스크립트화가 가능하다. `netsh ipsec static` 명령어 체계를 쓰면:

```batch
REM 정책 목록 확인
netsh ipsec static show policy all

REM 필터 목록 확인  
netsh ipsec static show filterlist all

REM 현재 적용된 정책 확인
netsh ipsec static show rule name=all
```

작업 전에 현재 상태를 파일로 백업해두는 것도 중요하다:

```batch
netsh ipsec static exportpolicy file=ipsec_backup.ipsec
```

문제가 생기면 이걸로 즉시 롤백 가능하다:

```batch
netsh ipsec static importpolicy file=ipsec_backup.ipsec
```

**RDCMan 대신 쓸 수 있는 것들**

지금 같은 상황이라면 추가로 고려할 수 있는 방법들:

- **PowerShell Remoting (WinRM)**: 도메인 환경이라면 `Invoke-Command`로 원격 일괄 실행 가능. 단, WinRM이 활성화돼 있어야 함
- **PsExec**: Sysinternals 도구, 폐쇄망에서도 쓸 수 있음
- **Ansible for Windows**: WinRM 기반으로 Windows 서버 일괄 관리 가능

폐쇄망 + 보안 제약 환경에선 선택지가 좁아지는데, 그 안에서 쓸 수 있는 걸 조합하는 게 현실적인 인프라 운영이다.

---

## 한 줄 요약

> 반복 작업은 스크립트로, 다중 접속은 도구로. 단순한 조합이 시간을 아낀다.
