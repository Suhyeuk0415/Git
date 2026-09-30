# 🛡️ [Network Security] TCP SYN Flood 공격 시뮬레이션 및 커널 레벨 방어(SYN Cookie) 실습

## 1. 프로젝트 개요
* **목적:** TCP 3-Way Handshake의 취약점을 이용한 SYN Flood 공격 원리 검증 및 커널 파라미터 조작을 통한 방어 기제(SYN Cookie) 적용
* **실습 환경 구성**
  * **Attacker (Kali Linux):** `hping3` 공격 도구 사용 (CLI 환경)
  * **Victim (Rocky Linux):** `192.168.62.130` (80번 포트 웹 서버 구동)
  * **Network:** NAT 환경

---

## 2. 공격 수행 및 자원 고갈 상태 분석

### ① 공격 전 (정상 상태)
* 희생자(Victim) 환경에서 80번 포트에 파이썬 웹 서버(`python3 -m http.server 80`)를 구동함.
* 공격자(Kali) 장비에서 희생자 서버로 `curl` 요청 시 `HTTP/1.0 200 OK` 응답이 즉각적으로 반환되는 정상 동작 상태를 확인함.
<img width="647" height="177" alt="image" src="https://github.com/user-attachments/assets/a6925a49-48be-4edd-b17d-8822513f9206" />


### ② SYN Flood 공격 실행 (IP Spoofing)
* 공격자 환경에서 `hping3` 도구를 이용하여 타겟 서버의 80번 포트로 대량의 SYN 패킷을 전송함.
* 타겟 서버의 백로그 큐를 고갈시키기 위해 출발지 IP를 무작위로 위장(`--rand-source`)하고 최고 속도(`--flood`)로 패킷을 쏟아부어 공격을 시도함.
<img width="666" height="139" alt="image" src="https://github.com/user-attachments/assets/b7c82365-3288-432e-9166-073650c77925" />


### ③ 공격 후 (서버 자원 마비 증명)
* 공격 진행 중 희생자 서버의 네트워크 큐 상태(`netstat -antp | grep SYN_RECV`)를 모니터링함.
* 분석 결과, 무작위 IP로부터 수신된 반쪽짜리 연결인 `SYN_RECV` (Half-Open) 대기 상태가 큐에 꽉 찬 것을 확인함.
* 과도한 공격 패킷 처리로 인해 서버 터미널에 `kernel:watchdog: BUG: soft lockup - CPU#1 stuck!` 경고가 출력되며 서버 자원이 고갈되고 마비됨.
<img width="881" height="167" alt="image" src="https://github.com/user-attachments/assets/e1213eec-8347-499e-bc87-aeed3838532b" />


---

## 3. 방어 기제 적용 결과 (최종 목표)

* 백로그 큐를 무력화하는 공격을 방어하기 위해 희생자 서버의 커널 설정에서 **SYN Cookie**를 활성화(`sysctl -w net.ipv4.tcp_syncookies=1`)함.
* 설정 적용 후 공격자의 강력한 SYN Flood 폭격이 지속되는 상황에서도, 서버가 이를 튕겨내고 정상 클라이언트의 `curl` 요청에 **`HTTP/1.0 200 OK` 응답을 정상적으로 반환하며 서비스 가용성(Availability)을 완벽히 방어**해 냄.
<img width="718" height="50" alt="image" src="https://github.com/user-attachments/assets/2799d4e5-f450-4f48-b3df-e3acfc7a1e65" />
<img width="588" height="171" alt="image" src="https://github.com/user-attachments/assets/16e4db04-b6bd-4514-a68c-d004170a14ee" />



---

## 4. 실습 중 발생한 이슈 및 트러블슈팅 (Troubleshooting)

실습 과정 중 발생한 환경적 제약과 휴먼 에러를 분석하고 극복함.

### 🔴 Issue 1: CLI 환경에서의 다중 세션 모니터링 문제
* **증상:** 희생자 서버(Rocky Linux)가 GUI가 지원되지 않는 CLI 전용 환경이어서, 웹 서버를 실행 중인 화면에서는 실시간 네트워크 상태(`netstat`) 확인 명령어를 칠 수 없음.
* **해결:** 가상 콘솔 전환 단축키(`Alt+F2`)를 활용하여 새로운 백그라운드 로그인 세션을 띄움. 웹 서비스 구동 화면과 네트워크 큐 모니터링 화면을 분리하여 공격 상태를 실시간으로 교차 검증하는 데 성공함.

### 🔴 Issue 2: IP 위장 공격 시 타겟팅 오류 발생
* **증상:** Kali 환경에서 `hping3` 공격 명령어 실행 후에도 타겟 서버의 연결 대기 큐(`SYN_RECV`)에 아무런 부하가 걸리지 않음.
* **원인:** 명령어 입력 시 타겟 IP 주소의 3번째 옥텟에 오타(`192.168.130`)를 내어 패킷이 엉뚱한 라우팅 경로로 유실됨.
* **해결:** 타겟 서버 큐에 반응이 없는 것을 본 즉시 IP 대역 설정을 검토함. 타겟 주소를 `192.168.62.130`으로 올바르게 정정하여 공격을 재수행하고, 성공적으로 서버 자원 고갈 상태를 유도해냄.
