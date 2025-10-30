# Dacon 전력사용량 예측 AI 경진대회

(2025년 8월 완료한 Dacon 경진대회 프로젝트입니다.)

100개 건물의 1시간 단위 전력소비량(kWh)을 예측하는 시계열 예측 프로젝트입니다.
- **예측 기간:** 2024.08.25 00:00 ~ 2024.08.31 23:00 (총 7일)

---

## 🏆 평가 지표 (Evaluation Metric)

대회의 공식 평가 지표는 **SMAPE (Symmetric Mean Absolute Percentage Error)**입니다.

$$
\text{SMAPE} = \frac{100\%}{n} \sum_{t=1}^{n} \frac{2 \cdot |\hat{y}_t - y_t|}{|\hat{y}_t| + |y_t|}
$$

* **Public Score:** 2024.08.25 ~ 2024.08.27 (3일간의 데이터로 측정)
* **Private Score:** 2024.08.25 ~ 2024.08.31 (전체 7일간의 데이터로 측정)

---

## 💾 데이터셋 (Dataset)

총 4개의 `.csv` 파일이 제공됩니다.

### 1. train.csv
* **기간:** 2024.06.01 ~ 2024.08.24 (85일 분량)
* **내용:** 100개 건물의 1시간 단위 기상 데이터(기온, 강수량, 풍속, 습도, 일조, 일사)와 **타겟 변수인 `전력사용량(kWh)`**가 포함되어 있습니다.

### 2. building_info.csv
* **내용:** 100개 건물에 대한 메타데이터입니다.
* **주요 정보:** 건물 유형(공공, 학교, 병원 등), 연면적(m²), 냉방 면적(m²), 태양광/ESS/PCS 설비 용량 정보.

### 3. test.csv
* **기간:** 2024.08.25 ~ 2024.08.31 (7일 분량)
* **내용:** 예측 대상 기간의 1시간 단위 **기상 예보 정보** (기온, 강수량, 풍속, 습도)만 포함되어 있습니다. (**`전력사용량(kWh)` 없음**)

### 4. sample_submission.csv
* **내용:** 최종 예측 결과를 제출하기 위한 양식 파일입니다.
* `num_date_time` (건물번호와 시간으로 구성된 ID)에 맞춰 예측된 `answer` (전력사용량) 값을 기입하여 제출합니다.

---

## ⚙️ 환경 설정 (Setup)

* 본 스크립트는 **Google Colab** 환경에서 GPU를 사용하여 실행하는 것을 전제로 작성되었습니다.
* `google.colab.drive`를 사용하여 Google Drive를 마운트합니다.

## 📋 필수 파일 및 경로

스크립트를 실행하기 전, 본인의 Google Drive **`/content/drive/MyDrive/위아이티/`** 경로에 다음 4개의 원본 데이터 파일이 준비되어 있어야 합니다.

*(만약 경로가 다르다면, 스크립트 내의 `path` 변수를 수정해야 합니다.)*

* **`train.csv`**
    * **(입력)** 모델 학습 및 검증에 사용될 85일간의 원본 시계열 데이터입니다.
    * **(처리)** `building_info.csv`와 **결합(merge)**된 후, 8월 18일을 기준으로 `model_train_set`과 `model_valid_set`으로 **분리(split)**됩니다.

* **`test.csv`**
    * **(입력)** **최종 예측(final prediction)**의 대상이 되는 7일간의 기상 예보 데이터입니다.
    * **(처리)** `building_info.csv`와 결합되며, `train` 데이터의 통계량(일조/일사 등)으로 보간됩니다.

* **`building_info.csv`**
    * **(입력)** 100개 건물의 유형, 면적, 설비 정보가 담긴 메타데이터입니다.
    * **(처리)** `train.csv`와 `test.csv` 양쪽 모두에 **결합(merge)**되어, 데이터를 3개의 '장비그룹'으로 나누는 등 핵심 피처로 사용됩니다.

* **`sample_submission.csv`**
    * **(입력)** 최종 제출 양식 파일입니다. (본 전처리 스크립트에서는 참고용으로 로드됩니다.)

---

## 🚀 프로젝트 워크플로우 (Project Workflow)

본 프로젝트는 **2개의 주요 노트북 파일**로 구성되어, '전처리'와 '모델링' 단계를 명확히 분리하여 실행합니다.

1.  **`데이터_전처리.ipynb` (1단계: 전처리)**
    * **목적:** Dacon의 원본 CSV 파일 4개(`train.csv` 등)를 입력받습니다.
    * **수행:** 복잡한 피처 엔지니어링, 시계열 Lag 피처 생성, 건물 그룹 분리 등을 수행합니다.
    * **결과:** 모델 학습에 즉시 사용 가능한 **9개의 최적화된 CSV 파일**(`train_group_1.csv` 등)을 생성합니다.

2.  **`SAINT.ipynb` (2단계: 모델링 및 예측)**
    * **목적:** 1단계에서 생성된 9개의 CSV 파일을 입력받습니다.
    * **수행:** `pytorch-widedeep`의 **SAINT** 모델을 사용하여, **2-Stage 학습**(자가학습 → 미세조정)을 진행하고 3개의 그룹별 모델을 저장합니다.
    * **결과:** 저장된 모델을 불러와 **재귀 예측(Recursive Forecasting)**을 수행하고, 최종 제출 파일인 `final_answer.csv`를 생성합니다.

---

## 1. `데이터_전처리.ipynb` (Preprocessing)

원본 데이터를 가공하여 총 9개의 학습용/검증용/테스트용 CSV 파일을 생성합니다.

### 🛠️ 주요 전처리 로직
1.  **기본 피처 엔지니어링 (Feature Engineering)**
    * `일시` 데이터를 기반으로 `월`, `요일`, `요일구분`(평일/토/휴일), `hour` 등 시간 관련 피처를 생성합니다.
    * `hour`를 `sin/cos` 변환하여 시간의 주기성을 반영합니다.
    * `기온`, `습도`, `풍속`을 조합하여 `체감온도`, `온습도지수` 등 파생 변수를 생성합니다.
    * `test.csv`에 누락된 `일조(hr)`, `일사(MJ/m2)` 값을 `train.csv`의 시간대별 평균으로 보간합니다.

2.  **시계열 Lag 피처 생성 (Lag Features)**
    * **Train Set:** `1시간_전`, `하루_전(24h)`, `일주일_전(168h)` 피처를 생성합니다.
    * **Test Set:** `train` 데이터의 마지막 날과 `test` 데이터를 **연결(concat)**하여 데이터 연속성을 보장한 후, `1시간_전`, `하루_전` Lag 피처를 정확하게 생성합니다.

3.  **데이터 분할 및 그룹화 (Grouping)**
    * `building_info.csv`의 설비 정보 유무에 따라 건물을 **3개의 '장비그룹'**으로 분리합니다.
    * `train` 데이터를 8월 18일을 기준으로 `model_train_set`과 `model_valid_set`으로 분리합니다.

4.  **최종 저장 (Save Files)**
    * `train`, `valid`, `test` 데이터를 각각 3개의 '장비그룹'으로 나누어, **총 9개의 데이터프레임**을 생성합니다.
    * `optimize_memory` 함수로 메모리를 최적화하여 9개의 CSV 파일로 저장합니다.

### 🏁 최종 산출물 (Output)
* `train_group_1.csv` / `train_group_2.csv` / `train_group_3.csv`
* `valid_group_1.csv` / `valid_group_2.csv` / `valid_group_3.csv`
* `test_group_1.csv` / `test_group_2.csv` / `test_group_3.csv`

---

## 2. `SAINT.ipynb` (Model Training & Inference)

`데이터_전처리.ipynb`에서 생성된 9개의 CSV 파일을 입력받아, SAINT 모델을 학습시키고 최종 예측을 수행합니다.

### 🚀 실행 환경
* **Google Colab (GPU)**: `SAINT` 모델의 Transformer 구조는 연산량이 많으므로 **GPU 런타임** 사용이 필수적입니다.
* **핵심 라이브러리**: `pytorch-widedeep` (SAINT 모델 구현체)
* **Google Drive**: 전처리된 9개의 CSV 파일을 읽고, 학습된 모델(`model_*.pt`)을 저장하기 위해 Google Drive를 마운트합니다.

### 🛠️ 모델링 파이프라인 (2-Stage 학습 전략)
1.  **자가학습 (Self-Supervised Pre-training)**
    * **목적:** 타겟 변수 없이, 데이터의 전반적인 구조와 피처 간의 관계를 모델이 미리 학습합니다.
    * **방식:** `train_group_1, 2, 3` 데이터를 모두 합쳐, `pretrain_method='masked'`로 훈련시킵니다.
    * **결과:** 데이터의 내재된 패턴을 학습한 `base_model` 1개가 생성됩니다.

2.  **미세조정 (Supervised Fine-tuning)**
    * **목적:** `base_model`을 실제 타겟(`전력소비량(kWh)_log`) 예측에 맞게 튜닝합니다.
    * **방식:** `base_model`을 3개로 복사하여, `train_group_1`, `train_group_2`, `train_group_3` 데이터로 각각 그룹에 특화된 모델로 미세조정합니다.
    * **결과:** `group_1`, `group_2`, `group_3`에 각각 특화된 **총 3개의 최종 모델**이 생성됩니다.

### 📦 저장 파일 (Artifacts)
미세조정이 완료된 후, 3개의 그룹별로 다음 3가지 핵심 객체가 `trained_models_final/` 폴더에 저장됩니다. (총 9개 파일)

1.  **`model_{group_name}.pt`**: 모델 가중치
2.  **`preprocessor_{group_name}.pkl`**: 피처 스케일링/임베딩용 `TabPreprocessor` 객체
3.  **`y_scaler_{group_name}.pkl`**: 타겟 정규화용 `StandardScaler` 객체

### 🔮 최종 예측 (Inference)
`test.csv` (7일)에 대한 최종 예측은 **재귀 예측(Recursive Forecasting)** 방식을 사용합니다.

1.  **초기값(Seed) 설정:** `valid_group` 데이터의 마지막 7일치를 초기 입력 시퀀스로 사용합니다.
2.  **1-Step 예측:** `t+1` 시점의 전력량을 예측합니다.
3.  **시퀀스 업데이트:** `test.csv`의 `t+1` 시점 기상 예보와 2번의 예측값을 조합하여 `t+1` 레코드를 만들고, 시퀀스에 추가합니다.
4.  **반복:** 이 과정을 7일(168 스텝) 동안 반복하여 `test` 기간 전체의 예측값을 생성합니다.
5.  **결합:** 3개 그룹의 예측 결과를 `pd.concat`을 통해 `final_answer.csv`로 통합하여 제출합니다.
