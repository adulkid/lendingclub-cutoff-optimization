# PD 모델링 최종 Feature 리스트

Model A(등급 포함, sub_grade만 사용) / Model B(등급 제외) × 로지스틱회귀 / 트리(LightGBM) 4개 조합 기준.
모든 컬럼은 `loans_model` 뷰 기준(`_filled` 등은 이미 만든 파생 컬럼).

---

## 1. 그대로 사용 (결측 없음, 그룹 무관, 4개 조합 공통)

`annual_inc_filled`, `dti`, `inq_last_6mths_filled`, `loan_amnt`, `early_era`, `credit_history_months`

> 참고: annual_inc/inq_last_6mths는 초기엔 결측 0건으로 보였으나, 학습 단계에서 2007년 초기 레코드 중 소수(annual_inc 4건, inq_last_6mths 29건)의 숨은 결측이 발견돼 `_filled` 처리로 전환됐다(model.ipynb 기준으로 최신화).

## 2. 결측 처리만 거친 단독 feature (그룹 무관, 4개 조합 공통)

`collections_12_mths_ex_med_filled`, `tot_coll_amt_filled`, `pct_tl_nvr_dlq_filled`

## 3. 이력없음(그룹A 중 8그룹 밖) — 4개 조합 공통, 원값+플래그 둘 다 사용

| 원본 | 사용 컬럼 |
|---|---|
| mths_since_last_delinq | `mths_since_last_delinq_filled`, `had_delinq` |
| mths_since_last_major_derog | `mths_since_last_major_derog_filled`, `had_major_derog` |
| mths_since_recent_bc_dlq | `mths_since_recent_bc_dlq_filled`, `had_bc_dlq` |
| mths_since_recent_revol_delinq | `mths_since_recent_revol_delinq_filled`, `had_revol_delinq` |

## 4. 범주형 (grade/sub_grade 제외) — 4개 조합 공통

`home_ownership`, `purpose`, `verification_status`, `term`, `emp_length_filled`
- 로지스틱: 원-핫인코딩
- 트리: LightGBM category 타입 그대로

## 5. 구간화/이진화 특수 처리 — 로지스틱 vs 트리 다름

| feature | 로지스틱 | 트리 |
|---|---|---|
| mort_acc | `mort_acc_has` (이진) | `mort_acc_filled` (원값) |
| avg_cur_bal | `avg_cur_bal_filled` + `hinge1_avg_cur_bal` + `hinge2_avg_cur_bal` (3항) | `avg_cur_bal_filled` (원값 1개) |

## 6. 다중공선성 8개 그룹 — 로지스틱=대표 1개, 트리=그룹 전체

| 그룹 | 트리(전체 사용) | 로지스틱(대표만) |
|---|---|---|
| ①연체이력 | delinq_2yrs_filled, delinq_amnt, acc_now_delinq, num_tl_30dpd_filled, num_tl_90g_dpd_24m_filled, num_tl_120dpd_2m_filled+had_120dpd_2m_data, num_accts_ever_120_pd_filled, chargeoff_within_12_mths_filled | **delinq_2yrs_filled** |
| ②계좌수 | open_acc, total_acc, num_sats_filled, num_bc_sats_filled, num_actv_bc_tl_filled, num_actv_rev_tl_filled, num_bc_tl_filled, num_il_tl_filled, num_op_rev_tl_filled, num_rev_accts_filled, num_rev_tl_bal_gt_0_filled, acc_open_past_24mths_filled, num_tl_op_past_12m_filled | **acc_open_past_24mths_filled** |
| ③리볼빙 | revol_bal, revol_util_filled, bc_open_to_buy_filled, bc_util_filled, total_bc_limit_filled, total_rev_hi_lim_filled, percent_bc_gt_75_filled | **bc_open_to_buy_filled** |
| ④잔액한도 | tot_cur_bal_filled, total_bal_ex_mort_filled, tot_hi_cred_lim_filled, total_il_high_credit_limit_filled (+ avg_cur_bal은 5번 항목 규칙 적용) | **avg_cur_bal 계열(5번 항목)** |
| ⑤계좌나이 | mo_sin_old_il_acct_filled+had_il_acct, mo_sin_old_rev_tl_op_filled, mo_sin_rcnt_rev_tl_op_filled, mo_sin_rcnt_tl_filled | **mo_sin_rcnt_tl_filled** |
| ⑥최근사건 | mths_since_recent_bc_filled, mths_since_recent_inq_filled+had_recent_inq | **mths_since_recent_inq_filled + had_recent_inq** |
| ⑦공공기록 | pub_rec_filled, pub_rec_bankruptcies_filled, mths_since_last_record_filled+had_record, tax_liens_filled | **pub_rec_filled** |
| ⑧신용점수 | fico_range_low, fico_range_high | **fico_range_low** |

## 7. 등급 축 (Model A/B)

| | Model A | Model B |
|---|---|---|
| 포함 컬럼 | `sub_grade` (grade는 제외 — sub_grade가 상위호환) | 없음 |

- 로지스틱: sub_grade 원-핫인코딩
- 트리: sub_grade category 타입

## 8. Target / 분할 기준 (feature 아님, 참고용)

- target: `target`
- train/test 분할: `issue_year` (2007~2014 train, 2015 주test, 2016 보조test, 2017~2018 제외) — **feature로는 사용 안 함**

## 9. 최종 feature 개수 (model.ipynb 기준 검증됨)

| 조합 | 개수 |
|---|---|
| 로지스틱 B | 34개 |
| 로지스틱 A | 35개 (B + sub_grade) |
| 트리 B | 67개 |
| 트리 A | 68개 (B + sub_grade) |

> 2026-07-13 model.ipynb 실제 코드(final_cols, common/group_repr_logistic/group_full_tree/logistic_special/tree_special 정의 셀)와 전수 대조해 최신화함. 위 4곳(annual_inc, inq_last_6mths, delinq_2yrs, pub_rec)의 `_filled` 접미사 누락을 발견해 수정.
