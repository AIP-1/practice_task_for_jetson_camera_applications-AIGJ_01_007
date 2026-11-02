# Jetson 카메라 응용 실습 8~10단계 (AIGJ_01_007)

Jetson Orin Nano와 USB 웹캠으로 진행한 EmbeddingGemma 2 실습의 카메라 응용 단계입니다. 웹캠 물체 추출과 검색(8단계), 사람 동작·행동 인식(9단계), 두 카메라 근접 알람(10단계)을 담았습니다. 1~7단계(텍스트·이미지·음성 임베딩)는 [embedding_gemma2-AIGJ_01_007](https://github.com/AIP-1/embedding_gemma2-AIGJ_01_007) 저장소에 있습니다. 실습 안내문은 [docs/guide_README.md](docs/guide_README.md)에 있습니다.

## 함께 진행

8~10단계는 AIGJ_01_004와 보드 B에서 함께 진행했습니다. 코드와 측정 결과는 공동 작업물이며, 같은 내용이 [practice_task_for_jetson_camera_applications-AIGJ_01_004](https://github.com/AIP-1/practice_task_for_jetson_camera_applications-AIGJ_01_004)에도 있습니다. 제가 따로 남긴 단계별 판정과 학습 노트는 [embedding_gemma2-AIGJ_01_007의 docs/reports](https://github.com/AIP-1/embedding_gemma2-AIGJ_01_007/tree/main/docs/reports)에 있습니다.

## 환경 (보드 B, team5-b)

- Jetson Orin Nano 8GB, L4T R39 (JetPack 7 계열), Python 3.12, MAXN_SUPER
- torch 2.14.1+cu132, ultralytics 8.4.184, 모델 google/embeddinggemma-2
- USB 웹캠 Logitech C920 2대 (10단계는 두 대 동시 사용)

## 진행 단계

| 단계 | 내용 | 상태 |
| --- | --- | --- |
| 8 | 웹캠 물체 추출과 웹 검색 (YOLO, 임베딩 재라벨, YOLOE) | 완료 |
| 9 | 사람 동작 인식(만세·손 흔들기 등)과 행동 패턴, GRU 학습 | 완료 |
| 10 | 두 카메라 보정, ankle1·stereo·fused 거리 비교, 타겟 자동 찾기, 이동 타겟 충돌 알람 | 완료 |

## 폴더 구성

```
app.py              8단계 물체 추출 + 웹 검색 서버 (--mode yolo|relabel|yoloe)
pose_app.py         9단계 동작·행동 인식 (--raw, --record, --behavior, --model)
pose_model.py       9단계 관절 특징과 GRU 모델
train_pose.py       9단계 학습 스크립트
data/action/        9단계 동작 학습 데이터 (관절 좌표 csv)
data/behavior/      9단계 행동 학습 데이터 (관절 좌표 csv)
prox/               10단계 두 카메라 근접 알람
  calib_intrinsics.py   렌즈 보정 (체커보드)
  calib_extrinsics.py   카메라 위치 보정 (바닥 기준점 클릭)
  stereo.py             두 카메라 삼각측량
  proximity_app.py      근접·이탈·충돌 알람 앱
  calib/                기준점 좌표, 클릭 좌표, 측정 기록
  check/                카메라 위치 변화 확인 이미지
results/            8, 9, 10단계 측정 결과
reports/            PDF 보고서 (9단계, 10단계, 1~10단계 전체)
docs/               실습 안내문
```

## 주요 결과

**8단계 물체 추출**

| 모드 | fps | 특징 |
| --- | --- | --- |
| yolo | 15 | COCO에 없는 물체를 비슷한 클래스로 오인식 (이어폰→remote, 에어컨→microwave) |
| relabel | 15~17 | 목록에 넣은 물체는 바로잡지만 목록 밖 물체는 엉뚱한 이름으로 바꿈 |
| yoloe | 13 | 선풍기·벽시계처럼 COCO에 없는 물체도 이름으로 찾음, 작은 물체는 헷갈림 |

**9단계 동작·행동 인식**
- 프레임 단위 판정으로는 손 흔들기를 구분할 수 없었고, 시간 패턴(1.5초 안 방향 전환)으로 해결.
- 동작 GRU(window 30) 검증 정확도 0.989. 실시간으로 4개 동작 모두 인식.
- 행동(지나감/힐끗 봄/관심 있음)은 한 번 지나가는 시간이 짧아 window 15로 학습.

**10단계 두 카메라 근접 알람**
- 카메라 보정: 재투영 오차 3.78 / 3.22 px, 확인점 오차 3 cm.

| 실제 거리 (서 있음) | ankle1 | stereo | fused |
| --- | --- | --- | --- |
| 0.5 m | 0.56 | 0.61 | 0.63 |
| 1.0 m | 1.18 | 1.06 | 1.21 |
| 2.0 m | 2.05 | 2.14 | 2.11 |
| fps | 26.7 | 23.1 | 23 |

- 타겟 자동 찾기 후 알람 시나리오 5개 모두 3/3 성공.
- 이동 타겟: 충돌 위험 알람이 근접 알람보다 천천히 걸을 때 약 3초, 보통 속도일 때 약 1초 먼저 울림. `--ttc 3.0`이면 각각 1초 더 일찍 울림.

## 안내문과 다르게 한 점

- ultralytics 8.4.184에서 `half=True`가 프레임마다 경고를 띄워서 `app.py`, `pose_app.py`, `proximity_app.py`에서 제거.
- 9단계 `train_pose.py`: 표준편차가 0에 가까운 특징 때문에 실시간 판정이 틀어져서 `std.clamp(min=0.03)`으로 수정.
- 10단계 카메라는 `--cams 0 2`(/dev/video0, /dev/video2), 렌즈 보정은 태블릿 체커보드(칸 22 mm, `--square 0.022`).

학습된 모델(.pt), 카메라 보정 결과(.npz), 녹화 영상은 저장소에 넣지 않았습니다.
