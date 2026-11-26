# Korean Diary Emotion Analysis

한국어 일기에서 여러 감정을 동시에 분석하고, 날짜별 감정 기록을 누적하여 감정 달력과 긍정·부정 변화 그래프로 확인할 수 있는 **KoBERT 기반 다중 감정 분석 시스템**입니다.

기존 감정 분석 모델을 단순히 사용하는 데서 끝나지 않고, KoBERT 다중 라벨 분류 결과에 **감정별 threshold**와 **한국어 문맥 검증형 후처리**를 추가하여 일기 감정 분석에 맞게 개선했습니다.

---

## 빠른 실행

### 최종 사용자 시스템

```text
notebooks/11_diary_system_user.ipynb
```

### 최종 평가

```text
notebooks/12_diary_system_evaluation.ipynb
```

### 실행 순서

```text
1. Python 3.11 가상환경 생성
2. requirements.txt 설치
3. SKT KoBERT tokenizer 설치
4. models/kobert_kote_nall_e5.pt 배치
5. 11_diary_system_user.ipynb에서 Run All
```

> `11_diary_system_user.ipynb`는 같은 `notebooks` 폴더의 `10_diary_system.ipynb` 분석 로직을 사용합니다.  
> 따라서 10번 노트북을 삭제하거나 셀 순서를 임의로 변경하지 않는 것을 권장합니다.

---

## 1. 프로젝트 목표

일기에는 하나의 감정만 존재하지 않습니다.

예를 들어,

> 오늘 시험을 잘 봐서 뿌듯했지만 다음 발표가 걱정되고 너무 피곤했다.

와 같은 문장에는 `뿌듯함`, `불안/걱정`, `힘듦/지침` 등이 동시에 나타날 수 있습니다.

본 프로젝트에서는 다음 기능을 구현했습니다.

- 한국어 일기를 문장 단위로 분리
- KoBERT를 이용한 **44개 KOTE 정서 다중 라벨 분석**
- 감정별 threshold 적용
- 직접 언급된 감정의 문맥 검증
- 44개 세부 감정을 7개 대표 감정으로 집계
- 날짜별 일기 및 분석 결과 저장
- 감정 달력 생성
- 긍정·부정 감정 변화 그래프 생성
- 사용자가 코드 수정 없이 사용할 수 있는 Jupyter UI
- 별도 30개 데모 일기를 이용한 대표 감정 판정 확인

---

## 2. 데이터셋

### KOTE

한국어 온라인 댓글 기반 다중 감정 데이터셋인 **KOTE (Korean Online That-gul Emotions)**를 사용했습니다.

- 전체 데이터: 50,000개 댓글
- 정서 라벨: **44개**
- Multi-label 형태
- Train: 40,000
- Validation: 5,000
- Test: 5,000

프로젝트에서는 다음 파일을 사용합니다.

```text
data/raw/
├─ train.tsv
├─ val.tsv
└─ test.tsv
```

---

## 3. 모델 구조

기본 모델은 `skt/kobert-base-v1`입니다.

```text
한국어 문장
   ↓
KoBERT Tokenizer
   ↓
BertModel
   ↓
[CLS] hidden state
   ↓
Dropout
   ↓
Linear(768 → 44)
   ↓
Sigmoid
   ↓
44개 감정별 확률
```

다중 라벨 분류이므로 학습에는 `BCEWithLogitsLoss`를 사용합니다.

각 감정 확률에 threshold를 적용하여 여러 감정을 동시에 선택합니다.

---

## 4. 주요 개선 내용

### 4.1 다중 감정 분석

단일 대표 감정만 출력하는 방식 대신 44개 감정 확률을 계산하고, threshold를 넘는 여러 감정을 동시에 선택합니다.

최종 사용자 화면에서는 중요한 **세부 감정 Top 5**도 함께 보여줍니다.

### 4.2 감정별 threshold

모든 감정에 동일한 threshold를 적용하지 않고 Validation set을 이용해 감정별 threshold를 계산했습니다.

이를 통해 감정별 출력 특성 차이를 보완하고자 했습니다.

### 4.3 문맥 검증형 직접 감정 탐지

감정 단어가 포함되었다는 이유만으로 현재 작성자의 감정으로 판단하지 않도록 후처리 규칙을 추가했습니다.

검사하는 문맥의 예시는 다음과 같습니다.

- 부정 표현
  - `행복하지 않았다`
  - `안 피곤하다`
- 가정
  - `합격하면 행복할 것 같다`
- 희망
  - `행복했으면 좋겠다`
- 예상
  - `절망할 것 같았다`
- 타인 감정
  - `친구가 행복해 보였다`
- 인용 / 전달
- 과거 회상과 현재 감정 구분
- 원인절의 행동 주체와 감정 경험자 구분
  - `친구가 밥을 사 줘서 고마웠다`
  - `동생이 아파서 걱정됐다`

문맥상 현재 작성자의 감정이 아닌 것으로 판단된 세부 감정은 사용자용 Top 5와 최종 조정 점수 계산에서 제한적으로 제외합니다.

### 4.4 대표 감정

44개 세부 감정을 다음 7개 대표 감정으로 묶습니다.

```text
행복
기대
슬픔
분노
불안
피로
중립
```

대표 감정과 함께 2순위 대표 감정 및 점수 차이를 계산하여 대표 감정의 우세 정도도 확인합니다.

---

## 5. 핵심 노트북

| 파일 | 역할 |
|---|---|
| `02_kobert_multilabel_train.ipynb` | KoBERT 44개 감정 다중 라벨 모델 학습 |
| `03_kobert_singlelabel_baseline.ipynb` | Single-label baseline과 Multi-label 모델 비교 |
| `10_diary_system.ipynb` | 최종 NLP 분석 로직, 문맥 검증, 대표 감정, 그래프 및 감정 달력 |
| `11_diary_system_user.ipynb` | 최종 사용자 실행 인터페이스 |
| `12_diary_system_evaluation.ipynb` | 30개 데모 일기 대표 감정 판정 평가 |

일반적인 시연 또는 실제 사용에는 `11_diary_system_user.ipynb`를 실행합니다.

---

## 6. 사용자 기능

`11_diary_system_user.ipynb`에서는 `ipywidgets` 기반 UI를 제공합니다.

### 일기 작성

- 작성 날짜 선택
- 여러 줄 일기 입력
- 감정 분석
- 분석 후 저장
- 작성한 일기 원문과 분석 결과 확인

### 월간 감정

연도와 월을 선택하면 다음 결과를 확인할 수 있습니다.

- 월간 감정 요약
- 대표 감정 분포
- 자주 나온 세부 감정
- 감정 달력
- 일별 긍정 감정 변화
- 일별 부정 감정 변화

### 날짜별 조회

저장한 날짜를 선택하여 다음 내용을 다시 확인할 수 있습니다.

- 작성한 일기 원문
- 대표 감정
- 긍정·부정 점수
- 세부 감정 Top 5
- 사용자용 코멘트

---

## 7. 데모 데이터와 실제 사용자 데이터

11번 노트북에서는 데이터를 두 가지 모드로 분리합니다.

### 데모 데이터 30개

- 2026년 9월 1일 ~ 30일
- 행복 / 기대 / 슬픔 / 분노 / 불안 / 피로 / 중립 포함
- 모델 학습에는 사용하지 않음
- 사용자 기능 시연 및 소규모 평가용
- 기준 데이터를 보호하기 위해 데모 모드에서는 저장 기능 비활성화

### 내 일기

사용자가 직접 작성한 일기는 데모 데이터와 별도 파일에 저장됩니다.

```text
data/sample_diary_11_user.csv
```

개인 일기 파일은 `.gitignore`를 통해 GitHub 업로드 대상에서 제외합니다.

---

## 8. 30개 데모 시나리오 평가

`12_diary_system_evaluation.ipynb`에서 30개의 별도 데모 일기를 대상으로 **7개 대표 감정 판정 결과**를 확인했습니다.

> 본 평가는 모델의 일반화 성능을 대표하는 공식 벤치마크가 아니라, 최종 시스템의 대표 감정 판정 동작을 확인하기 위한 소규모 시나리오 평가입니다.

또한 아래 수치는 **KoBERT 자체의 전체 모델 정확도**가 아니라, 30개 데모 일기에 대한 **최종 시스템의 대표 감정 정답률**입니다.

| 목표 감정 | 정답 / 전체 | 정답률 |
|---|---:|---:|
| 행복 | 5 / 5 | 100% |
| 기대 | 4 / 5 | 80% |
| 슬픔 | 4 / 4 | 100% |
| 분노 | 3 / 4 | 75% |
| 불안 | 4 / 4 | 100% |
| 피로 | 4 / 4 | 100% |
| 중립 | 0 / 4 | 0% |
| **전체** | **24 / 30** | **80.0%** |

참고로 비중립 감정 사례 26개에서는 24개를 맞혔습니다.

```text
24 / 26 = 92.3%
```

이는 전체 오분류가 중립 판정에 집중되어 있음을 보여주는 참고 지표입니다.

주요 오분류:

```text
중립 → 행복 : 4회
분노 → 슬픔 : 1회
기대 → 행복 : 1회
```

현재 시스템의 가장 큰 한계는 **중립과 약한 긍정 감정의 구분**입니다.

---

## 9. 개발 과정

중간 버전은 최종 코드의 발전 과정을 확인하기 위해 유지합니다.

| 버전 | 주요 내용 |
|---|---|
| 04 | 일기 감정 분석 시스템 기본 구조 |
| 05 | 중립 / 그래프 / 감정 그룹 구성 |
| 06 | 문장 분리 및 긍정·부정 집계 개선 |
| 07 | 감정별 threshold, 직접 감정 우선 처리 |
| 08 | 문맥 검증형 직접 감정 탐지 도입 |
| 08-v2 | 실제 오판 사례 수정, Top5 보정, 대표 감정 우세도 |
| 09 | `~서` 원인절 처리 개선 및 상세 사용자 화면 |
| 10 | 최종 분석 로직 + 단순화된 사용자 카드 |
| 11 | 날짜 지정, 저장, 월간 조회, 데모/실사용 분리 UI |
| 12 | 30개 데모 일기 소규모 시나리오 평가 |

---

## 10. 프로젝트 구조

```text
korean-diary-emotion-analysis/
├─ data/
│  ├─ raw/
│  │  ├─ train.tsv
│  │  ├─ val.tsv
│  │  └─ test.tsv
│  └─ sample_diary.csv
│
├─ models/
│  └─ kobert_kote_nall_e5.pt      # GitHub에는 포함하지 않음
│
├─ notebooks/
│  ├─ 02_kobert_multilabel_train.ipynb
│  ├─ 03_kobert_singlelabel_baseline.ipynb
│  ├─ 04_diary_system.ipynb
│  ├─ ...
│  ├─ 10_diary_system.ipynb
│  ├─ 11_diary_system_user.ipynb
│  └─ 12_diary_system_evaluation.ipynb
│
├─ outputs/
├─ .gitignore
├─ README.md
└─ requirements.txt
```

---

## 11. 실행 환경 및 설치

권장 환경:

```text
Python 3.11
transformers 4.44.2
PyTorch
pandas
numpy
scikit-learn
matplotlib
ipywidgets 8.1.9
```

Windows PowerShell 기준:

```powershell
py -3.11 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
```

SKT 공식 KoBERT tokenizer 설치:

```powershell
python -m pip install "https://github.com/SKTBrain/KoBERT/archive/refs/heads/master.zip#subdirectory=kobert_hf"
```

---

## 12. 학습된 모델 파일

학습된 모델 파일은 용량 문제로 GitHub 저장소에 포함하지 않습니다.

모델 파일은 별도로 제공받아 다음 위치에 배치해야 합니다.

```text
models/kobert_kote_nall_e5.pt
```

Windows PowerShell에서 모델 파일의 SHA256 해시는 다음 명령으로 확인할 수 있습니다.

```powershell
Get-FileHash .\models\kobert_kote_nall_e5.pt -Algorithm SHA256
```

> README에는 특정 해시값을 고정해서 적지 않았습니다. 최종 제출 시 필요하다면 현재 사용하는 모델 파일의 해시를 직접 확인한 뒤 기록하는 것을 권장합니다.

---

## 13. 실행 방법

### 사용자 시스템

VS Code 또는 Jupyter에서:

```text
notebooks/11_diary_system_user.ipynb
```

를 열고 프로젝트 `.venv` 커널을 선택한 뒤 **Run All**을 실행합니다.

11번 노트북은 같은 `notebooks` 폴더의 `10_diary_system.ipynb` 분석 로직을 사용하므로 두 파일을 함께 유지해야 합니다.

### 평가

```text
notebooks/12_diary_system_evaluation.ipynb
```

을 열고 **Run All**을 실행합니다.

평가에서는 다음을 확인할 수 있습니다.

- 30개 목표 감정 vs 예측 감정
- 전체 대표 감정 정답률
- 감정별 정답률
- Confusion Matrix
- 주요 오분류 유형
- 오답 일기 목록

---

## 14. 현재 한계 및 향후 개선

현재 확인된 주요 한계는 다음과 같습니다.

1. 중립 일기가 행복 계열로 분류되는 경향
2. 여러 감정이 비슷한 강도로 나타날 때 대표 감정 1개 선택의 어려움
3. 규칙 기반 문맥 검증이 모든 한국어 표현을 완전히 처리할 수 없음
4. 30개 데모 평가는 소규모 시나리오 평가이므로 대규모 독립 평가 데이터가 추가로 필요함

향후에는 중립 판정 기준 개선과 별도의 문맥 평가 데이터셋 확장을 고려할 수 있습니다.

---

## 15. 참고 자료

- KOTE Dataset  
  https://github.com/searle-j/KOTE
- KOTE Paper  
  https://arxiv.org/abs/2205.05300
- SKTBrain KoBERT  
  https://github.com/SKTBrain/KoBERT
- Hugging Face Transformers Documentation  
  https://huggingface.co/docs/transformers
- 수업 및 구현 참고 코드  
  `deep-learning-from-scratch-2`

---

## 핵심 흐름

```text
KOTE 44개 다중 감정
        ↓
KoBERT Multi-label Classification
        ↓
감정별 Threshold
        ↓
한국어 문맥 검증형 후처리
        ↓
7개 대표 감정 + 세부 감정 Top5
        ↓
날짜별 저장
        ↓
감정 달력 / 긍정·부정 변화
        ↓
사용자 인터페이스
        ↓
30개 데모 시나리오 평가
```
