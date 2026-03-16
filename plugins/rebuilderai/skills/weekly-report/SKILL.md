---
name: weekly-report
description: >
  RebuilderAI 전사 업무표(월) 자동 업데이트 스킬. 매주 초(월요일) 주간 업무 보고서 작성 시 반드시 사용할 것.
  Daily DB와 Notes DB를 분석해 지난주 달성 현황과 이번 주 업무 계획을 자동으로 작성하고
  전사 업무표의 개인 항목(김하영)을 업데이트한다.
  "주간 업무표", "업무 보고", "전사 업무표", "주간 업데이트", "주간 계획", "지난주 정리" 등의 표현이 있을 때 반드시 이 스킬을 사용한다.
---

# RebuilderAI 주간 업무표 자동 업데이트 스킬

## 개요

매주 초(주로 월요일) 실행하는 작업으로, Notion의 Daily DB와 Notes DB를 파싱하여 아래 전사 업무표에 자동으로 업데이트한다.

## 핵심 ID / URL 참조

| 항목 | ID / URL |
|------|----------|
| 내 홈 페이지 | `31119aac-27e4-80ad-96d0-ffd3f2cdd3ef` |
| Daily DB (data source) | `collection://31119aac-27e4-8008-bb4b-000bd838f764` |
| Notes DB (data source) | `collection://31119aac-27e4-80d9-a83c-000be57af066` |
| 전사 업무표(월) 페이지 | `31819aac-27e4-8016-9d2f-df6ab93f06cd` |
| 개인별 일정 업데이트 DB | `collection://31819aac-27e4-815f-8599-000ba2b76236` |
| 내 업무표 행 (김하영) | `31819aac-27e4-81ca-ae5c-c978b6cfabec` |

## 실행 절차

### Step 1: 날짜 범위 계산

오늘 날짜를 기준으로:
- **지난주**: 직전 월요일 ~ 금요일 (예: 오늘이 3/9이면 3/2~3/6)
- **이번주**: 오늘 ~ 이번 주 금요일 (예: 3/9~3/13)

### Step 2: Daily DB 쿼리

```sql
-- {지난주_월요일}은 Step 1에서 계산한 날짜 값으로 치환
SELECT url, "Name", "date:Date:start"
FROM "collection://31119aac-27e4-8008-bb4b-000bd838f764"
WHERE "date:Date:start" >= '{지난주_월요일}'
ORDER BY "date:Date:start" DESC
LIMIT 15
```

반환된 각 페이지 URL을 `notion-fetch`로 개별 조회한다.
각 노트에서 아래를 추출:

**Daily DB 포맷 (SSoT 구조):**
Daily 페이지는 `## todo` 섹션 아래에 이니셔티브별로 구조화되어 있다:
```
## todo
* [이니셔티브명]
	- [x] 완료 항목 (1탭 들여쓰기 = 하위 요소)
	- [ ] 미완료 항목
### misc.
- 이니셔티브에 속하지 않는 기타 업무
```

`* [이니셔티브명]`은 최상위 bullet이고, 그 아래 체크박스들은 탭 들여쓰기된 하위 요소이다.
`* [이니셔티브명]` 헤더별로 완료(`[x]`) / 미완료(`[ ]`) 항목을 분류한다.
레거시 포맷(`### todo`, `### misc.` 섹션)도 하위 호환으로 지원한다.

### Step 3: Notes DB 쿼리

```sql
-- {지난주_월요일}은 Step 1에서 계산한 날짜 값으로 치환
SELECT url, "Name", createdTime
FROM "collection://31119aac-27e4-80d9-a83c-000be57af066"
WHERE createdTime >= '{지난주_월요일}T00:00:00Z'
ORDER BY createdTime DESC
LIMIT 20
```

Notes는 내가 직접 생성한 학습/기록 문서들로, 어떤 주제를 연구했는지 파악하는 데 활용한다.

### Step 4: 이번 주 계획 파악

이번 주 Monday Daily 노트를 `notion-fetch`로 조회한다 (아직 빈 경우가 많음).
추가로 홈 페이지의 `work on progress` 섹션과 `Reminder` callout을 확인한다:

```
Home 페이지 ID: 31119aac-27e4-80ad-96d0-ffd3f2cdd3ef
```

`work on progress`에 나열된 항목이 이번 주 주요 계획이다.

### Step 5: 업무표 컬럼 매핑

수집한 데이터를 아래 컬럼에 맞게 정리한다:

| 컬럼명 | 내용 |
|--------|------|
| `지난주 업무 계획` | 지난주 Daily todo 기반, 원래 계획했던 것 |
| `지난주 달성 현황` | 실제로 완료(`[x]`)된 항목들, Notes 생성 내역 포함 |
| `달성 여부 (미달성 사유)` | 미완료(`[ ]`) 항목이 있으면 사유 기재, 없으면 "전체 달성" |
| `주간 업무 계획(구체적으로 작성)` | work on progress + 이번 주 Daily/Reminder 기반 |
| `정량적 목표/달성율` | 예: "LDM 학습 완료 / 추론 샘플 5개 이상" |

### Step 6: 초안 작성 및 검토 요청

작성된 내용을 대화에서 **마크다운 표 형태**로 하영에게 먼저 보여준다.
수정 요청이 없으면 업데이트 진행 여부를 확인한다.

### Step 7: Notion 업데이트

확인 후 `notion-update-page`로 업데이트:

```
page_id: 31819aac-27e4-81ca-ae5c-c978b6cfabec
command: update_properties
```

---

## 주의사항

- **Notion MCP 연결이 필요하다.** 연결이 안 되어 있으면 사용자에게 안내한다.
- **하영이 검토한 후에만** 실제 Notion 업데이트를 진행한다.
- 미완료 항목은 삭제하지 않고 "진행 중" 또는 미달성 사유로 표기한다.
- 학습 중인 모델 (LDM, GarmageJigsaw 등)은 완료가 아닌 "진행 중"으로 표기한다.
- 컬럼명은 한국어 그대로 정확히 사용한다 (괄호 포함).
- 전사 업무표는 월요일 전사 미팅 전에 업데이트 완료되어야 한다.
