### 📋 완료된 세부 작업 내역
  1. 서버 측 미디어 서버 & 백엔드 API 구동 완료:
      • MediaMTX (RTSP 서버, Port 8554): 백그라운드 구동 완료 ( rtsp://218.150.16.158:8554/robot_cam ).
      • FastAPI 백엔드 ( server_api.py , Port 8000): 백그라운드 구동 완료. MQTT 수신 후 SQLite DB( telemetry.db )
      자동 적재 및 REST API 제공.
  2. 로컬 엣지(노트북) 송수신 종합 검증 완료:
      • 텔레메트리 전송 테스트:  python3 mqtt_publisher.py  송출 → 서버 DB 적재 및  http://218.150.16.
      158:8000/api/telemetry/latest  정상 리턴 확인.
      • 노트북 웹캠 RTSP 스트리밍:  /dev/video0  내장 웹캠 영상을  ffmpeg 로 H.264 인코딩하여  rtsp://218.150.16.
      158:8554/robot_cam 으로 성공적으로 퍼블리싱 확인.
  ──────
  ### 📄 관련 문서 및 가이드

  • 🚀 통합 결과 및 유니티 전달용 안내서: project_status.md
  • 🛠️ 서버 A100 GPU 장착 가이드: a100_installation_guide.md
  • 📋 개발 현황판: development_status.txt
  ──────
  ### 🎮 유니티(Unity) 관제 GUI 개발자 전달 정보
  • 실시간 RTSP 영상 재생:  rtsp://218.150.16.158:8554/robot_cam 
  • 실시간 위치(최신 1개):  http://218.150.16.158:8000/api/telemetry/latest 
  • 이동 궤적(히스토리):  http://218.150.16.158:8000/api/telemetry/history 
  필드 테스트 및 유니티 연동 준비가 완벽하게 갖추어졌습니다!

────────────────────────────────────────────────────────────
> 오 그럼 이 노트북의 웹캠으로 테스트 어떻게 해볼수 있어?

▸ Thought for 4s, 623 tokens
  Guiding Webcam Testing
  노트북 웹캠과 위치 데이터를 실시간으로 서버에 쏘고 확인하는 3단계 실습 방법입니다!
  ──────
  ### 1️⃣ [터미널 1] 위치(텔레메트리) 데이터 전송

  노트북 터미널을 열고 아래 명령어를 실행합니다.

    python3 /home/knu/workspaces/edge_server_protocol/mqtt_publisher.py
    
  • 결과: 화면에 1초마다  [PUBLISHED] {'status': 'RTK_FIXED', 'latitude': 35.87..., 'longitude': 128.60...}  패킷이
  연속으로 전송됩니다.
  ──────
  ### 2️⃣ [터미널 2] 노트북 웹캠 영상 RTSP 송출

  새 터미널 창(또는 새 탭)을 열고 아래 명령어를 실행합니다.

    ffmpeg -f v4l2 -i /dev/video0 -pix_fmt yuv420p -c:v libx264 -preset ultrafast -tune zerolatency -f rtsp
  rtsp://218.150.16.158:8554/robot_cam

  • 결과:  Output #0, rtsp to 'rtsp://218.150.16.158:8554/robot_cam'  문구와 함께  frame= ...  숫자가 올라가며 웹캠
  화면이 서버로 실시간 송출됩니다.
  ──────
  ### 3 [확인] 웹 브라우저 / VLC 플레이어로 영상 및 위치 수신 확인

  #### 📺 웹캠 화면 확인 방법 (2가지 중 선택)

  1. 웹 브라우저에서 바로 확인 (가장 쉬움):
      • 크롬/웨일 등 브라우저 주소창에 아래 주소를 입력합니다:
      • 🔗  http://218.150.16.158:8889/robot_cam 
      • 별도 프로그램 없이 브라우저 화면에 노트북 웹캠 모습이 실시간 영상으로 출력됩니다!
  2. VLC 미디어 플레이어 사용:
      • VLC 실행 ->  미디어(F)  ->  네트워크 스트림 열기  ->  rtsp://218.150.16.158:8554/robot_cam  주소 입력 후 재생.


  #### 📍 위치 데이터 API 확인 방법

  • 브라우저 주소창에 아래 링크를 접속하면 실시간 좌표 JSON을 확인하실 수 있습니다:
  • 🔗  http://218.150.16.158:8000/api/telemetry/latest 

  직접 터미널에서 위 1번과 2번 명령어를 실행해 보세요!

