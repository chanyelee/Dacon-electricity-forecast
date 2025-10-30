# Dacon 전력사용량 예측 - 데이터 전처리

본 리포지토리는 'Dacon 전력사용량 예측 AI 경진대회'의 원본 데이터를 모델 학습에 적합하도록 가공하는 **데이터 전처리(`데이터_전처리.ipynb`)** 스크립트입니다.

이 스크립트의 주된 목적은 원본 데이터(`train.csv`, `building_info.csv` 등)를 입력받아, **총 9개의 최적화된 CSV 파일**(`train_group_1.csv`, `valid_group_2.csv` 등)을 생성하는 것입니다.

---

## 🚀 실행 환경

* 본 스크립트는 **Google Colab** 환경에서 GPU를 사용하여 실행하는 것을 전제로 작성되었습니다.
* `google.colab.drive`를 사용하여 Google Drive를 마운트합니다.

## 📋 필수 파일 및 경로

스크립트를 실행하기 전, 본인의 Google Drive에 다음과 같은 구조로 원본 데이터 파일이 준비되어 있어야 합니다.

* **지정 경로 (Path):**
    * `/content/drive/MyDrive/위아이티/`
* **필수 파일 (Files):**
    * `building_info.csv`
    * `train.csv`
    * `test.csv`
    * `sample_submission.csv`

*(만약 경로가 다르다면, 스크립트 내의 `path` 변수를 수정해야 합니다.)*

---

## 🛠️ 전처리 파이프라인 요약

본 스크립트는 다음의 순서로 데이터를 전처리합니다.

1.  **기본 피처 엔지니어링 (Feature Engineering)**
    * `일시` 데이터를 기반으로 `월`, `요일`, `요일구분`(평일/토/휴일), `hour` 등 시간 관련 피처를 생성합니다.
    * `hour`를 `sin/cos` 변환하여 시간의 주기성을 반영합니다.
    * `기온`, `습도`, `풍속`을 조합하여 `체감온도`, `온습도지수` 등 파생 변수를 생성합니다.
    * `test.csv`에 누락된 `일조(hr)`, `일사(MJ/m2)` 값을 `train.csv`의 시간대별 평균으로 보간합니다.

2.  **시계열 Lag 피처 생성 (Lag Features)**
    * **Train Set:** `1시간_전`, `하루_전(24h)`, `일주일_전(168h)` 전력소비량 피처를 생성합니다.
    * **Test Set:**
        * `일주일_전` 피처는 `train` 데이터의 해당 날짜 값을 `merge`하여 생성합니다.
        * `1시간_전`, `하루_전` 피처는 **데이터 연속성을 보장**하기 위해, `train` 데이터의 마지막 날(8월 24일)과 `test` 데이터(8월 25일 시작)를 **연결(concat)**한 후 `shift()`를 적용하여 정확하게 값을 채워 넣습니다.

3.  **데이터 분할 및 그룹화 (Grouping)**
    * `building_info.csv`의 설비 정보(`태양광용량`, `ESS저장용량`, `PCS용량`) 유무에 따라 건물을 **3개의 '장비그룹'**으로 분리합니다.
    * `train` 데이터를 8월 18일을 기준으로 `model_train_set`과 `model_valid_set`으로 분리합니다.

4.  **최종 저장 (Save Files)**
    * `train`, `valid`, `test` 데이터를 각각 3개의 '장비그룹'으로 나누어, **총 9개의 데이터프레임**을 생성합니다.
    * `optimize_memory` 함수를 통해 `float64` -> `float32` 등으로 변환하여 메모리를 최적화합니다.
    * 최종 9개의 파일을 `train_group_1.csv`, `valid_group_2.csv` 등의 이름으로 Google Drive 경로에 저장합니다.

## 🏁 최종 산출물 (Output)

이 스크립트를 모두 실행하면 `path`로 지정된 폴더에 다음 9개의 CSV 파일이 생성되며, 이 파일들은 `SAINT`, `TST` 등 딥러닝 모델의 학습 입력값으로 바로 사용됩니다.

* `train_group_1.csv` / `train_group_2.csv` / `train_group_3.csv`
* `valid_group_1.csv` / `valid_group_2.csv` / `valid_group_3.csv`
* `test_group_1.csv` / `test_group_2.csv` / `test_group_3.csv`
