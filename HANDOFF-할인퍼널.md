# 할인퍼널(discount-me) 파이프라인 반영 지시서

> 다른 노트북(데이터 파이프라인 `fetch_data.py` / `fetch_ga4.py` / `ga4_config.json` 보유)에서
> 이 내용을 반영해야 **파이프라인 재실행 후에도 할인퍼널이 유지 + 최신화**됩니다.
> (이 문서는 배포용 저장소에서 손으로 먼저 반영해 둔 것을, 파이프라인에도 이식하기 위한 안내입니다.)

## 0. 월요일 작업 순서 (중요)
1. **먼저 `git pull`** — 이 저장소는 손수정 커밋(할인퍼널 관련 5개)이 앞서 있음. pull 안 하면 재생성/푸시 시 그 수정이 덮어써짐.
2. 아래 1~4를 `ga4_config.json` + 스크립트에 반영.
3. `fetch_data.py`, `fetch_ga4.py` 재실행 → `data.json`/`ga4_data.json` 재생성.
4. 재생성 결과에 할인퍼널이 그대로 나오는지 확인 후 커밋/푸시.

## 1. 할인퍼널 정의
- **id**: `discount-me`
- **label(카드 제목)**: `할인퍼널 (impact-me 랜딩 · 7/15~)`
- **hostname(GA4)**: `impact-me.real-me.co.kr`  ← 진단퍼널(impact-me)과 **같은 호스트**
- **메타 캠페인**: `Conv_260715_disc50_AB`
- **시작일**: 2026-07-15 (impact-me 랜딩을 할인 내용으로 전면 교체)

## 2. 핵심 로직 — 같은 호스트를 날짜로 분리
impact-me 랜딩(호스트 `impact-me.real-me.co.kr`)을 **날짜 기준으로 두 퍼널로 분리**한다.
- **~2026-07-14**: 진단퍼널(`impact-me`)
- **2026-07-15~**: 할인퍼널(`discount-me`)

→ 설정만으로 안 되면 `fetch_ga4.py`/`fetch_data.py`에 "이 호스트는 CUT(2026-07-15) 이전/이후로 landing_id를 분기" 하는 코드가 필요.

### fetch_data.py (메타 → data.json)
- `landing_buckets`에 discount-me 추가:
  ```json
  { "id": "discount-me", "label": "할인퍼널 (disc50)", "campaign_names": ["Conv_260715_disc50_AB"] }
  ```
- `landing_metrics` 생성 시 `Conv_260715_disc50_AB` 캠페인(=할인 광고비/방문/리드)을 `landing_id: "discount-me"`로 집계. (진단 impact-me는 기존 캠페인만, ~7/14)

### fetch_ga4.py (GA4 → ga4_data.json)
- `impact-me.real-me.co.kr` 호스트의 세션/이벤트 rows를 CUT(2026-07-15)로 분리:
  - date < 2026-07-15 → `impact-me` 퍼널
  - date >= 2026-07-15 → `discount-me` 퍼널
- `landing_funnels`에 discount-me 항목을 아래 라벨로 출력.

## 3. 단계 라벨 (discount-me)
| 단계 | 라벨 | 비고 |
|---|---|---|
| stage2 | `신청 버튼 클릭` | GA4 이벤트(기존 impact-me와 동일 이벤트) |
| stage3 | `타입폼 완료` | GA4 이벤트 |
| booking | `예약금 결제` | index.html에서 `lf.id==='discount-me'`일 때 처리(이미 반영됨) |

## 4. ga4_config.json 스니펫 (⚠️ 추정 — 실제 스키마에 맞게 조정 필요)
> 실제 `ga4_config.json` 구조를 열어보고, 기존 impact-me 항목을 복제해 아래 값으로 맞추세요.
> 특히 stage 이벤트 이름은 기존 impact-me 설정 값을 그대로 재사용.
```jsonc
{
  "id": "discount-me",
  "label": "할인퍼널 (impact-me 랜딩 · 7/15~)",
  "hostname": "impact-me.real-me.co.kr",
  "date_from": "2026-07-15",            // 이 호스트에서 7/15부터만 이 퍼널로
  "stage2": { "event": "<impact-me와 동일>", "label": "신청 버튼 클릭" },
  "stage3": { "event": "<impact-me와 동일>", "label": "타입폼 완료" }
}
```
그리고 기존 impact-me 항목에는 `"date_to": "2026-07-14"`(7/14까지)를 걸어 겹침 방지.

## 5. index.html (이미 반영됨 — 재생성 대상 아님, 유지됨)
- 랜딩 퍼널 노출 = **조회기간 내 광고비(landing_metrics.spend)가 있는 퍼널만** 표시(하드코딩 날짜 제거).
  → 파이프라인은 discount-me의 landing_metrics.spend만 정확히 채우면 됨.
- 카드 순서: `find-me`(예약) → `discount-me`(할인) → `change-me` → `impact-me`.
  → `ga4_data.json`의 `landing_funnels` 배열 순서가 곧 표시 순서이므로, 파이프라인이 이 순서로 출력하거나 정렬 유지.
- `bookingLabel`은 discount-me일 때 `예약금 결제`로 이미 분기됨.

## 6. 손수정으로 반영해 둔 데이터(참고 — 재실행하면 GA4 최신치로 대체됨)
- 할인 광고비: 7/15 ₩37,783 + 7/16 ₩119,849 + 7/17 ₩125,076 = **₩282,708** (구글시트 raw Meta 표 기준)
- GA4 유입: 7/15=73, 7/16=19 세션 (7/17 이후는 GA4 미수집 → 재실행 시 채워짐)
- 예약금 결제·매출: 현재 0 (시트에 아직 없음)
