# LendingClub 대출 승인 Cut-off 최적화: PD 모델링과 기대이익 시뮬레이션

LendingClub의 공개 대출 데이터(2007~2018, accepted 약 213만 건)를 분석해, 이자수익과 기대손실(EL)을 함께 고려했을 때 기대이익을 최대화하는 승인 cut-off를 도출하고, 방법론 검증까지 수행한 개인 프로젝트.

## 핵심 요약

- 문제: 신용대출 심사는 승인/거절 이분법처럼 보이지만, 실제로는 cut-off를 어디에 긋느냐에 따라 포트폴리오 기대이익이 크게 달라짐
- 분석: PD(부도확률)·LGD(손실률)·EAD(노출금액) 기반 EL 모델 구축 → 로지스틱/트리 4개 모델 비교(AUC보다 calibration 우선) → 승인율별 누적 기대이익 곡선으로 최적 cut-off 도출 → LGD·모델선택 민감도로 방법론 검증
- 결론: 최적 cut-off는 **부도확률 19.55% 이하(승인율 65.4%)**, 이때 기대이익 **$124.47M**(test_2015 기준). 백테스트 결과 승인 대상 실제 부도율(11.7~15.23%)이 거절 대상(36.21~39.76%)의 1/3 수준으로 일관되게 확인됨

## 전체 분석 보고서

[Notion 보고서 링크] https://app.notion.com/p/LendingClub-Cut-off-PD-39c91bab7ce980178c84db8a21956cfb?source=copy_link

## 대시보드

예정 (본 리포트 완료 후 진행 여부 결정)

## 기술 스택

Python (pandas, scikit-learn, LightGBM, seaborn, matplotlib), DuckDB (SQL), Jupyter

## 레포 구조

```
lendingclub-cutoff-optimization/
├── README.md
├── EDA.ipynb                      # [1] EDA — 이자수익 계산식, LGD, EAD, Rejected 비교
├── model.ipynb                    # [2] PD 모델링 — 4개 모델 비교, 트리A 확정
├── expected_profit.ipynb          # [3] 기대이익 시뮬레이션 — cutoff 도출, 백테스트, 민감도
├── docs/
│   ├── DEVLOG.md                  # 진행기록·의사결정 로그
│   ├── 최종_feature_리스트.md      # PD 모델 최종 feature 명세
│   ├── 여신리스크_도메인지식_정리.md
│   ├── 머신러닝_기초개념_정리.md
│   └── *_복습노트.md               # 단계별 복기 노트 (EDA/모델링/기대이익)
├── images/
│   ├── 6-1_grade_term_이자율_부도율_실현률.png
│   ├── 6-1_rejected_accepted_분포비교.png
│   ├── 6-2_calibration_curve.png
│   └── 6-3_cutoff_curve.png
└── data/
    └── (원본 CSV — 용량 문제로 미포함, 아래 안내 참고)
```

## 분석 흐름

1. EDA — 이자수익 계산식(원리금균등분할+조기상환), LGD(0.898), EAD, Rejected 비교로 EL 4개 입력값 확정
2. PD 모델링 — 발행연도 기준 out-of-time 분할(2007~2014 train / 2015·2016 test), Model A/B × 로지스틱/트리 4개 비교 → calibration 기준으로 트리A 채택
3. 기대이익 시뮬레이션 — 확률가중 기대이익 공식 확립, 승인율별 누적 기대이익 곡선으로 cutoff 도출(PD≤0.1955, 승인율 65.4%)
4. 방법론 검증 — LGD 가정(0.85/0.898/0.95), 모델 선택(트리A vs 로지스틱A), 연도(2015→2016) 3가지 민감도 분석
5. 의사결정 제안 — 비즈니스 임팩트($124.47M) 정리, LendingClub 기존 정책과의 비교 한계 명시

## 데이터

[Kaggle: wordsforthewise/lending-club](https://www.kaggle.com/datasets/wordsforthewise/lending-club). 용량 문제(원본+정제본+rejected 합계 4.5GB+)로 저장소에는 포함하지 않았습니다. 재현하려면 위 링크에서 `accepted_2007_to_2018Q4.csv`, `rejected_2007_to_2018Q4.csv`를 다운로드해 루트에 놓고 `EDA.ipynb`부터 순서대로 실행하세요. 정제 결과와 예측 CSV들도 노트북 재실행 시 자동으로 생성됩니다.

## 한계 및 확장 과제

- Reject Inference(거절자 실제 부도 추정) 미구현 — 데이터 구조상 근본적 한계
- 자금조달비용·운영비용 미반영(기대이익 = 이자수익 − EL만 사용)
- 대시보드(여신심사팀이 자체 가정을 입력해 cutoff를 재계산하는 도구)는 이후 진행 예정
