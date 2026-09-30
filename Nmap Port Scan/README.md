iptables를 이용한 Nmap 포트 스캔 탐지 및 로그 분석
1. 실습 배경 및 목적

이론으로만 배우던 포트 스캔 기법들이 실제 방화벽(iptables) 커널 로그에 어떻게 남는지 직접 눈으로 확인해보고 싶어 진행한 개인 랩입니다.

일반적인 스캔(TCP Connect)과 흔적을 숨기려는 스텔스 스캔(SYN Scan)의 패킷 흐름 차이를 로그를 통해 직접 비교 분석했습니다.

2. 환경 구성 및 트러블슈팅(Troubleshooting)

초기 구성 이슈: 처음엔 공격자(Kali)와 방어자(Ubuntu)로 VM을 2대 구성했으나, 로컬 PC 리소스 부족으로 인해 지속적인 커널 패닉 및 프리징 현상이 발생했습니다.

해결 방법 (Plan B):

리소스 확보를 위해 Rocky Linux 10 환경을 GUI에서 텍스트 모드(CLI, Multi-user target)로 전환했습니다.

네트워크 충돌 변수를 없애기 위해 로컬호스트(127.0.0.1) 기반의 자가 스캔(Self-scan) 방식으로 변경하여 안정적으로 실습을 완료했습니다.

3. 탐지 룰 설정 (iptables)
서버의 22번 포트(SSH)로 들어오는 비정상적인 접근을 커널 레벨에서 로깅하도록 룰을 추가했습니다.

Bash
sudo iptables -A INPUT -p tcp --dport 22 -j LOG --log-prefix "[PORT_SCAN_LOCAL] "
4. 스캔 로그 분석 결과
<img width="1362" height="186" alt="image" src="https://github.com/user-attachments/assets/a2983d53-87d8-4678-b3eb-455e28a3c791" />


테스트 1: 일반 스캔 (TCP Connect Scan, -sT)

로그 확인: SYN -> ACK -> ACK RST 순서로 패킷이 찍힘.

분석: 대상 포트와 정상적인 3-Way Handshake를 끝까지 맺고 연결을 끊는 방식입니다. 연결이 완전히 성립되었기 때문에 애플리케이션 단에도 기록이 남습니다.

테스트 2: 스텔스 스캔 (TCP SYN Scan, -sS)

로그 확인: SYN -> RST 순서로만 패킷이 찍힘.

분석: 이른바 Half-open 스캔. 포트가 열려있는지 확인하기 위해 SYN을 보내고, 서버가 응답하면 ACK를 보내지 않고 RST로 바로 강제 종료해버리는 것을 확인했습니다.

인사이트: 이렇게 하면 서버의 애플리케이션(SSH 등) 로그에는 접근 기록이 남지 않아 추적이 어렵지만, 커널 레벨의 방화벽(iptables) 로그에는 이 비정상적인 패킷 흐름이 모두 탐지된다는 것을 직접 검증했습니다.
