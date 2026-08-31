# Antigravity User Preferences & Global System Rules

## 🚨 [MANDATORY RULE] 사용자 직접 검증 우선 원칙 (Direct Self-Verification First)

1. **사용자 직접 검증 방법 최우선 제공 (1-Click Verification)**:
   - 모든 센서, 네트워크, RTSP, DB, REST API, 백엔드 및 모듈 구축/변경 작업 시, 단순 백엔드 점검에 그치지 않고 **사용자가 자신의 화면에서 클릭 1번(로컬 웹뷰어 주소 `http://localhost:...`) 또는 명령어 1줄(`ffplay rtsp://...`)로 즉시 눈으로 직접 확인할 수 있는 수단**을 가장 먼저 제공한다.

2. **사전 검증 후 외부 공유 (Verify Before Sharing)**:
   - 사용자가 직접 눈으로 수신/동작을 확인하기 전에는 외부 협력사(JSFLUX 등) 회신이나 완료 선언을 먼저 하지 않는다.

3. **정확한 센서/카메라 장치 식별**:
   - 카메라나 센서 연동 시 내장 웹캠과 외장 USB 카메라(OAK-D, V4L2 USB 등)의 디바이스 노드(`/dev/video*`, `depthai`)를 사전에 정확히 구분하여 전용 스트리머 노드로 구동한다.
