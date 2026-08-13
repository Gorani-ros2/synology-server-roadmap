# 🏛️ 메인 서버 & 시놀로지 NAS 아키텍처 구축 및 기술 의사결정 기록 (Architecture & Insights)

**저장소**: [https://github.com/Gorani-ros2/synology-server-roadmap](https://github.com/Gorani-ros2/synology-server-roadmap)  
**상세 실행 로드맵 문서**: [roadmap.md](roadmap.md)  
**최종 작성일**: 2026-07-28  

> **작성 목적**: 서버 인프라 구축, 계정 권한 관리, 네트워크 및 시놀로지 연동, 엣지 프로토콜 개발 과정에서 발생한 **시행착오, 기술 선택의 배경, 대안 비교 및 핵심 의사결정 근거(Insights)**를 정돈된 구조로 기록하여 지식 자산화.

---

## 📑 목차 (Table of Contents)

1. [Chapter 1. 원격 접속 환경 및 SSH 보안 체계 수립](#chapter-1-원격-접속-환경-및-ssh-보안-체계-수립)
2. [Chapter 2. 사용자 계정 권한 격리 및 SSH 공개키 인증 시스템](#chapter-2-사용자-계정-권한-격리-및-ssh-공개키-인증-시스템)
3. [Chapter 3. 시놀로지 NAS 물리 네트워크 감지 트러블슈팅 및 10G 직결 아키텍처](#chapter-3-시놀로지-nas-물리-네트워크-감지-트러블슈팅-및-10g-직결-아키텍처)
4. [Chapter 4. 외부 협력사 계정 SFTP Chroot Jail 격리 및 리눅스 ACL 설계](#chapter-4-외부-협력사-계정-sftp-chroot-jail-격리-및-리눅스-acl-설계)
5. [Chapter 5. 엣지-서버 실시간 로봇 관제 데이터 파이프라인 (MediaMTX & PostGIS)](#chapter-5-엣지-서버-실시간-로봇-관제-데이터-파이프라인-mediamtx--postgis)
6. [Chapter 6. 100TB 스토리지 용량 분배 및 Tailscale P2P 보안 팀 공유 저장소](#chapter-6-100tb-스토리지-용량-분배-및-tailscale-p2p-보안-팀-공유-저장소)

---

## Chapter 1. 원격 접속 환경 및 SSH 보안 체계 수립

### 1.1 배경 및 과제 정의
연구실 외부망(LTE 모바일 핫스팟, 외부 출장지)에서 중앙 메인 서버(`218.150.16.158`)로 원격 접속하여 로봇 데이터를 제어해야 하는 환경이 요구되었다. 
그러나 인터넷에 노출된 기본 SSH 포트(`22번`)는 무차별 대입 공격(Brute-Force Attack) 및 무작위 포트 스캐너의 최우선 표적이 되므로, 보안성과 가용성을 동시에 확보하는 원격 접속 인프라 설계가 필수적이었다.

### 1.2 기술 옵션 비교 및 최종 선정 근거

| 구분 | 대안 A: 기본 SSH 22번 포트 사용 | 대안 B: VPN 전용망 구축 | **최종 선택: 커스텀 SSH 보안 포트(7289) + 포트 포워딩** |
| :--- | :--- | :--- | :--- |
| **특징** | 표준 포트 개방 | 서버 전단에 VPN 개설 | 비표준 고번호 포트(`7289`) 매핑 |
| **보안성** | 🔴 극도로 취약 (자동화 봇 공격 집중) | 🟢 최고 수준 보안 | 🟡 우수한 자동화 스캐닝 회피 효과 |
| **복잡도** | 🟢 설정 없음 | 🔴 초기 세팅 및 인증서 관리 복잡 | 🟢 공유기 포트 매핑 단 1회 설정 |
| **선정 근거** | - | - | **자동화 봇의 99% 이상이 기본 22번 포트만 타겟팅한다는 점에 착안, 비표준 포트(`7289`)로 재배치하여 불필요한 로그인 시도 트래픽을 원천 차단함.** |

### 1.3 세부 기술 구현 및 시행착오 해결
* **Ubuntu 24.04 LTS의 Socket-Based Activation 메커니즘 대응**:
  기존 Ubuntu 20.04 이전 버전에서는 `sshd` 서비스가 독립 프로세스로 상시 구동되어 `sudo systemctl restart ssh`만으로 변경 사항이 반영되었으나, **Ubuntu 24.04 LTS부터는 리소스 효율화를 위해 systemd 소켓 기반 호출(Socket-Based Activation)** 방식으로 동작 구조가 변경되었다.
  이를 고려하지 않고 단순 ssh 재시작만 수행할 경우 변경된 포트(`7289`)가 리스닝되지 않는 현상이 발생하여, 소켓 유닛 갱신을 수반하는 절차를 적용하였다.

```bash
# 1. SSH 설정 파일 내 비표준 포트 지정
sudo nano /etc/ssh/sshd_config  # Port 7289 설정

# 2. Ubuntu 24.04 소켓 유닛 재설정 및 적용
sudo systemctl daemon-reload
sudo systemctl restart ssh

# 3. 소켓 리스닝 상태 최종 검증
netstat -tnlp | grep 7289
```

### 1.4 최종 검증 결과
외부망에서 `ssh -p 7289 knu-lch@218.150.16.158`을 통해 안정적으로 원격 셸 접속이 확인되었으며, `/var/log/auth.log` 확인 결과 22번 포트로 유입되던 무차별 대입 공격 로그가 완전히 사라짐을 검증하였다.

---

## Chapter 2. 사용자 계정 권한 격리 및 SSH 공개키 인증 시스템

### 2.1 배경 및 과제 정의
내부 연구원(`knu-lch`, `knu-khw`, `knu-oym`, `knu-hjg`)과 외부 협력사(`jsf-kyd`) 등 다수의 사용자가 단일 메인 서버에 접속함에 따라, **비밀번호 유출 위험을 없애고 계정별 접근 권한을 엄격히 격리**해야 하는 요구사항이 발생하였다. 
특히 초기 환경에서는 모든 사용자가 `sudo` 권한을 보유하고 있어, 실수로 타 연구원의 데이터나 시스템 핵심 설정 파일이 훼손될 위험이 높았다.

### 2.2 기술 옵션 비교 및 최종 선정 근거
* **비밀번호 인증 vs ED25519 타원곡선 비대칭키 인증**:
  비밀번호 방식은 키로깅 및 훔쳐보기(Shoulder Surfing)에 취약하다. 이에 차세대 타원곡선 암호화 알고리즘인 **ED25519 공개키 인증 방식**을 전면 채택하였다. RSA 2048 대비 키 길이가 짧으면서도 연산 속도가 빠르고 해독이 불가능한 수준의 보안성을 제공한다.
* **최소 권한 원칙(Principle of Least Privilege) 적용**:
  시스템 설정 변경이 불필요한 일반 연구원 계정에서 `sudo` 권한을 100% 회수하고, 관리자(`knu`, `knu-lch`)만 관리 권한을 유지하도록 통제 구조를 일원화하였다.

### 2.3 세부 기술 구현 및 시행착오 해결
* **OpenSSH StrictModes 권한 거부(Permission Denied) 트러블슈팅**:
  공개키 등록 시 `authorized_keys` 파일 및 상위 디렉토리의 파일 소유권과 권한 퍼미션이 비인가 사용자에게 열려 있는 경우 OpenSSH 데몬이 보안상 이유로 키 접속을 거부하는 현상이 발생하였다.
  이를 해결하기 위해 소유권 및 엄격한 퍼미션(`700`, `600`)을 표준화하여 적용하였다.

```bash
# 일반 계정의 Sudo 관리자 권한 회수
sudo deluser knu-oym sudo
sudo deluser knu-khw sudo

# 계정별 SSH 디렉토리 및 공개키 보안 권한 표준화
sudo mkdir -p /home/<username>/.ssh
sudo chmod 700 /home/<username>/.ssh
sudo chmod 600 /home/<username>/.ssh/authorized_keys
sudo chown -R <username>:<username> /home/<username>/.ssh

# Sudo 3회 오류 시 세션 즉시 강제 종료 보안 트리거 (~/.bashrc)
sudo() {
    /usr/bin/sudo -v
    if [ $? -ne 0 ]; then
        echo "Sudo 인증 실패: 세션을 강제 종료합니다."
        kill -9 $$
    fi
    /usr/bin/sudo "$@"
}
```

### 2.4 최종 검증 결과
공개키를 소지한 인원만 SSH 접속이 허용되었으며, 일반 연구원 계정에서 `sudo` 실행 시 즉시 거부 및 로그 기록(`auditctl`)이 정상 동작함을 확인하였다.

---

## Chapter 3. 시놀로지 NAS 물리 네트워크 감지 트러블슈팅 및 10G 직결 아키텍처

### 3.1 배경 및 과제 정의
대용량 연구 데이터 저장용 시놀로지 NAS(100TB)를 메인 서버에 연결하여 활용하고자 하였다. 
그러나 초기 전원 인가 시 네트워크상에서 시놀로지 장비가 전혀 탐지되지 않는 감지 장애가 발생하였으며, 이를 해결하고 대용량 파일 전송을 위한 전용 10G 네트워크망을 구축해야 했다.

### 3.2 기술 옵션 비교 및 최종 선정 근거
* **네트워크 물리 구조 선택 (스위칭 허브 경유 vs 10G 랜카드 직접 연결)**:
  일반 1Gbps 공유기/스위치를 경유할 경우 30GB~50GB 대용량 데이터 전송 시 대역폭 병목 현상이 발생한다. 따라서 **메인 서버의 10G 랜카드(`eno2`)와 시놀로지 10G 포트를 랜선으로 1:1 직결(Direct Attachment)**하여 최고 속도의 스토리지 채널을 확보하였다.
* **Dual-Homed 서버 라우팅 충돌 회피 근거**:
  서버가 2개의 랜카드(인터넷망 `eno1` + 직결망 `eno2`)를 갖는 구조에서, 직결망 `eno2`에 Default Gateway를 설정할 경우 메인 패킷 라우팅 경로가 파괴되어 서버 인터넷이 끊어진다. 따라서 **`eno2`에는 IP만 지정하고 Gateway 설정을 배제하는 정밀 네트워크 설계를 채택**하였다.

### 3.3 세부 기술 구현 및 시행착오 해결 스토리라인

```
[원인 추적 분석 흐름]
1차 시도 (노트북 직결) ──► 센서 망(vlp16) 오설정 위험 감지 ──► "라이다 망 독립 보존" 규칙 수립
                                      │
2차 시도 (tcpdump 분석) ──► MAC (90:09:d0...) DHCP Request 포착 ──► 랜선 물리 미연결 원인 규명
```

1. **라이다(VLP-16) 센서망 보호 조치**:
   노트북으로 시놀로지 IP를 찾던 중 로봇 전용 벨로다인 라이다 네트워크 프로필(`vlp16`)을 건드릴 위험을 포착하고, **"센서 고정 IP 망은 절대 변경하지 않는다"**는 장비 안전 가이드라인을 제정하였다.
2. **`tcpdump` 패킷 분석을 통한 물리 원인 규명**:
   노트북 터미널에서 `sudo tcpdump -i eno1 -c 20` 명령어로 트래픽을 캡처한 결과, 시놀로지 MAC 주소(`90:09:d0:98:e5:43`)가 계속 브로드캐스팅 패킷을 보내며 DHCP 대기 상태임을 확인하였다. 물리적 랜선 미연결 상태였음을 규명하고 랜선 정상 직결을 통해 네트워크를 정상화하였다.

```yaml
# 메인 서버 10G 직결 전용 Netplan 설정 (/etc/netplan/90-synology.yaml)
network:
  version: 2
  ethernets:
    eno2:
      addresses: [10.0.0.1/24]   # Gateway 및 DNS를 생략하여 인터넷 라우팅 충돌 원천 방지
```

```bash
# 시놀로지 패키지 센터 및 시간 동기화를 위한 인터넷 NAT 공유 설정
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -o eno1 -j MASQUERADE
sudo netfilter-persistent save
```

### 3.4 최종 검증 결과
메인 서버와 시놀로지 간 `10.0.0.1` ↔ `10.0.0.2` (0.2ms 응답속도) 10G 직결 핑 통신에 성공하였으며, 시놀로지 NAS 또한 서버 NAT를 경유하여 패키지 센터 다운로드 및 NTP 시간 동기화가 정상 작동함을 확인하였다.

---

## Chapter 4. 외부 협력사 계정 SFTP Chroot Jail 격리 및 리눅스 ACL 설계

### 4.1 배경 및 과제 정의
외부 협력사(`JSFLUX`) 김영동 팀장 계정(`jsf-kyd`)에게 40TB 스토리지 전송 권한을 부여해야 하는 상황이 발생하였다. 
그러나 외부 인원이 메인 서버의 CLI 터미널 셸에 접속할 수 있게 두면 타 연구원의 소스코드 및 내부 시스템 파일에 접근하거나 수정하는 심각한 보안 리스크가 존재하였다.

### 4.2 기술 옵션 비교 및 최종 선정 근거

| 구분 | 대안 A: 일반 SSH 셸 로그인 허용 | 대안 B: FTP / Samba 서비스 개설 | **최종 선택: OpenSSH internal-sftp Chroot Jail** |
| :--- | :--- | :--- | :--- |
| **접근 범위** | 서버 시스템 전체 셸 접근 가능 | 파일 전송만 가능 | 지정된 `/mnt/synologyDB/shared_external`로 감옥(Jail) 격리 |
| **보안성** | 🔴 극도로 위험 (소체 코드 수정 위험) | 🟡 추가 포트 개방 및 불완전 암호화 | 🟢 SSH 터미널 완전 차단 + 암호화 전송 |
| **관리 효율** | 🟢 설정 없음 | 🔴 별도 디몬 관리 및 포트 추가 | 🟢 기존 SSH(7289) 포트 재활용 |
| **선정 근거** | - | - | **서버 OS 터미널 셸 진입을 물리적으로 차단하면서, 지정된 40TB 스토리지 폴더 내부로만 외부 인원을 완전히 좁혀두기(Jail) 위해 최선의 대안으로 선정.** |

### 4.3 세부 기술 구현 및 시행착오 해결
* **OpenSSH ChrootDirectory 디렉토리 소유권 필수 규칙**:
  OpenSSH 보안 규격상 `ChrootDirectory`로 지정되는 루트 경로(`/mnt/synologyDB/shared_external`)는 **반드시 `root:root` 소유여야 하며 타 사용자의 쓰기 권한이 배제(`755`)**되어야 한다. 이 조건을 충족하지 않으면 SFTP 접속 시 즉시 접속이 끊어진다.
  따라서 루트는 root 소유로 유지하고, 실제 업로드 공간인 하위 `/data` 폴더를 생성하여 권한을 할당하였다.
* **Linux Default ACL(Access Control List) 자동 권한 상속 설계**:
  외부인(`jsf-kyd`)이 올린 파일에 대해 내부 연구원(`knu` 그룹)이 매번 chmod를 치지 않고도 자동으로 읽기/쓰기 권한을 갖도록 POSIX Default ACL 상속을 설정하였다.

```bash
# /etc/ssh/sshd_config 하단 설정
Match Group sftp_only
    ChrootDirectory /mnt/synologyDB/shared_external
    ForceCommand internal-sftp
    AllowTcpForwarding no
    X11Forwarding no

# Chroot 보안 소유권 및 업로드 전용 data 폴더 구축
sudo chown root:root /mnt/synologyDB/shared_external
sudo chmod 755 /mnt/synologyDB/shared_external
sudo mkdir -p /mnt/synologyDB/shared_external/data
sudo chown jsf-kyd:sftp_only /mnt/synologyDB/shared_external/data
sudo chmod 770 /mnt/synologyDB/shared_external/data

# 내부 연구원 그룹(knu) 자동 권한 상속 (ACL)
sudo setfacl -d -m g:knu:rwx /mnt/synologyDB/shared_external/data
sudo setfacl -m g:knu:rwx /mnt/synologyDB/shared_external/data
```

### 4.4 최종 검증 결과
노트북에서 `ssh -p 7289 jsf-kyd@218.150.16.158` 실행 시 즉시 접속이 거부(`This service allows sftp connections only.`)되어 터미널 진입이 완벽히 차단됨을 확인하였다. SFTP 클라이언트 접속 시 40TB 용량 인식 및 `/data` 디렉토리 내 업로드가 정상 검증되었다.

---

## Chapter 5. 엣지-서버 실시간 로봇 관제 데이터 파이프라인 (MediaMTX & PostGIS)

### 5.1 배경 및 과제 정의
현장에 투입된 이동 로봇(엣지 디바이스)에서 수집되는 카메라인 실시간 영상(RTSP)과 RTK GPS 고정밀 위치 좌표 데이터를 중앙 서버로 수집하고, 이를 유니티(Unity) 기반 3D 관제 GUI 개발자에게 실시간 REST API 및 스트림 형태로 제공해야 하는 과제가 부여되었다.

### 5.2 기술 옵션 비교 및 최종 선정 근거
* **영상 스트리밍: HLS vs MediaMTX (RTSP/WebRTC)**:
  HLS 방식은 5~10초 이상의 지연시간(Latency)이 발생하여 원격 관제에 부적합하다. 초저지연(< 500ms)을 제공하는 리눅스 미디어 서버인 **MediaMTX (Port 8554/8889)**를 채택하여 실시간 재생 환경을 구축하였다.
* **위치 DB: 일반 RDBMS vs PostgreSQL + PostGIS Extension**:
  로봇의 이동 궤적 계산 및 공간 쿼리(Spatial Query)를 처리하기 위해 **PostGIS 공간 확장 기능이 탑재된 PostgreSQL 데이터베이스**를 선용하였다.

### 5.3 세부 기술 구현

```
[로봇 엣지 디바이스]                        [중앙 서버 (218.150.16.158)]                      [유니티 관제 GUI]
  ├─ ffmpeg (카메라)   ─── RTSP (8554)  ───►  MediaMTX (Port 8554/8889) ─── WebRTC/RTSP ──►  실시간 영상 렌더링
  └─ mqtt_publisher   ─── MQTT (1883)  ───►  FastAPI (server_api.py)     ─── REST API ────►  /api/telemetry/latest
                                                   │
                                                   ▼
                                              PostgreSQL (PostGIS POINT)
```

```bash
# 엣지 노트북 카메라 H.264 초저지연 RTSP 송출 명령
ffmpeg -f v4l2 -i /dev/video0 -pix_fmt yuv420p -c:v libx264 -preset ultrafast -tune zerolatency -f rtsp rtsp://218.150.16.158:8554/robot_cam
```

### 5.4 최종 검증 결과
노트북 웹캠 영상이 브라우저(`http://218.150.16.158:8889/robot_cam`) 및 VLC 플레이어로 지연 없이 출력되었으며, MQTT로 송출된 좌표 데이터가 SQLite/PostgreSQL `robot_telemetry` 테이블에 누락 없이 적재되고 FastAPI REST API(`http://218.150.16.158:8000/api/telemetry/latest`)를 통해 JSON으로 반환됨을 확인하였다.

---

## Chapter 6. 100TB 스토리지 용량 분배 및 Tailscale P2P 보안 팀 공유 저장소

### 6.1 배경 및 과제 정의
시놀로지 NAS의 100TB 물리 용량을 메인 서버, 백업, 팀원 공용 공간으로 효율적으로 분배해야 했다. 
특히 팀원들이 다루는 30GB~50GB 대용량 CAD/SolidWorks 설계 파일 전송을 지원하면서, **터미널 커맨드 사용이 어려운 상사분들을 위해 윈도우 탐색기(`Z:` 드라이브) 형태의 100% GUI 환경**을 제공해야 했다. 
또한 시놀로지와 팀원 PC가 서로 다른 공유기(망 분리) 환경에 위치해 있다는 기술적 제약이 존재하였다.

### 6.2 기술 옵션 비교 및 최종 선정 근거

| 구분 | 대안 A: 공인 IP 포트 포워딩 (SMB 445 포트) | 대안 B: 전통적 IPsec/OpenVPN 구축 | **최종 선택: Tailscale P2P Mesh VPN** |
| :--- | :--- | :--- | :--- |
| **보안성** | 🔴 극도로 위험 (랜섬웨어 SMB 무차별 타겟) | 🟢 우수함 | 🟢 최고 수준 (WireGuard 암호화 통신) |
| **사용성** | 🟡 윈도우 설정 필요 | 🔴 접속 시마다 VPN 클라이언트 연결 동작 필요 | 🟢 1회 로그인 후 윈도우 탐색기 `Z:` 드라이브 영구 고정 |
| **전송 속도** | 🟢 라우터 속도 | 🔴 VPN 서버 집중으로 인한 속도 병목 | 🟢 **P2P 1:1 직접 연결로 랜선 속도(1Gbps+) 100% 유지** |
| **선정 근거** | - | - | **공인 IP 포트 개방 없이 해커의 접근을 원천 차단하고, 서로 다른 공유기 환경에서도 P2P 직접 통신으로 30~50GB 대용량 전송 속도를 100% 보장하므로 최종 선택.** |

### 6.3 100TB 스토리지 할당 아키텍처

```
┌────────────────────────────────────────────────────────────────────────┐
│                        시놀로지 NAS (100TB)                            │
├───────────────────┬───────────────────┬────────────────────────────────┤
│   60TB (메인 서버) │    20TB (백업용)  │      20TB (팀원 공용 저장소)   │
├───────────────────┼───────────────────┼────────────────────────────────┤
│ • 메인 서버 마운트 │ • 서버 DB 및 데이터│ • 개인 방 (User Home 서비스)   │
│   (/mnt/synologyDB)│   자동 백업       │ • 팀원 공용 폴더 (Common)      │
│ • iSCSI / NFS     │ • Hyper Backup /  │ • 팀원 권한 (knu-lch, oym,    │
│ • external SFTP   │   rsync / Snapshot│   knu-oym, knu-khw 등)         │
│   [상태: 완료 ✅]  │   [상태: 완료 ✅]  │   [상태: 진행 중 ⏳]           │
└───────────────────┴───────────────────┴────────────────────────────────┘
```

### 6.4 대용량 CAD 파일 최적화 및 윈도우 마운트 설정
1. **설계 파일 동시 수정 덮어쓰기 방지 (Oplocks)**:
   시놀로지 DSM 공유 폴더 `00_Asia_hub` 설정에서 `Opportunistic Locking(기회주의적 잠금)` 및 `SMB3` 프로토콜을 활성화하여 30~50GB SolidWorks/AutoCAD 파일 동시 수정 시 파일 파손을 방지함.
2. **윈도우 탐색기 `Z:` 및 `Y:` 드라이브 고정**:
   `\\100.118.194.54\00_Asia_hub` (Z: 공용 5TB) 및 `\\100.118.194.54\home` (Y: 개인 500GB) 경로를 윈도우 탐색기 [네트워크 드라이브 연결]에 등록하여 내 컴퓨터 폴더와 동일한 사용 환경 구축.

---

### 📌 실행 로드맵 및 가이드 문서 안내
* **상세 실행 로드맵**: [roadmap.md](roadmap.md)
* **전체 구축 히스토리 및 기술 기록**: [history_and_setup_guide.md](history_and_setup_guide.md)
* **팀원용 1분 네트워크 드라이브 사용 설명서**: [team_user_guide.md](team_user_guide.md)

