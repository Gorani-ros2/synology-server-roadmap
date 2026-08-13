# 🏛️ Dell Precision 7920 Rack GPU 업그레이드 검토 및 기술 의사결정 보고서

* **문서 위치**: `file:///home/knu/workspaces/server/gpu_upgrade_decision_guide.md`
* **작성일**: 2026-07-31
* **타겟 서버**: Dell Precision 7920 Rack (`dt-core`)
* **목적**: 센서 융합(라이다+8K RGB), 3D 디지털 트윈 구축, 생육 센서 예측 모델용 최적 GPU 선정 및 호환성 검증

---

## 📑 목차 (Table of Contents)

1. [현재 서버 사양 및 연구 워크로드 정의](#1-현재-서버-사양-및-연구-워크로드-정의)
2. [GPU 후보군 비교 검토 (A100 vs L40S vs RTX 6000 Ada vs RTX Pro 5000 Blackwell)](#2-gpu-후보군-비교-검토)
3. [최종 선정: RTX Pro 5000 Blackwell D7 48GB (x 2장) 선정 이유](#3-최종-선정-rtx-pro-5000-blackwell-d7-48gb-x-2장-선정-이유)
4. [기존 서버(Dell 7920) 물리적 및 전력 호환성 검증](#4-기존-서버dell-7920-물리적-및-전력-호환성-검증)
5. [하드웨어 설치 및 전원 케이블 가이드](#5-하드웨어-설치-및-전원-케이블-가이드)
6. [소프트웨어 환경 구축 및 운용 팁 (PyTorch / CUDA / Docker)](#6-소프트웨어-환경-구축-및-운용-팁)

---

## 1. 현재 서버 사양 및 연구 워크로드 정의

### 1.1 서버 현재 사양 (`dt-core`)
* **서버 모델**: Dell Precision 7920 Rack (2U 랙마운트)
* **CPU**: Dual Intel(R) Xeon(R) Gold 6140 @ 2.30GHz (총 36코어 72스레드)
* **시스템 메모리(RAM)**: **256 GB DDR4**
* **저장장치**: 1 TB NVMe SSD
* **파워서플라이(PSU)**: **1300W Redundant PSU**
* **기존 그래픽카드**: NVIDIA RTX A4000 (16GB) x 2장

### 1.2 연구 워크로드 핵심 요구사항
1. **대용량 멀티센서 데이터 융합**:
   - Ouster OS0-128 (128채널 수억 개 포인트클라우드) + Insta360 X5 (8K 360도 파노라마 비디오) 프레임 단위 동기화 및 3D Calibration.
2. **3D 디지털 트윈 구축 (3D Reconstruction & Rendering)**:
   - 3D Gaussian Splatting, NeRF, OptiX 광선 추적(Ray Tracing) 실시간 뷰포트 렌더링.
3. **생육 환경 센서 기반 AI 예측 모델**:
   - 온·습도, CO2, 토양 수분, EC, pH 시계열 센서 데이터 기반 예측 모델 (Time-Series Transformer, XGBoost GPU, PyTorch).

---

## 2. GPU 후보군 비교 검토

| 비교 항목 | NVIDIA A100 40GB | NVIDIA L40S 48GB | RTX 6000 Ada 48GB | **NVIDIA RTX Pro 5000 Blackwell 48GB** |
| :--- | :--- | :--- | :--- | :--- |
| **아키텍처** | Ampere (2020) | Ada Lovelace (2023) | Ada Lovelace (2023) | **Blackwell (최신 차세대)** |
| **VRAM** | 40GB HBM2 | 48GB GDDR6 | 48GB GDDR6 | **48GB GDDR7 (D7)** |
| **VRAM 대역폭** | 1,555 GB/s | 864 GB/s | 960 GB/s | **~1,500 ~ 1,800 GB/s (HBM급)** |
| **FP32 연산력** | 19.5 TFLOPS | 91.6 TFLOPS | 91.1 TFLOPS | **약 140+ TFLOPS (A100 대비 ~7배)** |
| **RT Cores (3D)**| **0개 (없음)** | 142개 (3rd Gen) | 142개 (3rd Gen) | **4th Gen RT Cores (A100 대비 ~10배)** |
| **Tensor Cores**| 3rd Gen (FP8 미지원)| 4th Gen (FP8 지원) | 4th Gen (FP8 지원) | **5th Gen (FP4, FP6, FP8 지원)** |
| **비디오 인코더** | 없음 (0개) | 3x NVENC + NVDEC | 3x NVENC + NVDEC | **Dual 8th Gen NVENC + NVDEC** |
| **쿨링 방식** | Passive 방열판 | Passive 방열판 | Active Blower 팬 | **Active Blower / Passive 선택** |
| **소비 전력** | 250W | 350W | 300W | **300W ~ 350W** |

---

## 3. 최종 선정: RTX Pro 5000 Blackwell D7 48GB (x 2장) 선정 이유

1. **GDDR7 메모리 대역폭 폭발적 향상**:
   - A100의 유일한 무기였던 HBM2 대역폭 수준(~1.5 TB/s)을 차세대 **GDDR7(D7)**이 그대로 달성하여 대용량 포인트클라우드 및 8K 영상 디코딩 병목을 완전히 제거함.
2. **4세대 RT 코어 탑재 (디지털 트윈 필수)**:
   - A100은 RT 코어가 0개여서 3D Gaussian Splatting / NeRF 렌더링에 취약함. 블랙웰은 4세대 RT 코어를 통해 실시간 60fps 이상 렌더링 보장.
3. **5세대 텐서 코어 & FP4/FP6 양자화 지원**:
   - 시계열 생육 예측 AI 및 멀티모달 VLM 학습/추론 속도가 기존 세대 대비 2~3배 이상 향상됨.
4. **합리적인 가성비 (+200만 원으로 세대교체)**:
   - L40S(약 1,300만 원) 대비 200만 원 차이(약 1,500만 원)로 최신 차세대 아키텍처를 도입하여 향후 4~5년 이상 기술적 우위 유지 가능.

---

## 4. 기존 서버(Dell 7920) 물리적 및 전력 호환성 검증

### 4.1 파워서플라이(1300W) 전력 산정
- Dual Xeon Gold 6140 CPUs: 약 280W
- 메인보드, 256GB RAM, SSD, 쿨링팬: 약 120W
- **시스템 기본 전력**: 약 **400W**
- **RTX Pro 5000 Blackwell 2장**: 약 300W x 2 = **600W**
- **최대 피크 총 전력**: **약 1,000W** (1300W PSU 기준 사용률 약 76%로 **매우 안전한 범위**)

> 💡 **의사결정**: 신규 랙 서버 추가 구매(1,000만~2,000만 원 지출) 없이, 기존 서버의 **RTX A4000 2장을 교체(Swap)**하여 블랙웰 2장을 교체 장착하는 것이 예산 절감 및 성능 극대화에 최적임.

---

## 5. 하드웨어 설치 및 전원 케이블 가이드

### 5.1 슬롯 배치
* **Riser 1 슬롯**: RTX Pro 5000 Blackwell #1 (48GB) 장착
* **Riser 2 슬롯**: RTX Pro 5000 Blackwell #2 (48GB) 장착
* *(기존 RTX A4000 2장은 탈거하여 연구실 데스크톱 PC로 이전)*

### 5.2 전원 케이블 수량 및 규격
* 블랙웰 카드는 최신 **16핀 (12V-2x6 / 12VHPWR)** 커넥터를 사용함.
* Dell 7920 메인보드/라이저 구역에는 **8핀 GPU 전원 포트가 총 4개** 내장되어 있음.
* **필요 케이블**: **Dell Riser 전용 Dual 8-pin to 16-pin (12V-2x6) GPU 전원 케이블 총 2개** 사용.

---

## 6. 소프트웨어 환경 구축 및 운용 팁

### 6.1 드라이버 및 CUDA 설치
* **추천 드라이버**: NVIDIA Enterprise Driver **550 / 560 / 595+**
* **CUDA Toolkit**: **CUDA 12.x / 13.x**

### 6.2 도커(Docker) 기반 제로 스토크 환경 구축
서버 환경에서는 호스트 드라이버 충돌을 피하기 위해 **NVIDIA Container Toolkit**을 활용함:

```bash
# 엔비디아 공식 PyTorch 도커 컨테이너 구동 (GPU 2장 지정)
docker run --gpus all -it --rm -v /home/knu/workspaces:/workspace nvcr.io/nvidia/pytorch:24.03-py3
```

### 6.3 파이썬 코드 상에서의 GPU 지정 실행
```bash
# 새로 추가한 48GB 블랙웰 카드로 3D Gaussian Splatting 실행
CUDA_VISIBLE_DEVICES=0,1 python train_3dgs.py
```

---

*문서 생성 완료: 2026-07-31 (KNU AI Robotics Lab)*
