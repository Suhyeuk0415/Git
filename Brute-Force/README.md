Markdown
# SSH Brute Force 방어 실습 (iptables recent 모듈 활용)

## 1. 실습 목적
- 서버의 SSH 포트(22)를 노리는 무차별 대입 공격(Brute Force)을 탐지하고 자동으로 차단하는 방화벽 룰 구축.
- iptables의 `recent` 모듈을 활용하여 '특정 시간 내 일정 횟수 이상'의 비정상적인 접근을 제어하는 임계치 기반 차단 로직 적용.

## 2. 실습 환경
- **방어 서버 (Target):** Rocky Linux (192.168.62.130)
- **공격 서버 (Attacker):** Kali Linux (192.168.62.131)
- **공격 툴:** Hydra

## 3. 방어 방화벽(iptables) 룰 설정
60초 이내에 4회 이상 새로운 SSH 연결을 시도하는 IP를 공격자로 간주하고, 로그 기록 후 접속을 완전히 끊어버리도록(DROP) 설정.

```bash
# 1. 60초 내 4회 이상 접속 시도 시 [SSH_BLOCK] 로그 남기기
sudo iptables -A INPUT -p tcp --dport 22 -m state --state NEW -m recent --update --seconds 60 --hitcount 4 --name SSH_BRUTE -j LOG --log-prefix "[SSH_BLOCK] "

# 2. 로그를 남긴 후 패킷을 버림(DROP)하여 통신 차단
sudo iptables -A INPUT -p tcp --dport 22 -m state --state NEW -m recent --update --seconds 60 --hitcount 4 --name SSH_BRUTE -j DROP

# 3. 새로운 SSH 접속 시도자의 IP를 SSH_BRUTE 명단에 등록 (카운트 시작)
sudo iptables -A INPUT -p tcp --dport 22 -m state --state NEW -m recent --set --name SSH_BRUTE
4. 공격 테스트 (Hydra)
Kali Linux에서 5개의 임의 비밀번호가 담긴 사전 파일(pass.txt)을 생성한 후, 다중 로그인을 시도.

Bash
# 사전 파일(pass.txt)을 이용해 Target으로 SSH 로그인 시도
hydra -l admin -P pass.txt ssh://192.168.62.130
5. 실습 결과 및 트러블슈팅
공격 테스트 결과, 3번째 로그인 시도까지는 서버가 정상적으로 반응했으나 4번째 시도부터 iptables 임계치 룰이 발동하여 공격자(Kali)의 터미널이 멈추고 통신이 완전히 차단(Timeout)되는 것을 확인함.

💡 트러블슈팅 및 로그 분석 포인트

공격 옵션 실수 해결: 초기에 -p (소문자) 옵션을 사용하여 여러 비밀번호를 입력했을 때, 툴이 이를 1개의 긴 비밀번호로 인식하여 임계치(4회)를 넘지 못하는 문제가 있었음. 이를 -P (대문자) 옵션과 텍스트 사전 파일(pass.txt) 조합으로 수정하여 정상적인 다중 접속 공격 테스트를 구현함.

방화벽 로그 폭주 현상 분석 (TCP Retransmission): 방어 서버의 실시간 로그(tail -f /var/log/messages) 모니터링 중 [SSH_BLOCK] 로그가 화면을 덮을 정도로 대량 발생하는 현상 확인. 이는 방화벽이 패킷을 REJECT(거절)가 아닌 DROP(무시) 처리했기 때문에 발생한 정상적인 현상임. 공격자 측은 서버로부터 응답을 받지 못해 패킷을 지속적으로 재전송(TCP Retransmission)했고, 방화벽은 이 재전송 패킷들까지 모두 차단해 내며 로그를 반복 기록했음을 원리적으로 이해함.

📸 실습 결과 화면
Kali Linux (차단되어 멈춘 화면)
![Kali Linux](https://github.com/user-attachments/assets/53ab08f0-c7cd-4b71-8197-fcb1f61edf5c)



Rocky Linux (로그 도배 화면)
![Rocky Linux](https://github.com/user-attachments/assets/7b5a9a41-a41c-4b72-b6e1-222ebc33d63a)
