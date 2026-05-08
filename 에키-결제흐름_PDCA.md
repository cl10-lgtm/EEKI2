# 에키-결제흐름 — 핵심기능 명세 (PDCA)

> 기반: Persona1.md (박서연), Persona2.md (최민준), strategy_v1.md
> 범위: IC카드·현금 일회권·환불·트래블월렛·애플페이 전체
> 잔액 부족 대응: 알림 + ATM 지도 + 역무원 카드 자동 연동
> 깊이: 개발자용 (API + 데이터 구조 + 예외처리)
> 참조: 에키-온보딩_PDCA_v2.md (카드 선택·잔액 알림 설정 중복 제외)
> 작성일: 2026-04-24

---

## 퍼소나별 핵심 시나리오

| 구분 | 박서연 (P1 · 도쿄) | 최민준 (P2 · 후쿠오카) |
|------|------------------|----------------------|
| 결제 상황 | Suica 잔액 부족, 신주쿠 개찰구 앞 막힘 | 현금 일회권 매번 구매, ATM 위치 몰라 헤맴 |
| 현재 대응 | 트래블월렛 앱 따로 켜서 잔액 확인 | 창구 줄 서서 현금 결제, 매번 요금 계산 |
| 핵심 고통 | "찍었다가 잔액부족으로 뜸" | "매번 같은 실수 반복" |
| 성공 기준 | 개찰구 막히기 전에 알림 받고 해결 | IC카드 한 번 설정으로 이후 걱정 없음 |

---

## P — Plan (기획 목표 및 설계 원칙)

### 기능 목표

1. **사전 차단**: 잔액 부족이 개찰구에서 터지기 전에 앱이 먼저 알린다
2. **원스톱 해결**: 알림 → ATM 위치 → 역무원 카드까지 앱 이탈 없이 연결
3. **결제수단 안내**: 한국인이 쓰는 카드(트래블월렛·애플페이 Suica)별 차이를 명확히 정리

> U6: *"교통카드 잔액 표시가 안 되어서 그냥 찍었다가 잔액부족으로 뜸"*
> U9: *"구매처와 방법에 대한 설명 부족으로 인한 시간 요소"*
> U5: *"결제 시 일본 앱 설치 및 애플페이 연동 혼란"*
> U4: *"트래블월렛 사용 시 빠진 요금을 일일이 확인하는 점이 불편"*

### 설계 원칙

- **선제 알림** — 잔액 임계값 도달 시 이동 중에도 즉시 푸시
- **3단계 연쇄 해결** — 알림 → ATM 지도 → 역무원 카드 (각 단계에서 탈출 가능)
- **결제수단 중립** — 특정 카드 유도 없이 장단점 객관적 제공
- **오프라인 우선** — ATM 지도는 로컬 캐시 기반, 네트워크 없이도 최근 위치 표시

---

## D — Do (기능 명세)

---

### FEAT-01. 결제수단 비교 허브

**트리거**: 설정 > 교통카드 또는 온보딩 카드 선택 이후 "자세히 보기"
*(온보딩 FEAT-02 카드 선택과 연결 — 중복 명세 제외)*

**결제수단 데이터 스키마**
```json
{
  "payment_methods": [
    {
      "id": "suica",
      "name": "Suica",
      "type": "ic_card",
      "issuer": "JR East",
      "coverage_rating": 5,
      "purchase_difficulty": 1,
      "initial_deposit_jpy": 500,
      "recommended_charge_jpy": 3000,
      "supported_regions": ["전국"],
      "unsupported_lines": [],
      "korean_card_linkable": false,
      "apple_pay_supported": true,
      "google_pay_supported": true,
      "refundable": true,
      "refund_fee_jpy": 220,
      "notes_ko": "가장 범용적. 애플페이 연동 가능"
    },
    {
      "id": "travel_wallet",
      "name": "트래블월렛",
      "type": "prepaid_korean",
      "issuer": "트래블월렛",
      "coverage_rating": 4,
      "purchase_difficulty": 1,
      "initial_deposit_jpy": 0,
      "recommended_charge_jpy": 5000,
      "supported_regions": ["도쿄", "오사카", "후쿠오카", "대부분"],
      "unsupported_lines": ["일부 사철"],
      "korean_card_linkable": true,
      "apple_pay_supported": false,
      "google_pay_supported": false,
      "refundable": true,
      "refund_fee_jpy": 0,
      "notes_ko": "한국 카드로 충전 가능. 일부 노선 불가"
    },
    {
      "id": "apple_pay_suica",
      "name": "애플페이 Suica",
      "type": "digital_ic",
      "issuer": "Apple / JR East",
      "coverage_rating": 5,
      "purchase_difficulty": 3,
      "initial_deposit_jpy": 0,
      "recommended_charge_jpy": 3000,
      "supported_regions": ["전국"],
      "unsupported_lines": [],
      "korean_card_linkable": false,
      "apple_pay_supported": true,
      "google_pay_supported": false,
      "refundable": false,
      "notes_ko": "iPhone 전용. 한국 카드 연동 불가 (해외 카드 불가)"
    },
    {
      "id": "cash_ticket",
      "name": "현금 일회권",
      "type": "single_use",
      "issuer": null,
      "coverage_rating": 5,
      "purchase_difficulty": 3,
      "initial_deposit_jpy": 0,
      "recommended_charge_jpy": null,
      "supported_regions": ["전국"],
      "unsupported_lines": [],
      "korean_card_linkable": false,
      "apple_pay_supported": false,
      "google_pay_supported": false,
      "refundable": false,
      "notes_ko": "노선별 요금 직접 계산 필요. ATM 창구 줄 필요"
    }
  ]
}
```

**비교 화면 UI**
```
┌──────────────────────────────────────┐
│  어떤 결제수단이 나에게 맞을까요?     │
│                                      │
│  [Suica] [트래블월렛] [애플페이] [현금]│  ← 탭 전환
│  ──────────────────────────────────  │
│  Suica                               │
│  ✅ 전국 거의 모든 노선              │
│  ✅ 애플페이 연동 가능               │
│  ✅ 환불 가능 (수수료 ¥220)          │
│  ❌ 한국 카드 직접 충전 불가         │
│  ──────────────────────────────────  │
│  초기 보증금: ¥500                   │
│  권장 충전액: ¥3,000                 │
│  구매 장소: 역내 자동발매기           │
└──────────────────────────────────────┘
```

---

### FEAT-02. 잔액 실시간 감지 및 알림

**트리거**: 앱 백그라운드 실행 중 NFC 잔액 읽기 또는 수동 입력값 기준

**잔액 감지 방식**

```
방식 A — NFC 자동 읽기 (권장)
  Android: NFC Adapter + FeliCa Tag (Type F)
  iOS:     CoreNFC (iOS 13+), NDEF polling

  읽기 시점:
    1. 경로 검색 완료 후 자동 1회
    2. 앱 포그라운드 진입 시
    3. 유저 수동 "잔액 확인" 버튼 탭

방식 B — 수동 입력 폴백 (NFC 미지원 기기)
  유저가 직접 현재 잔액 입력 → 경로 요금 기반 시뮬레이션
  예) 현재 ¥800, 이번 경로 요금 ¥210 → 예상 잔여 ¥590
```

**잔액 상태 스키마**
```json
{
  "balance": {
    "card_type": "suica",
    "amount_jpy": 380,
    "last_updated": "2026-04-24T14:32:00Z",
    "source": "nfc | manual",
    "alert_threshold_jpy": 1000,
    "status": "warning"
  }
}
```

**알림 발송 로직**
```
IF balance.amount_jpy <= alert_threshold_jpy:
  → 푸시 알림 즉시 발송
  → 앱 내 상단 배너 노출 (포그라운드 시)
  → 경로 결과 화면에 잔액 경고 배지 추가

IF balance.amount_jpy < 예상_이번경로_요금:
  → "이번 경로 탑승 불가" 긴급 알림
  → FEAT-03 (ATM 지도) 자동 팝업

푸시 알림 포맷:
  제목: "💳 Suica 잔액 부족 임박"
  내용: "현재 ¥380 — 권장 충전 금액 ¥3,000"
  액션: [ATM 찾기] [나중에]
```

---

### FEAT-03. 충전 ATM 지도

**트리거**:
- FEAT-02 알림에서 "ATM 찾기" 탭
- 잔액 부족으로 경로 탑승 불가 감지 시 자동 팝업
- 대화카드 "잔액 부족" 선택 후 하단 딥링크

**ATM 데이터 스키마**
```json
{
  "atm_id": "shinjuku_south_01",
  "station_id": "shinjuku",
  "location_ko": "남쪽 개찰구 옆 기둥",
  "location_ja": "南改札口 横",
  "supported_cards": ["suica", "pasmo", "icoca"],
  "cash_only": true,
  "operating_hours": "05:30-24:00",
  "coordinates": { "lat": 35.6896, "lng": 139.7006 },
  "last_verified": "2026-04"
}
```

**API 명세**
```
GET /api/v1/atm/nearby
Request:
  {
    "station_id": "shinjuku",    // 현재 역 (GPS 또는 경로 기반)
    "card_type": "suica",
    "lat": 35.6896,
    "lng": 139.7006
  }

Response:
  {
    "atms": [
      {
        "atm_id": "shinjuku_south_01",
        "distance_m": 45,
        "location_ko": "남쪽 개찰구 옆",
        "walk_seconds": 30,
        "operating_now": true
      }
    ]
  }

오프라인 시: 로컬 캐시 ATM 목록 반환 (last_verified 날짜 표시)
```

**화면 구조**
```
┌──────────────────────────────────────┐
│  💳 Suica 충전 ATM                   │
│  현재 잔액: ¥380   임계값: ¥1,000    │
│  ──────────────────────────────────  │
│  📍 가장 가까운 ATM                  │
│                                      │
│  남쪽 개찰구 옆 기둥    도보 30초    │
│  ● 현재 운영 중                      │
│                                      │
│  중앙 개찰구 앞         도보 1분 20초│
│  ● 현재 운영 중                      │
│  ──────────────────────────────────  │
│  💡 기계에서 "チャージ" 버튼을 누르세요│
│                                      │
│  [역무원에게 도움 요청하기]           │  ← 역무원 카드 연동
└──────────────────────────────────────┘
```

**역무원 카드 연동**
```
"역무원에게 도움 요청하기" 탭 시:
  → 에키-역무원카드 FEAT-02로 진입
  → 카테고리: 개찰구 문제 자동 선택
  → 카드: "잔액 부족으로 개찰구가 열리지 않아요" 자동 선택
  → 카드 화면 즉시 표시 (2탭 단축)
```

---

### FEAT-04. 현금 일회권 구매 가이드

**트리거**: FEAT-01 비교 화면에서 "현금 일회권" 선택 또는 경로 결과 화면 "현금으로 탈게요"

**목적**: 매번 창구 줄 서는 최민준의 반복 고통 해결
> U3: *"현금 충전이 불편"*
> 최민준 행동 패턴: *"매번 현금으로 일회권 구매 — 창구마다 줄, 일일이 요금 계산 번거로움"*

**요금 자동 계산 + 발매기 안내**
```
경로 결과 연동:
  출발: 하카타역  도착: 텐진역
  ↓
  일회권 요금: ¥210
  ↓
  ┌──────────────────────────────────────┐
  │  현금 일회권 구매 방법               │
  │                                      │
  │  1. 발매기에서 금액 선택             │
  │     → ¥210 버튼 또는 "텐진" 탭      │
  │                                      │
  │  2. 현금 투입 후 표 수령             │
  │                                      │
  │  📍 발매기 위치: 개찰구 앞 (6대)    │
  │  ──────────────────────────────────  │
  │  💡 IC카드 설정하면 이 과정 생략돼요 │
  │  [ Suica 설정하기 ]   [ 괜찮아요 ]  │
  └──────────────────────────────────────┘
```

**요금 조회 API**
```
GET /api/v1/fare
Request:
  {
    "origin_station_id": "hakata",
    "dest_station_id": "tenjin",
    "ticket_type": "cash_single"
  }

Response:
  {
    "fare_jpy": 210,
    "child_fare_jpy": 110,
    "ic_fare_jpy": 207,
    "ic_discount_jpy": 3
  }
```

---

### FEAT-05. 환불 가이드

**트리거**: 설정 > 교통카드 > "환불하기" 또는 여행 종료 후 홈 배너

**환불 가능 카드별 처리**

```
Suica / Pasmo:
  환불 수수료: ¥220
  조건: 잔액 ¥220 이상
  장소: 역 창구 (みどりの窓口 등)
  필요서류: 카드 본체
  환불 예상액: 잔액 - ¥220 + 보증금 ¥500

트래블월렛:
  환불 수수료: 없음
  방법: 앱 내 원화 환급
  소요 시간: 영업일 3~5일

애플페이 Suica:
  환불 불가 (잔액 이월만 가능)
  → 다음 일본 방문 시 사용 안내
```

**환불 화면 + 역무원 카드 연동**
```
┌──────────────────────────────────────┐
│  Suica 환불                          │
│  현재 잔액: ¥1,240                   │
│  예상 환급액: ¥1,520  (잔액+보증금-수수료)│
│  ──────────────────────────────────  │
│  환불 장소: みどりの窓口 (JR 창구)   │
│  소요 시간: 약 5분                   │
│  ──────────────────────────────────  │
│  창구에서 이렇게 말해보세요:          │
│  [ 환불 대화 카드 열기 ]             │  ← 역무원카드 연동
│                                      │
│  [환불 완료했어요]  (잔액 초기화)    │
└──────────────────────────────────────┘
```

---

## C — Check (검증 지표)

### 정량 지표

| 지표 | 목표값 | 측정 이벤트 |
|------|--------|------------|
| 잔액 알림 → ATM 화면 전환율 | ≥ 55% | `balance_alert_tapped` → `atm_map_shown` |
| ATM 화면 → 역무원 카드 전환율 | ≥ 20% | `atm_staff_card_tapped` |
| 개찰구 막힘 후 해결 완료율 | ≥ 70% | `balance_critical` → `balance_recharged` |
| NFC 잔액 읽기 성공률 | ≥ 65% (NFC 지원 기기) | `nfc_read_success` |
| 현금 일회권 → IC카드 전환 유도 클릭률 | ≥ 10% | `cash_to_ic_cta_tapped` |
| ATM 오프라인 캐시 사용 비율 | 측정 후 캐시 신선도 기준 수립 | `atm_cache_used` |

### 정성 지표

- "잔액 알림을 받고 실제로 미리 충전했나요?"
- "ATM을 찾는 데 앱이 도움이 됐나요? 실제 위치와 맞았나요?"
- "현금에서 IC카드로 바꾼 이유가 앱 때문이었나요?"

---

## A — Act (개선 트리거 및 로드맵)

### 즉시 개선 트리거

| 상황 | 기준 | 액션 |
|------|------|------|
| 알림 → ATM 전환율 저조 | < 30% | 알림 문구 긴급도 강화, 딥링크 직접 연결 |
| NFC 읽기 실패 급증 | 실패율 > 40% | 수동 입력 기본 UI로 전환, NFC 보조 안내 개선 |
| ATM 위치 정보 불일치 | 유저 오류 리포트 > 5건/주 | 해당 역 ATM 데이터 즉시 검토·수정 |
| 현금→IC 전환 유도 무시 | 클릭률 < 3% | CTA 문구·위치 변경 또는 타이밍 조정 |

### 다음 단계 로드맵

```
MVP (v1.0)
  ├── FEAT-02  잔액 알림 (수동 입력 기반)
  ├── FEAT-03  ATM 지도 (주요 5개 역 로컬 캐시)
  └── FEAT-04  현금 일회권 요금 안내 + 발매기 위치

v1.1
  ├── FEAT-01  결제수단 비교 허브
  ├── FEAT-02  NFC 자동 잔액 읽기 (Android 선행, iOS 후속)
  └── FEAT-03  역무원 카드 자동 연동

v1.2
  └── FEAT-05  환불 가이드 + 역무원 카드 연동
```

---

## 미해결 리스크

| 항목 | 내용 |
|------|------|
| NFC FeliCa 읽기 권한 | iOS CoreNFC는 NDEF만 공식 지원 — FeliCa(Suica) 읽기는 iOS 13+에서 제한적 허용, 심사 리젝 가능성 검토 필요 |
| ATM 데이터 정확도 | 역사 내 ATM 위치·운영시간 변경 잦음 → 유저 오류 리포트 채널 + 월 1회 검증 프로세스 필수 |
| 트래블월렛 API | 잔액 조회 API 비공개 시 수동 입력만 지원 → 트래블월렛 측 제휴 협의 필요 |
| 애플페이 Suica 한국 카드 | 애플페이 Suica는 해외 발급 카드 연동 불가 — 명확히 안내하지 않으면 유저 혼란 |
| 요금 데이터 갱신 | 일본 지하철 요금은 연 1~2회 개정 → 서버 요금 DB 갱신 주기 및 알림 체계 사전 설계 필요 |
