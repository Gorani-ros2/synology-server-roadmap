# 📜 메인 서버 & 시놀로지 NAS 구축 및 시행착오 전체 히스토리

**문서 위치**: `file:///home/knu/workspaces/server/history_and_setup_guide.md`  
**최종 작성일**: 2026-07-28  
**작성 목적**: 서버 인프라 구축, 계정 권한 관리, 네트워크 및 시놀로지 연동, 엣지 프로토콜 개발 과정에서 발생한 **시행착오와 학습한 핵심 개념들을 시간순으로 상세히 기록**하여 추후 복구 및 지식 자산화.

---

## 📑 목차 (Table of Contents)

1. [Chapter 1: 서버 포트 포워딩 & 기본 네트워크 구조 수립](#chapter-1-서버-포트-포워딩--기본-네트워크-구조-수립)
2. [Chapter 2: 사용자 계정 체계, Sudo 권한 회수 & SSH 공개키 인증](#chapter-2-사용자-계정-체계-sudo-권한-회수--ssh-공개키-인증)
3. [Chapter 3: 시놀로지 NAS 물리 연결 & 네트워크 감지 시행착오](#chapter-3-시놀로지-nas-물리-연결--네트워크-감지-시행착오)
4. [Chapter 4: 시놀로지 NAS 10G 직결 IP & 인터넷 NAT 공유 세팅](#chapter-4-시놀로지-nas-10g-직결-ip--인터넷-nat-공유-세팅)
5. [Chapter 5: 100TB 용량 분배 & 외부 계정 SFTP Chroot Jail 격리](#chapter-5-100tb-용량-분배--외부-계정-sftp-chroot-jail-격리)
6. [Chapter 6: 엣지-서버 관제 프로토콜 구축 (MediaMTX, FastAPI, PostGIS, MQTT)](#chapter-6-엣지-서버-관제-프로토콜-구축-mediamtx-fastapi-postgis-mqtt)
7. [Chapter 7: 팀원 공용 20TB 저장소 & Tailscale 보안 네트워크 로드맵 확립](#chapter-7-팀원-공용-20tb-저장소--tailscale-보안-네트워크-로드맵-확립)

---

## Chapter 1: 서버 포트 포워딩 & 기본 네트워크 구조 수립

### 1.1 서버 고정 IP 및 공유기 포트 포워딩
* **배경**: 외부망(LTE/스마트폰 핫스팟/외부 연구실)에서 메인 서버로 접속하고, 로봇 데이터를 수신하기 위해 공유기 NAT 포트 포워딩 설정.
* **내부 고정 IP**: `192.168.1.11` (공유기 DHCP 매핑으로 고정)
* **포트 포워딩 바인딩 맵**:
  * `9000`: MinIO Object Storage API
  * `9001`: MinIO Web Console
  * `5432`: PostgreSQL 데이터베이스
  * `7289`: SSH 원격 보안 접속 포트 (기본 22번 포트는 무차별 대입 공격에 노출되므로 보안 포트 7289로 변경)

### 1.2 배운 점: Ubuntu 24.04 LTS와 20.04의 SSH 데몬(sshd) 재시작 차이
* **Ubuntu 20.04 이전**: `sshd`가 프로세스로 상시 구동되어 있어 `sudo systemctl restart ssh`만 치면 적용됨.
* **Ubuntu 24.04 LTS 이후**: 시스템 리소스 효율화를 위해 **Socket-Based Activation** 방식으로 변경됨. 호출될 때만 `sshd`가 일어남.
* **수정 명령어 패턴**:
  ```bash
  sudo nano /etc/ssh/sshd_config   # Port 7289 지정
  sudo systemctl daemon-reload     # 소켓 유닛 재설정 갱신
  sudo systemctl restart ssh
  netstat -tnlp | grep 7289        # 7289 포트 대기 확인
  ```

---

## Chapter 2: 사용자 계정 체계, Sudo 권한 회수 & SSH 공개키 인증

### 2.1 사용자 계정 구조
* **내부 연구원 계정**: `knu-lch` (이충한), `knu-khw` (김해원), `knu-oym` / `oym` (오용민), `knu-hjg` (홍종근)
* **외부 협력사 계정**: `jsf-kyd` (JSFLUX 김영동 팀장)
* **최고 관리자 계정**: `knu`, `knu-lch`

### 2.2 SSH 공개키(Public Key) 로그인 인증 원리 및 시행착오
비밀번호 로그인 대신 공개키 인증 방식을 도입하여 보안을 대폭 강화함.

* **핵심 학습 원리**:
  1. 클라이언트(노트북/PC)에서 생성한 공개키(`id_ed25519.pub` 또는 `id_rsa.pub`)의 `ssh-ed25519 AAAAC3...` 문자열을 서버의 `~/.ssh/authorized_keys` 파일에 저장.
  2. **주의점**: 공개키 텍스트는 중간에 줄바꿈(Enter) 없이 **반드시 한 줄로 길게** 들어가야함.

* **보안 권한(Permission Denied) 트러블슈팅 시행착오**:
  OpenSSH 데몬은 퍼미션이 너무 넓게 열려 있으면 보안상 키 접속을 허용하지 않고 튕겨냄 (`Permission denied (publickey)`).
  ```bash
  # 계정별 올바른 보안 권한 세팅 필수 명령어
  sudo mkdir -p /home/<username>/.ssh
  sudo chmod 700 /home/<username>/.ssh
  sudo chmod 600 /home/<username>/.ssh/authorized_keys
  sudo chown -R <username>:<username> /home/<username>/.ssh
  ```

### 2.3 Sudo 권한 회수 및 감시(Audit) 강화
* **문제점**: 초기 모든 사용자 계정에 `sudo` 권한이 들어가 있어 일반 사용자가 다른 연구원의 파일이나 시스템 설정을 실수로 지울 위험 존재.
* **조치 내용**:
  1. 관리자(`knu`, `knu-lch`)를 제외한 나머지 사용자 계정에서 `sudo` 권한 전부 회수:
     ```bash
     sudo deluser knu-oym sudo
     sudo deluser knu-khw sudo
     ```
  2. `~/.bashrc`에 `sudo` 3회 오입력 시 세션 자동 강제 종료 함수 탑재 (`kill -9 $$`).
  3. auditd를 통한 파일 감시 레지스트리 구축 (`sudo auditctl -w /home -p rwxa -k home_watch`).

---

## Chapter 3: 시놀로지 NAS 물리 연결 & 네트워크 감지 시행착오

### 3.1 문제 발생 (시놀로지 무반응 현상)
시놀로지 NAS 전원을 켜고 랜선을 꽂았으나 웹 브라우저(`find.synology.com`)나 서버 및 노트북에서 시놀로지 IP가 전혀 감지되지 않음.

### 3.2 원인 추적 및 시행착오 과정
1. **1차 시도 (노트북 직접 연결)**: 노트북 랜 포트에 시놀로지를 직접 꽂았으나 IP 할당 실패.
2. **주의사항 발견**: 노트북 랜카드 IP를 수정하는 과정에서 벨로다인 라이다(Velodyne VLP-16) 프로필(`vlp16`)을 잘못 건드릴 뻔함. ➡️ **"vlp16 설정은 절대 수정 금지"** 규칙 확립.
3. **2차 시도 (패킷 분석 도구 `tcpdump` 사용)**:
   노트북 터미널에서 패킷 캡처 실행:
   ```bash
   sudo tcpdump -i eno1 -c 20
   ```
   * **결과**: 시놀로지 NAS의 MAC 주소(`90:09:d0:98:e5:43`)가 계속해서 `DHCP Request` 패킷을 전송하는 것을 포착!
4. **최종 밝혀진 원인**:
   * 시놀로지가 고정 IP가 아닌 DHCP(IP 자동 요청) 상태였으며, 물리적인 랜선 연결이 공유기 포트와 제대로 닿지 않아 IP를 할당받지 못했던 것이 원인이었음.
   * KT 홈허브 공유기에 랜선을 제대로 연결하자 `172.30.1.x` 대역 IP를 정상 할당받음.

---

## Chapter 4: 시놀로지 NAS 10G 직결 IP & 인터넷 NAT 공유 세팅

### 4.1 메인 서버 10G 직결 고정 IP 구성
* **네트워크 물리 구조**:
  * 메인 서버(`dt-core`)의 10G 랜카드 포트(`eno2`) ↔ 시놀로지 NAS의 10G 랜 포트 직결.
* **IP 주소 할당**:
  * **시놀로지 NAS**: 고정 IP `10.0.0.2` (Gateway: `10.0.0.1`, Subnet: `255.255.255.0`)
  * **메인 서버 (`eno2`)**: 영구 고정 IP `10.0.0.1/24`

* **Netplan 영구 저장 설정 (`/etc/netplan/90-synology.yaml`)**:
  ```yaml
  network:
    version: 2
    ethernets:
      eno2:
        addresses: [10.0.0.1/24]
  ```
  * **주의사항**: `eno2`는 서버 전용 직결 망이므로 게이트웨이(`gateway4`)나 DNS를 추가하면 메인 인터넷망(`eno1`)과 충돌하여 서버 인터넷이 끊어짐. IP만 등록해야 함.
  * 퍼미션 제한 적용: `sudo chmod 600 /etc/netplan/90-synology.yaml` 후 `sudo netplan apply`.

### 4.2 시놀로지 인터넷 공유 (IP Forwarding & NAT)
시놀로지가 패키지 센터 다운로드 및 NTP 시간 동기화를 하려면 인터넷 접속이 필요함. 메인 서버의 인터넷 포트(`eno1`)를 공유하도록 NAT 설정 적용.

1. **포패딩 및 NAT 트래픽 변환 활성화**:
   ```bash
   sudo sysctl -w net.ipv4.ip_forward=1
   sudo iptables -t nat -A POSTROUTING -o eno1 -j MASQUERADE
   ```
2. **부팅 시 영구 저장**:
   `/etc/sysctl.conf`에 `net.ipv4.ip_forward=1` 추가 후 `iptables-persistent` 패키지로 방화벽 규칙 저장 (`sudo netfilter-persistent save`).

---

## Chapter 5: 100TB 용량 분배 & 외부 계정 SFTP Chroot Jail 격리

### 5.1 시놀로지 100TB 용량 설계
* **60TB (메인 서버 전용)**: `/mnt/synologyDB` iSCSI / NFS 마운트 (DB 저장 및 외부 수신용)
* **20TB (자동 백업 전용)**: Hyper Backup / rsync 스케줄링 백업 볼륨
* **20TB (팀원 공용 저장소)**: User Home 서비스 + `Common` 공용 공유 폴더

### 5.2 외부 협력사 계정(`jsf-kyd`) SFTP Chroot Jail 격리
* **배경 및 필요성**: 외부 인원이 메인 서버 CLI 셸에 접속하여 다른 시스템 파일을 열어보거나 코드를 mutating하는 보안 위협 차단.

* **구축 상세 절차**:
  1. `jsf-kyd` 계정에서 `sudo` 및 `sshusers` 권한 제거 ➡️ `sftp_only` 전용 그룹 할당.
  2. SSH 설정(`/etc/ssh/sshd_config`) 하단에 격리 구문 추가:
     ```etc
     Match Group sftp_only
         ChrootDirectory /mnt/synologyDB/shared_external
         ForceCommand internal-sftp
         AllowTcpForwarding no
         X11Forwarding no
     ```
  3. **Linux 디렉토리 권한 법칙 (Chroot 조건)**:
     Chroot 루트인 `/mnt/synologyDB/shared_external` 디렉토리는 **반드시 `root:root` 소유**여야 하며 **다른 일반 사용자의 쓰기 권한이 없어야 함(`755`)**.
  4. 업로드 디렉토리 `/mnt/synologyDB/shared_external/data` 생성 및 `jsf-kyd:sftp_only` (`770`) 소유권 부여.
  5. **Linux ACL(Access Control List) 상속 설정**:
     외부 인원 `jsf-kyd`가 올린 파일을 내부 연구원(`knu-lch` 등 `knu` 그룹)이 자동으로 읽고 쓰기 권한을 가지도록 기본 ACL 적용:
     ```bash
     sudo setfacl -d -m g:knu:rwx /mnt/synologyDB/shared_external/data
     sudo setfacl -m g:knu:rwx /mnt/synologyDB/shared_external/data
     ```
  6. **검증 완료**: `ssh -p 7289 jsf-kyd@...` 접속 시 셸 진입 거부(`Permission denied`), SFTP 접속 시 40TB 용량 표시 및 `/data` 폴더 내부로 완전 격리(Jail) 확인.

---

## Chapter 6: 엣지-서버 관제 프로토콜 구축 (MediaMTX, FastAPI, PostGIS, MQTT)

### 6.1 미디어 서버 및 백엔드 파이프라인
```
[노트북 / 로봇 엣지]                      [메인 서버 (218.150.16.158)]                     [유니티 관제 GUI]
  ├─ ffmpeg (웹캠)  ─── RTSP 8554 ───────►  MediaMTX (Port 8554 / 8889)  ────── WebRTC/RTSP ─►  실시간 영상 재생
  └─ mqtt_publisher ─── MQTT 1883 ───────►  FastAPI (server_api.py)      ────── REST API ───►  /api/telemetry/latest
                                                 │
                                                 ▼
                                            PostgreSQL DB (PostGIS POINT)
```

### 6.2 주요 구성 요소
1. **MediaMTX (RTSP 서버)**: Port `8554` (RTSP), Port `8889` (WebRTC) 백그라운드 구동.
2. **FastAPI 백엔드 (`server_api.py`)**:
   * MQTT `robot/sensor/rtk` 토픽 구독 ➡️ PostgreSQL DB (`jsflux_field_db`, `robot_telemetry` 테이블 & PostGIS `robot_latest_status` View) 적재.
   * REST API 제공: `/api/telemetry/latest`, `/api/telemetry/history`.
3. **노트북 송출 테스트 명령어**:
   * 위치 데이터: `python3 /home/knu/workspaces/edge_server_protocol/mqtt_publisher.py`
   * 영상 스트리밍:
     ```bash
     ffmpeg -f v4l2 -i /dev/video0 -pix_fmt yuv420p -c:v libx264 -preset ultrafast -tune zerolatency -f rtsp rtsp://218.150.16.158:8554/robot_cam
     ```

---

## Chapter 7: 팀원 공용 20TB 저장소 & Tailscale 보안 네트워크 로드맵 확립

### 7.1 문제 요소 및 과제 해결
* **요구사항**: 30~50GB 대용량 CAD/설계 파일 전송 & 터미널 사용이 어려운 상사분들을 위해 윈도우 탐색기 `Z:` 드라이브 사용 환경 제공.
* **망 분리 문제**: 시놀로지 연결 공유기와 팀원/상사 PC 연결 공유기가 달라 직접 접근 불가.
* **해결책**: **Tailscale P2P Mesh VPN** 도입 ➡️ 공유기/네트워크가 달라도 P2P 1:1 직접 암호화 연결로 속도 저하(1Gbps+) 없이 보안 접속 구현.
* **로드맵 문서화**: [roadmap.md](file:///home/knu/workspaces/server/roadmap.md) 생성 및 GitHub (`Gorani-ros2/synology-server-roadmap`) 연동 완료.
