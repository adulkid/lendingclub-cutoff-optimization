# DEVLOG — LendingClub Cut-off 최적화 분석

> **이 파일의 목적**
> README = 프로젝트 헌장(변하지 않는 계획·원칙). 이 파일 = 진행하며 쌓는 시간순 기록.
> 나중에 포트폴리오 글을 쓸 때, 이 로그를 위→아래로 훑으면 "무엇을 왜 결정했고 결과가 어땠나"의 서사가 그대로 나오도록 남긴다.
>
> **기록 단위 (매 작업마다 이 4줄)**
> - **What** — 무엇을 했나
> - **Why** — 왜 그렇게 했나 (결정 + 근거). 열린 질문을 닫은 결정은 반드시 기록.
> - **Result** — 결과·수치·산출물
> - **Next** — 다음 할 일
>
> **운영 방식 (하이브리드)**
> - 평소: 작업 맥락이 뜨거울 때 Cursor에서 러프 메모를 직접 남긴다(마찰 최소화가 최우선).
> - 단계 종료 시: Claude가 러프 메모 + 노트북을 읽고 4줄 포맷으로 다듬고 의사결정 표를 채운다.
>
> 규칙: 최신 항목이 위로 오도록 **역순(최근이 맨 위)**. "왜"가 남는 것만 기록.

---

## 의사결정 로그 (Decision Log)

| ID | 질문 | 확정값 | 근거 |
|---|---|---|---|
| D1 | 데이터셋 | Kaggle `wordsforthewise/lending-club` (accepted.csv, 2007~2018, 226만 건) | PD·LGD·EAD 계산에 필요한 신청시점+사후결과 변수 모두 보유, 학계 선례 있음 |
| D2 | 데이터 소스 범위 | accepted 메인 분석 + rejected는 보조 비교용(Risk_Score, DTI 분포) | rejected는 공통변수 극소, reject inference는 스코프 밖. Risk_Score는 2013.11.5 기준 FICO/Vantage 척도 상이 — 비교 시 동일척도 기간 한정 필요(⚠️ 미해결 메모) |
| D3 | 소비자/KPI 확정 | 소비자=여신심사팀(리포트·대시보드 동일), KPI=기대이익(이자수익−EL) | 리스크관리팀·경영진 관점과 혼동 방지, 파편화 예방 |
| D4 | 산출물 구조 | 리포트(EDA+PD모델+기대이익분석→cut-off 논증) + 대시보드(가정 입력형 재계산 도구, 시나리오 확정 산출) | 대시보드 게이트(액션 유도) 충족, LGD·자금비용 등 가정 가변성 반영 |
| D5 | Feature 컬럼 1차 확정 | 151개 원본 중 leakage 35개 제외, 텍스트 3종(desc/title/emp_title) 제외, joint 16개 별도 표기 → feature 후보 81개 | 신청시점 정보만 남기는 leakage 필터링 |
| D6 | grade/sub_grade 처리 | Model A(포함)/Model B(제외) 병행 비교. 목적은 성능비교 아닌 "LC 등급 대비 우리 재평가 괴리 구간 발견" | LC 자체 심사결과 feature화 시 순환구조 우려. DA 프로젝트 목적 유지 |
| D7 | 컬럼 최종 정정 | member_id/url/zip_code/addr_state/initial_list_status/disbursement_method/application_type(필터 후) 드랍. last_fico_range_high/low는 feature→leakage 재분류(37개). id·int_rate는 feature 아님으로 보존 | zip_code/addr_state 중복 지리정보 제외. last_fico는 대출 실행 후 재조회값이라 leakage |
| D8 | Joint 최종 처리 | joint 신청 row 전체 제외(application_type='Individual'만 유지) + joint 관련 16개 컬럼 드랍 | 비중 5.7%, 개인신청과 데이터 구조(2인분 feature) 자체가 달라 스코프 제외 |
| D9 | 39.97% 결측 컬럼군 처리 | il_util/all_util 등 14개 컬럼 드랍, 전체 기간(2007~2018) 유지 | issue_year별 결측률 확인 결과 2015년 이전 100% 결측 → 신규 수집 필드로 판명. 대체 지표(revol_util 등) 존재, 표본 크기(부도 소수클래스 학습) 우선 |
| ~~D10~~ → D14 | Target(loan_status) 정의 | **D14에서 확정됨** (아래 참조) | 07-08 전체분포 재확인 후 07-10 EDA에서 D14로 종결 |
| **D11 (CLOSED 07-11)** | EL 구성요소 계산 방식 | PD=모델링(feature 66개). **LGD=0.898 단일 pooled**(안정기간 2012~2016, n=129,991). EAD=funded_amnt(검증 완료). 이자수익=상각식×prepayment factor(D15) | 통제 가능한 변수는 cutoff 하나뿐. **원안(LGD 세그먼트별 평균)은 등급별 편차가 작아(0.897~0.938) 단일값으로 수렴** — 계획이 데이터를 보고 바뀐 케이스. 2011 이전 crisis-era 저회수·2017~2018 censoring 제외하고 안정 vintage만 pooled |
| D17 | debt_settlement_flag 처리 | LGD 세그먼테이션 기준에서 **제외** | 사후 행동변수(대출 후 합의) → 심사 시점에 알 수 없는 정보로는 분기하지 않는다는 원칙 확정 |
| D18 | Rejected 비교 범위 종결 | Risk_Score/DTI 비교로 종료. 추가 볼 컬럼 없음 | 척도 분리(2013-11-05 이후 한정) + DTI 이상치(dti≤100) 필터 후 재비교 → "rejected가 신용점수 낮고 DTI 넓다"는 상식적 결과 확인, 그 이상 정보 없음 |

### 모델링 계획 확정 (D19~D27, 2026-07-11)

| ID | 질문 | 결정 | 근거 |
|---|---|---|---|
| D19 | train/test 분할 | 2007~2014 train(2007~2011=early_era 플래그), 2015 주test, 2016 보조test, 2017~2018 제외 | out-of-time 검증. 2017~2018은 censoring 심각(부도 미확정)이라 test 신뢰 불가 |
| D20 | 결측치 처리 규칙 | 그룹A(이력없음 mths_since_* 6개)=sentinel(−1)+개별플래그 / 그룹B(2013 수집시작 뭉치)=sentinel만+early_era로 대체 / 그룹B'(mo_sin_old_il_acct, num_tl_120dpd_2m)=sentinel+early_era+전용플래그 / emp_length='Unknown' / 미미결측=카운트류 0-fill·revol_util median / earliest_cr_line→credit_history_months 파생, 잔여 29건 sentinel | "결측=값없음"이 아니라 원인별 처리. **검증·학습 중 반올림에 숨은 미세결측이 계속 드러남**: 4.3~4.8% 뭉치(bc_util·num_sats 등 6개)·3.78% 뭉치(mort_acc·acc_open_past_24mths 등 4개)→그룹B 편입, annual_inc 4건(median), inq_last_6mths·delinq_2yrs·pub_rec 각 29건(2007 초기 동일 행, 0-fill) — sklearn NaN 에러로 뒤늦게 발견 후 처리. **최종 NULL 0건.** |
| D21 | 다중공선성 8그룹 | 로지스틱=대표 1개만 / 트리=그룹 전체 멤버 | 다중공선성은 선형모델만의 문제. **원안은 약한상관 그룹(①⑦)만 절충이었으나 원칙 일관성 위해 8개 그룹 전부 통일** |
| D22 | avg_cur_bal(역U자 비단조) | hinge 회귀 3구간(knot1=1633, knot2=7317) — 로지스틱 전용, 트리는 원값 | 상관계수로 못 잡는 역U자를 선형모델에 반영. 트리는 분할로 자체 포착 |
| D23 | mort_acc(문턱효과) | `mort_acc_has` 이진더미 — 로지스틱 전용, 트리는 원값 | 있/없 경계가 핵심이라 이진화가 선형모델에 유리 |
| D24 | 클래스 불균형(80.1/19.9) | 재조정 없이 원비율 학습, calibration 확인 후 문제 시에만 후처리 재보정 | PD를 확률값 그대로 EL에 넣으므로 인위적 리샘플링이 확률 왜곡 위험 |
| D25 | Model A/B 등급축 | A=`sub_grade`만(grade는 상위호환이라 제외) / B=등급정보 전부 제외 | 순환구조 우려(D6). grade+sub_grade 동시투입은 중복 |
| D26 | 범주형 인코딩 | 로지스틱=원-핫 / 트리=LightGBM category 타입 | 각 모델 생태계 표준 |
| D27 | issue_year 취급 | feature에서 제외, train/test 분할 기준으로만 사용 | 미래 데이터엔 못 쓰는 정보 → feature화 시 leakage 우려 |
| D28 | 로지스틱 전처리 | StandardScaler 표준화 필수 | 수렴 경고 해소 + AUC 0.64→0.70대로 개선. 트리는 스케일 불변이라 불필요 |
| **D29 (CLOSED)** | 최종 PD 모델 | **트리A (등급포함 LightGBM)** 확정. XGBoost 등 추가 실험은 생략 | AUC·KS·Brier·calibration 전부 test_2015·2016 양쪽 1위. 무게중심 원칙상 모델 실험 확대 = DS 흉내 경고 |

### 기대이익 분석 (D30~D33, 2026-07-12)

| ID | 질문 | 결정 | 근거 |
|---|---|---|---|
| D30 | 기대이익 분석 모집단 | test_2015(주test, out-of-time 유지) | 학습에 안 쓴 미래 연도로 검증. **전체 히스토리 아닌 단일 연도만 쓴다는 한계 명시**(2016은 백테스트 보조) |
| D31 | 이자수익 반영 방식 | **확률가중(b)**: 이자수익(조건부)×(1−PD) | "무조건 다 받는다"(a)는 부도확률과 모순되는 이중잣대 → 기각. 예시 숫자로 부호 뒤집힘 확인 후 결정 |
| D32 | 기대이익 공식(5단계) | ①월상환액(원리금균등) → ②상각 기대이자 → ③이자수익(조건부)=②×등급×term 실현률 → ④EL=PD×LGD×EAD → ⑤**기대이익=(1−PD)×③−④** | EDA 이자수익 보정공식 + EL을 한 식으로 결합 |
| D33 | 실현률 재계산 censoring 필터 | 만기 지난 대출만(issue_date+term ≤ 2019-03-01) 필터 후 재계산 | 최초엔 필터 누락 → 최근 vintage 초고속상환자만 관측돼 실현률 과소. **필터 추가 후 EDA 원본 차트값과 일치 확인**(버그 수정) |
| D12 | SQL/pandas 역할 분리 | DuckDB(대량 스캔·집계·profiling) → pandas(탐색·시각화·모델링) | DuckDB가 CSV 직접 쿼리 가능해 별도 DB 적재 불필요. seaborn/scikit-learn은 pandas 생태계 기준이라 모델링 단계는 pandas 유지 |
| D13 | 분석 환경 확정 | conda env `lendingclub` (python 3.11.15, pandas 3.0.3, duckdb 1.5.4, seaborn, scikit-learn, jupyter). 기존 `ecommerce` env 삭제, base 비워둠 | 프로젝트별 환경 분리로 base 오염 방지. **컬럼수 112 확인**(152=원본151+파생 issue_year1, −drop40=112. README/CLAUDE.md "111열" 표기는 오타로 확정) |
| **D14 (CLOSED)** | Target(loan_status) 정의 | 부도(1)=Default·Charged Off·정책플래그CO / 정상(0)=Fully Paid·정책플래그FP / 제외=Current·Grace·Late 전체 | 바젤 90일 기준상 Default(~121일+)는 확정 부도, Late(31-120)은 90일 경계 걸쳐 애매→제외. Current(37%)는 censoring이라 라벨 불가. **결과: 정상 1,059,283 / 부도 263,010 / 제외 817,665 → 학습모집단 1,322,293건(부도율 19.9%)** |
| **D15 (CLOSED)** | 이자수익 계산 공식 | 기대이자수익 = 원리금균등분할 총이자 × 조기상환보정계수(term별). 36개월=×0.807, 60개월=×0.706 (등급별 세분 계수도 산출) | naive공식(funded_amnt×int_rate×term/12)은 불릿/예금식 가정이라 실제 대비 2배+ 과대추정. 원인 분해: 할부구조(55~58%) + 조기상환. 등급별 단조감소 A(83.7/76.8%)→G(71.9/64.3%) |
| **D16 (CLOSED)** | PD feature 다중공선성 정리 | 8개 그룹(연체이력/계좌수/리볼빙/잔액한도/계좌나이/최근사건/공공기록/신용점수) 내부 상관 확인 후 대표 feature 선정. fico_range_low/high→avg_fico 통합 | \|r\|>0.7 기준 중복 제거. 계좌수 13→3, 리볼빙 7→3, 잔액한도 5→1 축소. 나머지 그룹은 중복 적음 |

---

## 진행 기록 (최근이 맨 위)

### 2026-07-16 — 시각화 5종 확정 & GitHub 퍼블리시 준비
- **What** — 노션초안 `[시각화: ...]` 5곳 전부 확정(등급×기간 이자율/부도율/실현률 3×2 barplot, rejected/accepted 신용점수·DTI 분포비교, 4개모델 AUC/KS/Brier 표, calibration curve, 승인율별 누적기대이익 곡선) → [4] 의사결정 제안 완료. 이후 리포트 전체를 노션 페이지로 이관하고 로컬 `의사결정제안_노션초안.md`는 삭제(원본은 노션에서 관리). GitHub 업로드 준비: 문서류 `docs/`·시각화 `images/`로 폴더 재구성, 노트북 4개 셀에 `savefig` 추가해 재실행 시 이미지 자동 저장되도록 처리, `.gitignore`(원본 대용량 CSV·예측 CSV·xlsx 제외) 및 GA4 프로젝트 양식을 참고한 `README.md` 작성. 불필요 파일 점검(`describe.ipynb` 외 캐시·체크포인트 없음 확인, 삭제는 사용자가 직접 진행).
- **Why** — 시각화는 매번 직접 계산 결과를 넘겨 검토받는 방식(코드 제공 원칙)으로 진행 — 등급×기간 지표는 sub_grade 대신 grade로, 시계열 대신 barplot으로 압축(본문 맥락과 결 맞추기). 리포트 최종본은 노션에서 직접 편집·공개하는 게 목적에 맞아 로컬 마크다운 초안은 이관 후 정리. GitHub는 대용량 원본 데이터를 포함하지 않고 Kaggle 링크로 재현 안내하는 방식이 표준.
- **Result** — `images/`에 PNG 4개 저장 경로 확정(테이블 차트는 이미지 없이 표로만 제공). `README.md`가 노션 리포트 링크를 직접 연결. git init/commit은 샌드박스 마운트 제약으로 사용자 로컬 터미널에서 진행 필요(안내 완료).
- **Next** — 사용자가 로컬에서 `describe.ipynb` 삭제 후 `git init`~`git push`로 GitHub 퍼블리시 마무리. 이후 대시보드 진행 여부 결정.

### 2026-07-13 — [4] 의사결정 제안 노션 초안 작성 & 문서 최신화
- **What** — `의사결정제안_노션초안.md` 신규 작성([4] 제안 + EDA~기대이익 리포트 전체를 노션 이관용으로). GA4 노션 템플릿 구조(번호 섹션·aside 콜아웃·표) 참고. 가독성 다회 반복(Target 표화, 영어용어 우리말화, 무게중심 맞춰 소단락 차등 압축, 방법론검증 표→①②③ 카드, −143달러 표현 정확화). 문제해결 5→3 통합, 회고 단순화. 문서 최신화: `최종_feature_리스트.md`를 model.ipynb 실코드와 전수 대조.
- **Why** — [4] 산출물을 노션 포트폴리오로 옮기기 전 초안 확정. "−143달러"는 회계손실이 아니라 **LC 기존 승인 풀 기준 모델 기대값**임을 명시해 오해 방지. feature 문서가 실제 학습코드와 어긋나면 재현성이 깨지므로 코드를 소스오브트루스로 대조.
- **Result** (실물 대조 완료)
  - `최종_feature_리스트.md`: **annual_inc / inq_last_6mths / delinq_2yrs / pub_rec 4곳 `_filled` 접미사 누락 발견·수정** (model.ipynb `final_cols`·`group_repr_logistic`와 일치 확인). 숨은 결측(2007 초기 소수)이 학습 중 `_filled` 처리로 전환됐던 이력 반영.
  - **최종 feature 개수 검증표 추가**: 로지스틱 B34/A35, 트리 B67/A68.
  - `LendingClub_컬럼분류_최종v2.xlsx` → "완료·보관용"(최종_feature_리스트.md가 대체) 상태 표시. README 체크리스트 중복 제거·도메인공부 완료·[4] 진행중 반영. CLAUDE.md 문서지도에 노션초안 등록.
  - **프로젝트 시작일 확정: 2026-07-01**(주제 탐색 포함), 노션초안 개요 반영.
- **Next** — 노션초안 `[시각화: ...]` 자리 **5곳** 제작: ①등급×기간 barplot ②rejected/accepted 분포비교 ③4개모델 성능비교 ④calibration curve ⑤승인율별 누적기대이익 곡선. 완료 시 [4] 의사결정 제안 사실상 마무리.

### 2026-07-12 — 기대이익 분석 (심장, 무게중심 40%) 완료
- **What** — `expected_profit.ipynb` 신규. model.ipynb에서 트리A/로지스틱A의 test_2015·2016 PD 예측을 CSV로 내보내 로드 → EL·이자수익 결합공식 확정·구현(D31·D32) → PD 오름차순 정렬 후 누적기대이익 곡선으로 최적 cutoff 도출 → 실제 target으로 백테스트 → LGD·모델선택 민감도 분석.
- **Why** — "이자수익을 무조건 받나 vs 부도확률만큼 깎나"가 결과 부호를 뒤집을 문제라 예시로 검증 후 확률가중 채택(D31). 실현률 재계산 중 censoring 필터 누락 버그 발견해 원본 대조 수정(D33). AUC/KS만이 아니라 **실제 기대이익(비즈니스 지표)으로도** 트리A 우위를 교차확인.
- **핵심 결과 (노트북 실물 대조 완료)**
  - **전체 승인 시 평균 기대이익 마이너스(−$143/건, test_2015)** → cutoff 분석의 필요성 자체를 입증.
  - **최적 cutoff(트리A, LGD=0.898): PD ≤ 0.1955, 승인율 65.40%, 누적 기대이익 $124.47M.**
  - **백테스트**: 승인군 실제 부도율 11.7%(2015)/15.23%(2016) vs 거절군 36.21%/39.76% — 2개 연도 모두 3배+ 분리, 안정성 확인.
  - **방법론검증① LGD 민감도**: 0.85→cutoff 0.2121·승인율 69.58%·$142.1M / 0.898→0.1955·65.40%·$124.5M / 0.95→0.1793·60.83%·$107.8M. 방향 안정(승인율 60~70%대), 절댓값은 가정에 유동 → **대시보드에서 LGD를 입력받아야 하는 근거.**
  - **방법론검증② 모델선택 민감도**: 트리A($124.47M / cutoff 0.1955 / 65.40%) vs 로지스틱A($113.03M / 0.1883 / 63.19%) — 기대이익 **약 10% 우위**(로지스틱A 대비 +10.1%, 또는 트리A 기준 9.2%). 트리A 선택이 통계지표뿐 아니라 비즈니스지표로도 정당화.
- **Next** — **[4] 리포트(EDA+PD모델+기대이익 통합 서술, 가정·한계 명시)** vs **대시보드 설계(LGD·자금비용·승인율 제약 입력 → cutoff 실시간 재계산)** 중 어디부터 갈지 다음 세션 논의.

### 2026-07-11 (밤) — PD 모델 4종 학습·평가 → 트리A 확정 (모델링 완료)
- **What** — train(2007~2014)/test(2015 주·2016 보조) 추출 → 로지스틱·LightGBM × Model A/B **4개 모델 학습** → AUC/KS/test_2016 재검증/calibration curve/Brier로 종합 평가 → **트리A 최종 확정**. 노트북 구조 정리(마크다운 헤더 10개, calibration curve seaborn 재시각화, 4개 모델 종합비교표).
- **Why** — feature engineering은 익숙한 SQL로 직접 구현·단계검증, sklearn/LightGBM은 새 문법이라 매 줄 설명하며 진행. "결측 처리 끝"이라 본 뒤에도 반올림에 숨은 미세결측이 학습 시 NaN 에러로 계속 드러나 그때그때 진단·처리(D20). 최종 모델 선택은 AUC/KS만이 아니라 **calibration(확률 절댓값 정확도)** 까지 봄 — PD를 EL에 절댓값 그대로 넣기 때문.
- **Result** (노트북 실물 대조 완료)
  - **트리A**: test_2015 **AUC 0.7402 / KS 0.3511**, test_2016 **AUC 0.7189 / KS 0.3162**, **Brier 0.14216** — 4개 지표 전부 양쪽 연도 1위. 두 test 연도 순위 안 흔들려 안정적 결론.
  - 비교표(cell 29): 로지스틱B(0.7201/0.3209) < 로지스틱A(0.7344/0.3419) ≈ 트리B(0.7339/0.3402) < **트리A(0.7402/0.3511)**.
  - calibration: 트리 계열은 고위험 구간까지 안정, 로지스틱은 최고위험 구간에서 과대예측 붕괴.
  - `머신러닝_기초개념_정리.md`·`최종_feature_리스트.md` 생성, model.ipynb Run All 검증 통과.
- **Next** — **[3] 기대이익 분석(무게중심 40%, 심장)**: EL=PD×LGD×EAD 결합 → 이자수익−EL 곡선 → 최적 cut-off 도출 → 방법론 검증. 모델링급 규모 예상, 다음 세션 시작.

### 2026-07-11 (오후) — 모델링 계획 확정 & feature engineering 구현
- **What** — 모델링 전체 조건 확정(train/test 연도 D19·결측처리 D20·다중공선성 대표 D21·비선형 구간화 D22/D23·불균형 D24·등급축 D25·인코딩 D26·issue_year D27) + `머신러닝_기초개념_정리.md`·`최종_feature_리스트.md` 문서 작성 + `model.ipynb`에서 SQL로 feature engineering 구현(`loans_model` 뷰).
- **Why** — EDA에서 발견한 비선형(avg_cur_bal)·문턱(mort_acc)·결측패턴을 실제 모델 입력으로 쓰려면 처리방식을 코드 전에 문서로 명문화해야 함. 코드 경험이 적어 익숙한 SQL로 feature engineering 후 매 단계 결과를 검증하는 방식 채택. 검증 중 누락 결측 뭉치(3.78%/4.3~4.8%)·earliest_cr_line 29건을 추가 발견해 보완.
- **Result** — `loans_model` 뷰 완성(early_era, 결측처리 다수, mort_acc_has, avg_cur_bal hinge, emp_length_filled, credit_history_months) — 단계별 검증 + **최종 NULL 0건 확인**. Model A/B × 로지스틱/트리 **4개 조합 최종 feature 명세 확정**. (실물 대조 완료: hinge knot 1633/7317, COALESCE −1 sentinel, had_* 플래그 확인)
- **Next** — 확정 feature 기준 train(2007~2014)/test(2015 주·2016 보조) 추출 → sklearn/LightGBM으로 4개 모델 학습 → AUC/KS/calibration 평가.

### 2026-07-11 (오전) — LGD·EAD·Rejected EDA 마무리 (EDA 전체 완료)
- **What** — LGD 확정(추심 도메인 학습 → censoring 검증 → settlement_flag 분포 확인 → 안정기간 2012~2016 등급별 pooled LGD 산출), EAD=funded_amnt 근거 검증, Rejected vs Accepted 비교(Risk_Score/DTI), 모델링 기초 개념 학습(로지스틱회귀 vs 트리, AUC/KS/calibration), EDA 노트북 구조 정리.
- **Why** — LGD는 vintage별 censoring(2018년물 추심 미종료)과 crisis-era(2007~2010) 저회수율이 뒤섞여 단순 평균 불가 → 안정 구간 pooled 방식 필요(D11). debt_settlement_flag는 사후 행동변수라 세그먼테이션에서 제외(D17). Rejected 비교는 척도 불일치·극단 이상치 두 함정을 다 짚어야 신뢰 가능(D18).
- **Result** — 검증 수치(노트북 대조 완료): **LGD pooled = 0.898**(n=129,991 / 연도별 2012~2016 = 0.902·0.885·0.889·0.902·0.910 / 등급별 0.897~0.938로 편차 작음 → 단일값 수렴). **EAD=funded_amnt**(out_prncp>funded_amnt 0건, total_rec_prncp>funded_amnt 136건 전부 Fully Paid=조기완납). Rejected는 신용점수 낮고 DTI 분포 넓다는 상식적 결과 확인 후 종료. **→ EL 4개 입력값(PD·LGD·EAD·이자수익) 전부 확정, EDA 전체 완료.**
- **Next** — 모델링 공부 후 복귀. Model A(등급 포함)/B(등급 제외) 비교 실험(로지스틱회귀+트리 병행)부터 시작. → 이후 cutoff별 기대이익 곡선 산출 가능 상태.

### 2026-07-10 — 이자수익 EDA + PD feature EDA
- **What**
  1. 이자수익 EDA: 연도별/등급별 int_rate 차이, 등급 외 결정요인 탐색(잔차분석 → 없음 확정), 연도별 기대 vs 실제 이자수익 비교 → **관측시점 절단(2019-03 스냅샷) 발견** → 만기 지난 코호트로 재계산 → 원리금균등분할+조기상환 이중효과 규명, term/등급별 보정계수 산출 (D15)
  2. Target(loan_status) 정의 확정 (D14)
  3. PD feature 66개 프로파일링(SUMMARIZE), 성격별 분류(금액/비율/카운트/개월수/신용점수/범주형)
  4. 결측 원인 구분: ~5.31% 클러스터=구조적 결측(2012년 중반 이전 미수집, tot_cur_bal로 검증), mths_since_* 6종=이력없음 결측(카운트 컬럼 대조, 5개 확정 / major_derog 재검증 후 확정)
  5. 이상치/sentinel 탐지: tot_hi_cred_lim·total_rev_hi_lim=9999999, mo_sin_old_il_acct=999, dti<0, annual_inc>100만(290건), earliest_cr_line 1960년 이전(211건) 등 → 전부 전체 0.02% 이하, 삭제보다 NULL치환/트리모델 강건성에 위임
  6. target raw correlation 스크리닝(수치 57 + 범주 7): fico(−0.132) 최강, sub_grade(0.494)/grade(0.435) 압도적, term은 int_rate엔 무관했으나 실제 부도율엔 유의(0.166) → **D6 재평가 후보**
  7. 8개 중복의심 그룹 다중공선성 검증 (D16), 상위 feature 8개 10분위 비선형관계 확인
  8. 클래스 불균형 확인(부도 19.9%), Model A/B 실험 **설계만 확정**(실행은 모델링 단계로 이연)
- **Why** — 이자수익 naive공식은 상환구조·조기상환 미반영 → cutoff에 쓸 수익 추정치 정확성 확보 필요. feature 66개라 개별 EDA 비효율 → 스크리닝(상관·결측구조·다중공선성) 후 대표만 정밀 확인. 다중공선성 미정리 시 선형모델 계수 불안정.
- **Result**
  - 이자수익 공식 확정: `할부 총이자 × 조기상환보정계수`(36m=0.807, 60m=0.706, 등급별 세분 계수 완료) (D15)
  - Target 확정: 학습모집단 **1,322,293건, 부도율 19.9%** (D14)
  - 결측 2종 원인 구분 + 처리방침 수립
  - PD feature 다중공선성 정리(계좌수 13→3, 리볼빙 7→3, 잔액한도 5→1) (D16)
  - 비선형 발견: avg_cur_bal 역U자형(상관계수로 포착 불가), mort_acc 문턱효과(0/1 이진화 후보)
  - Model A/B 설계안: 지표 AUC+KS, 분할 time-based(out-of-time), 로지스틱회귀 우선, "괴리구간"=Model B 예측확률 구간화 × 실제 grade 교차표 — **미실행**
- **Next**
  1. **LGD EDA**: issue_year·등급별 recoveries 완결도(censoring) → seasoning cut-off 연도 결정, 완결 vintage 손실율. term별 LGD 차이 검토 (D11 오픈 닫기)
  2. **EAD EDA**: 대출단위 원금증가 케이스(out_prncp/total_rec_prncp vs funded_amnt), LGD seasoning과 연동(total_rec_prncp도 censoring 가능성)
  3. LGD/EAD 마무리 → PD·LGD·EAD·이자수익 전부 갖춰짐 → **cutoff별 기대이익 곡선 산출 가능 상태**
  4. Model A/B 실험 실행 (모델링 단계 진입)

### 2026-07-08 — DuckDB 분석환경 세팅 & 데이터 조회 검증
- **What** — conda 가상환경(`lendingclub`) 신규 생성 후 duckdb 설치, `accepted_cleaned_v1.csv`를 DuckDB로 조회 가능한지 검증.
- **Why** — 앞으로 EDA·LGD/EAD 계산 등 대량 집계는 DuckDB(SQL), 탐색·시각화·모델링은 pandas로 역할 분리(D12). 기존 conda base에 패키지를 쌓지 않고 프로젝트 전용 env로 분리(D13).
- **Result** — 설치 확인: duckdb 1.5.4, pandas 3.0.3. `read_csv_auto`로 CSV 조회 성공, 행 수 **2,139,958**(기존 기록과 일치). loan_status **전체데이터** 분포 재확인: Fully Paid 1,057,295 / Current 787,191 / Charged Off 262,215 / Late(31-120) 19,289 / In Grace Period 7,342 / Late(16-30) 3,843 / Does not meet credit policy·Fully Paid 1,988 / Does not meet credit policy·Charged Off 761 / Default 34. → 기존 100만 샘플엔 없던 **"정책 미충족" 2종(총 2,749건) 신규 확인**, target 정의(D10) 시 처리 필요.
- **Next** — ① 작업환경을 VS Code + Jupyter(`lendingclub` 커널)로 전환 ② target(loan_status) 정의 확정(D10) ③ LGD 세그먼트 계산용 recoveries 완결도(vintage censoring) EDA로 확인(D11 오픈)

### 2026-07-06 (누적) — 컬럼 정제 & 1차 클린 데이터셋 구축
- **What** — (1) accepted 151개 컬럼 딕셔너리 한글 매핑 완료 (2) 컬럼 role 분류(feature/leakage/target/LGD·EAD용/텍스트) 1차~최종 확정 (3) emp_title 직업군 묶기 시도 후 포기(정규화해도 상위30 누적비율 0.16) (4) title=purpose 중복 확인, desc 제외 (5) joint row+컬럼 전체 제외 확정 (6) last_fico_range_high/low leakage 재분류, disbursement_method·application_type 드랍 확정 (7) 39.97% 결측 14개 컬럼 issue_year 기준 검증 후 드랍 확정
- **Why** — 150+ 컬럼을 신청시점/정보량/구조적 결측 기준으로 걸러 모델링 가능한 클린 데이터셋 구축. (D5~D9 결정)
- **Result** — 151개 원본 → row 2,260,701건 → **2,139,958건**(joint 제외) / 컬럼 125개(joint+텍스트+기타 드랍) → **111개**(39.97% 결측군 드랍) 최종. **`accepted_cleaned_v1.csv` 저장**
- **Next** — loan_status(target) 정의 확정(D10: Current 포함여부, 연체구간 처리) → EDA 설계 착수 (그룹별 중복 feature 상관관계 확인 포함: 계좌수/신용카드/리볼빙/연체이력 등 8개 그룹)

### 2026-07 이전 — 문제정의·게이트·데이터셋 확정
- **What** — 프로젝트 정체성(DA, cut-off 최적화), 소비자(여신심사팀), KPI(기대이익 = 이자수익 − EL), 의사결정(cut-off 확정), 5단계 구조, 무게중심(EDA 15 / PD 20 / 시뮬 40 / 제안 25)까지 확정. 데이터셋 선정(D1).
- **Why** — "AUC 달성"으로 끝나는 DS 흉내를 방지하고 의사결정 산출물로 귀결시키기 위해 착수 전 게이트를 못박음. 상세는 `lendingclub_project_README.md` 참조.
- **Result** — README(헌장) 완성. 문제정의·소비자·KPI·게이트 확정.
- **Next** — 도메인 공부(PD/LGD/EAD/EL 등) 병행하며 데이터 실물 스캔 → leakage 선별.

---

## 마일스톤 체크리스트
- [x] 데이터셋 확정
- [x] 소비자/KPI/산출물 구조 게이트 통과
- [x] 컬럼 1차 분류 (feature/leakage/텍스트)
- [x] Joint 스코프 최종 확정 (제외)
- [x] 결측치 원인 기반 컬럼군 드랍 (39.97% 그룹)
- [x] 1차 정제 완료 (row/컬럼 필터링 112열, `accepted_cleaned_v1.csv` 저장)
- [x] Target(부도) 정의 확정 (D14 — 학습모집단 132만건, 부도율 19.9%)
- [x] 분석환경 확정 (DuckDB+conda `lendingclub`, D12·D13)
- [x] 이자수익 공식 확정 (D15)
- [x] [1] EDA — 이자수익 + PD feature (결측구조·상관·다중공선성·비선형·불균형)
- [x] [1] EDA — LGD(0.898 pooled)/EAD(funded_amnt)/Rejected 비교 → **EDA 전체 완료**
- [x] EL 4개 입력값 전부 확정 (PD·LGD·EAD·이자수익)
- [x] [2] PD 모델링 — 계획·feature engineering 완료 (`loans_model` 뷰, NULL 0건, 4개 조합 명세)
- [x] [2] PD 모델링 — 4개 모델 학습·평가 → **트리A 확정**(AUC 0.7402/KS 0.3511, D29)
- [x] [3] 기대이익 시뮬레이션 — **최적 cutoff PD≤0.1955, 승인율 65.4%, $124.47M**(D30~D33)
- [x] 방법론 검증 — 백테스트(2연도) + LGD·모델선택 민감도 완료
- [x] [4] 의사결정 제안 — 시각화 5종(등급×기간 barplot, rejected/accepted 분포비교, 모델 성능표, calibration curve, cutoff curve) 전부 확정, 완료
- [x] [5] 리포트 완성 — 노션 이관 완료(로컬 초안 삭제, README가 노션 링크 직접 연결)
- [ ] GitHub 퍼블리시 — 로컬 준비(폴더 재구성·README·gitignore) 완료, git init/commit/push는 사용자 로컬에서 진행
- [ ] 대시보드 진행 여부 결정 / 한계·확장(Reject Inference, 단일연도 등)
