![banner](assets/banner.svg)

# FashionData — 패션 이미지 성별·스타일 분류 및 선호 스타일 추천

![Python](https://img.shields.io/badge/Python-3776AB?style=flat-square&logo=python&logoColor=white)
![PyTorch](https://img.shields.io/badge/PyTorch-EE4C2C?style=flat-square&logo=pytorch&logoColor=white)
![torchvision](https://img.shields.io/badge/torchvision-EE4C2C?style=flat-square&logo=pytorch&logoColor=white)
![scikit-learn](https://img.shields.io/badge/scikit--learn-F7931E?style=flat-square&logo=scikitlearn&logoColor=white)
![pandas](https://img.shields.io/badge/pandas-150458?style=flat-square&logo=pandas&logoColor=white)
![NumPy](https://img.shields.io/badge/NumPy-013243?style=flat-square&logo=numpy&logoColor=white)
![Google Colab](https://img.shields.io/badge/Google%20Colab-F9AB00?style=flat-square&logo=googlecolab&logoColor=white)

**2024 데이터 크리에이터 캠프(DCC) — 팀 어셈블** 프로젝트입니다.
패션 이미지 데이터셋을 대상으로 (1) 파일명 규칙 기반 데이터 통계, (2) ResNet-18 커스텀 학습을 통한 성별·스타일 분류, (3) 학습된 모델을 특징 추출기로 재활용한 협업 필터링 기반 스타일 선호 예측(추천)을 수행했습니다. 전체 실험은 Google Colab GPU 환경에서 진행했습니다.

<img src="스크린샷 2024-11-02 143617.png" alt="패션 스타일 이미지 예시" width="500"/>

## 목차

- [데이터셋](#데이터셋)
- [미션 1 — 데이터 통계와 성별·스타일 분류](#미션-1--데이터-통계와-성별스타일-분류)
- [미션 2 — 유효 라벨 검증과 스타일 선호 정보표](#미션-2--유효-라벨-검증과-스타일-선호-정보표)
- [미션 3 — 협업 필터링 기반 선호 스타일 예측](#미션-3--협업-필터링-기반-선호-스타일-예측)
- [결과 요약](#결과-요약)
- [실행 방법 (Google Colab)](#실행-방법-google-colab)
- [저장소 구성](#저장소-구성)

## 데이터셋

이미지 파일명은 아래 규칙을 따르며, 파일명만으로 성별·스타일 메타데이터를 얻을 수 있습니다.

```text
{W/T}_{이미지ID}_{시대}_{스타일}_{성별}.jpg
```

- 이미지: `training_image` / `validation_image` 폴더 (Training 총 4,070장)
- 라벨: 이미지별 JSON 파일 — 응답자 ID(`R_id`), 스타일 선호 설문(`Q5`: 1 비선호 / 2 선호) 포함
- 클래스: 성별 × 스타일 조합 총 31종 (남성 8종: bold, hiphop, hippie, ivy, metrosexual, mods, normcore, sportivecasual / 여성 23종: athleisure, bodyconscious, cityglam, classic, disco, ecology, feminine, genderless, grunge, hiphop, hippie, kitsch, lingerie, lounge, military, minimal, normcore, oriental, popart, powersuit, punk, space, sportivecasual)

<img src="스크린샷 2024-11-02 150238.png" alt="데이터 파일 형식" width="500"/>

## 미션 1 — 데이터 통계와 성별·스타일 분류

### [1-1] 파일명 규칙 기반 데이터 통계

파일명을 `_` 기준으로 파싱해 성별(W/T)과 스타일 토큰을 추출하고, `defaultdict`로 이미지 ID 수 기준의 **성별 × 스타일 통계표**를 집계했습니다 (pandas DataFrame으로 정렬 출력).

<img src="스크린샷 2024-11-02 142256.png" alt="성별·스타일 통계 결과" width="500"/>

### [1-2] ResNet-18 성별·스타일 분류

torchvision의 ResNet-18을 **사전학습 가중치 없이(`weights=None`) 처음부터 학습**해 31개 클래스(성별_스타일 조합)를 분류했습니다.

| 항목 | 설정 |
|---|---|
| 백본 | `torchvision.models.resnet18(weights=None)` + 커스텀 fc(512 → 31), Dropout 0.4 |
| 입력 데이터 | 세그멘테이션 전처리 이미지 (`processed_segmentation_cleaned`) |
| 증강 | RandomHorizontalFlip, RandomRotation(30°), ColorJitter, RandomResizedCrop(224) — 검증에는 미적용 |
| 손실 함수 | CrossEntropyLoss |
| 옵티마이저 | AdamP (`torch_optimizer`, lr 0.005, weight decay 0.02) |
| 스케줄러 | CosineAnnealingWarmRestarts (T_0=10) |
| 배치 크기 | 128, 총 400 에폭 (체크포인트 저장 후 이어서 학습) |

<img src="스크린샷 2024-11-02 142323.png" alt="분류 학습 결과" width="400"/>

## 미션 2 — 유효 라벨 검증과 스타일 선호 정보표

### [2-1] 유효 라벨링 데이터 확인

정규 표현식으로 이미지 파일명과 라벨 JSON 파일명을 각각 매칭해 **실제 이미지가 존재하는 유효 이미지 ID**만 추려내고, 해당 라벨의 설문 응답(Q5)과 응답자 ID를 수집해 통계표를 생성했습니다.

### [2-2] 100명의 스타일 선호 정보표

응답자별로 선호(2)·비선호(1) 응답을 집계하고, 총 응답 수 기준 **상위 100명의 응답자에 대한 스타일 선호 정보표**를 만들어 엑셀(`preferences0.xlsx`)로 저장했습니다. 이 표가 미션 3 추천의 입력이 됩니다.

<img src="스크린샷 2024-11-02 142407.png" alt="스타일 선호 정보표" width="400"/>

## 미션 3 — 협업 필터링 기반 선호 스타일 예측

### [3-1] 추천 방식 분석

아이템 기반 필터링(스타일 간 유사성 중심)과 사용자 기반 필터링(유사 취향 사용자 중심)의 정의·적용 방법·장단점을 비교 분석했습니다. 선호 데이터가 희소한 본 데이터셋 특성상, 이미 선호한 스타일 이미지와의 유사성으로 추천하는 접근을 채택했습니다.

### [3-2] 임베딩 유사도 기반 선호 예측

1. **특징 추출기**: 미션 1에서 학습한 ResNet-18에서 **fc 레이어를 제거**해 512차원 임베딩 추출기로 재활용
2. **유사도 계산**: 검증 이미지 임베딩과 사용자가 선호한 학습 이미지 임베딩 간 **코사인 유사도**와 **유클리드 거리(음수 변환)** 를 배치 단위로 계산
3. **점수 결합**: 두 유사도를 가중 결합

   ```text
   score = alpha * cosine_similarity + (1 - alpha) * (-euclidean_distance)
   ```

   `alpha`는 0.1~0.9 그리드 서치로 **F1 점수가 최대가 되는 값**을 탐색
4. **임계값 최적화**: 결합 점수에 대한 최적 threshold를 탐색해 선호/비선호를 최종 예측하고 accuracy·precision·recall·F1로 평가

## 결과 요약

| 과제 | 지표 | 결과 |
|---|---|---|
| 미션 1-2: 31클래스 분류 | 검증 정확도 (Best) | **64.14%** |
| 미션 3-2: 선호 예측 | Optimal Alpha / Threshold | 0.60 / 0.10 |
| 미션 3-2: 선호 예측 | Accuracy / F1 | **78.69% / 76.56%** (부록 재실행: 67.88% / 61.82%) |

## 실행 방법 (Google Colab)

1. `어셈블_최종코드_미션1~3_합본.ipynb`를 [Google Colab](https://colab.research.google.com/)에서 열기 (런타임 유형: **GPU**)
2. 데이터셋을 Google Drive에 업로드 후 드라이브 마운트

   ```python
   from google.colab import drive
   drive.mount('/content/drive')
   ```

3. 노트북 상단의 데이터 경로(`/content/drive/MyDrive/dataset/...`)를 본인 드라이브 경로에 맞게 수정
4. 추가 패키지 설치 셀 실행 후 (`torch_optimizer`, `ujson` 등) 미션 1 → 2 → 3 순서로 셀 실행

## 저장소 구성

| 파일 | 설명 |
|---|---|
| `어셈블_최종코드_미션1~3_합본.ipynb` | 미션 1~3 최종 코드 노트북 (메인) |
| `어셈블_최종_과제.ipynb` | 최종 과제 노트북 |
| `어셈블_최종코드_미션1~3_합본.md` | 노트북 마크다운 내보내기 (코드·실행 결과 열람용) |
| `24년 DCC PPT '어셈블'.pptx` | 발표 자료 |
| `스크린샷 *.png` | 데이터·결과 스크린샷 |
