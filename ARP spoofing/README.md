# 🛡️ [Network Security] ARP Spoofing 기반 MITM 공격 및 스니핑 실습

## 1. 프로젝트 개요
* **목적:** ARP 프로토콜의 취약점(인증 부재)을 이용한 중간자 공격(MITM) 원리 검증 및 평문 통신(HTTP)의 스니핑 위험성 분석
* **실습 환경 구성**
  * **Attacker (Kali Linux):** `192.168.62.131` (MAC: `00:0c:29:a9:39:d8`)
  * **Victim (Rocky Linux):** `192.168.62.130` (MAC: `00:0c:29:c4:dd:2e`)
  * **Gateway (Router):** `192.168.62.2`

---

## 2. 공격 수행 및 ARP 테이블 변조 분석

### ① 공격 전 (정상 상태)
* 희생자(Victim) 환경에서 라우팅 테이블 및 ARP 캐시 테이블을 확인한 결과, 게이트웨이 IP에 정상적인 라우터의 MAC 주소가 매핑되어 있음.
<img width="930" height="457" alt="image" src="https://github.com/user-attachments/assets/55c8c7b8-b84f-486e-8e37-5e0dfa3123f3" />


### ② ARP Poisoning 공격 실행
* 공격자 환경에서 패킷 포워딩(`net.ipv4.ip_forward=1`)을 활성화하여 통신 단절을 방지함.
* `arpspoof` 도구를 이용하여 게이트웨이와 희생자 양측에 위조된 ARP Reply 패킷을 지속적으로 브로드캐스트하여 세션을 가로챔.
<img width="793" height="556" alt="image" src="https://github.com/user-attachments/assets/b428b3c6-8a6e-4714-b8cc-69b8218686e3" />


### ③ 공격 후 (MAC 주소 변조 성공)
* 희생자 PC의 ARP 테이블을 재확인한 결과, 게이트웨이 IP(`192.168.62.2`)의 MAC 주소가 공격자(Kali)의 MAC 주소로 성공적으로 덮어씌워진(Poisoned) 것을 확인함.
<img width="647" height="233" alt="image" src="https://github.com/user-attachments/assets/8b51fd08-67b8-4a99-bd24-f63296d3c057" />


---

## 3. Wireshark 스니핑 결과 (최종 목표)

* 패킷 흐름이 공격자를 경유하도록 네트워크 구조를 변조한 뒤, Wireshark를 통해 `HTTP POST` 요청을 캡처함.
* 분석 결과, 희생자가 전송한 웹 로그인 데이터가 암호화되지 않고 평문(Plaintext)으로 전송되어 **계정 정보(`uname=hacker`, `pass=1234`)가 그대로 노출되는 치명적 취약점**을 확인함.
<img width="937" height="812" alt="image" src="https://github.com/user-attachments/assets/859fe71e-f4d7-4887-97e8-41494cfbeb26" />


---

## 4. 실습 중 발생한 이슈 및 트러블슈팅 (Troubleshooting)

실습 과정 중 가상 환경(VMware)의 특성으로 인해 발생한 네트워크 통신 이슈를 분석하고 우회 기법을 적용함.

### 🔴 Issue 1: 블랙홀 라우팅 (통신 단절)
* **증상:** 스푸핑 공격 직후 희생자 PC에서 `curl: (7) Failed to connect` 에러가 발생하며 외부 인터넷이 전면 단절됨.
* **원인:** 공격자(Kali)가 패킷을 수신 후 게이트웨이로 전달(Forwarding)하지 않아 발생.
* **해결:** 커널 변수 제어(`sysctl -w net.ipv4.ip_forward=1`)를 통해 패킷 릴레이를 활성화하여 정상 해결함.

### 🔴 Issue 2: VM 가상 라우터 보안 정책에 의한 DNS 및 패킷 드랍
* **증상:** 패킷 릴레이 활성화 후에도 희생자 환경에서 외부 사이트 대상 `POST` 패킷 전송 시 타임아웃(28) 및 도메인 해석 실패(6) 에러 지속 발생.
* **원인:** VMware 가상 망(NAT) 환경의 보안 기제가 출발지 IP와 MAC이 불일치하는 스푸핑 패킷을 비정상 트래픽으로 간주하여 강제 Drop 처리함.
* **해결 (Local Server Bypass):** 외부 인터넷망을 거치지 않도록 공격자(Kali) 장비 자체에 임시 파이썬 웹서버(`python3 -m http.server 80`)를 구동함. 이후 희생자가 공격자 IP로 직접 `POST` 패킷을 쏘되, HTTP 헤더의 Host 필드만 실제 웹사이트로 위조 전송(`-H "Host: testphp.vulnweb.com"`)하여 스니핑을 완수함.

<img width="946" height="191" alt="image" src="https://github.com/user-attachments/assets/350659e9-5d20-4bce-bb9c-cb21e1dc0dc3" />

> 외부망 차단을 우회하고 내부망에서 정상적으로 패킷 통신(501 응답)을 이끌어낸 모습.
