# HTML 점검보고서 데이터 형식 (coating-inspection/1)

모바일 앱의 **📄 보고서** 버튼으로 받은 HTML 파일에는, 사람이 보는 보고서와 별도로
**사내 웹앱이 읽기 위한 데이터 블록**이 들어 있습니다. 화면·인쇄에는 보이지 않습니다.

- 샘플 파일: [`sample-report.html`](sample-report.html) (가상의 협력사로 앱에서 실제 생성)

## 1. 파일 안에서의 위치

```html
<head>
  <meta name="generator" content="coating-inspection">
  <meta name="inspection-data-sha256" content="(64자리 16진수)">
  <script type="application/json" id="inspection-data">{ ...JSON... }</script>
</head>
...
<img src="data:image/jpeg;base64,..." data-photo-id="P01">
```

| 요소 | 용도 |
|---|---|
| `script#inspection-data` | 점검 데이터(JSON). `<` 문자는 `<`로 저장되므로 블록이 중간에 끊기지 않으며, 일반 JSON 파서로 그대로 읽으면 원래 문자로 복원됩니다. |
| `meta[name=inspection-data-sha256]` | 데이터 블록 **텍스트 그대로(UTF-8)** 의 SHA-256. 파일이 수정되었는지 확인하는 용도입니다(보안 서명은 아님). 비어 있으면 생성 환경에서 계산할 수 없었던 것입니다. |
| `img[data-photo-id]` | 사진 원본(JPEG). `data-photo-id`가 JSON의 `photos[].id`와 같습니다. |

> 업로드된 HTML은 브라우저로 열지 말고 **텍스트로 읽어서** 필요한 부분만 꺼내 주세요.

## 2. JSON 구조

```jsonc
{
  "schema": "coating-inspection/1",       // 데이터 형식 버전. 구조가 바뀌면 /2 로 올림
  "inspection_uid": "9bdfbc42-…",         // 점검 1건의 고유 ID. 같은 점검의 보고서를 다시 받아도 동일 → 중복 업로드 판별용
  "generated_at": "2026-10-06T13:31:40+09:00",  // 보고서 생성 시각
  "checklist": { "version": "v1.0", "updated_at": "2026-03-29 17:55:48" },  // 사용한 체크시트 버전
  "inspection": {
    "company": { "id": "C001", "name": "삼녹㈜" },  // 협력사는 id 기준으로 관리 권장
    "date": "2026-10-06",
    "type": "정기",                          // "정기" | "상하반기"
    "inspector": "홍길동",
    "remark": "종합 Remark"
  },
  "summary": {
    "raw_score": 51, "max_score": 102, "normalized_score": 50,   // 원점수 / 만점 / 환산점수(100점)
    "grade": "D", "grade_label": "시정조치실시",                  // 등급 (산출 불가 시 null)
    "answered": 30, "unanswered": 19, "excluded": 1,             // 항목 상태별 개수
    "findings": 7,                                               // 지적사항 수
    "photos": 2
  },
  "majors": [                               // 대분류별 점수
    { "id": "2", "name": "품질문서관리", "score": 20, "max_score": 25, "pct": 80 }
  ],
  "answers": [                              // 세부항목별 결과 (사용 중인 항목 전체)
    {
      "id": "2-3-1",
      "major_id": "2", "major_name": "품질문서관리",
      "item_id": "2-3", "item_name": "작업자 교육 및 유지 관리 / …",
      "status": "answered",                 // "answered" | "unanswered" | "excluded"
      "selected_index": 1,                  // 선택한 선택지 순번(0부터). 미선택이면 null
      "selected_text": "도장 교육 연간 계획서 미작성",
      "score": 0,                           // 미선택·제외 항목은 null (합계에서는 0점 처리)
      "max_score": 2,
      "finding": true,                      // 만점 미달 = 지적사항
      "memo": "비고",
      "options": [                          // 작성 당시의 선택지 전체 (체크시트가 바뀌어도 재현 가능)
        { "text": "도장 교육 연간 계획서 작성", "score": 2 },
        { "text": "도장 교육 연간 계획서 미작성", "score": 0 }
      ]
    }
  ],
  "photos": [
    {
      "id": "P01",                          // img[data-photo-id]와 연결
      "file": "IMG_1234.jpg",               // 촬영 당시 파일명
      "mime": "image/jpeg",
      "width": 1200, "height": 900,          // 앱에서 압축된 크기 (긴 변 최대 1200px)
      "bytes": 25231,                       // 디코딩한 JPEG 크기
      "sha256": "…",                        // 디코딩한 JPEG 바이트의 SHA-256
      "added_at": "2026-10-06T13:31:40+09:00",  // 앱에 추가한 시각 (이전 버전에서 추가한 사진은 null)
      "linked_to": null                     // 예약: 향후 지적사항과 사진 연결용
    }
  ]
}
```

### 계산 규칙
- `summary.raw_score` = `status`가 `excluded`가 아닌 항목들의 `score` 합 (null은 0)
- `summary.max_score` = 같은 항목들의 `max_score` 합
- `normalized_score` = `round(raw_score / max_score × 100)`
- `excluded`: 정기점검에서 1-1-1 항목은 점수에서 제외
- 체크시트에서 사용 중지(`"active": false`)된 항목은 `answers`에 들어가지 않음

## 3. 읽기 예시 (Python)

```python
import re, json, hashlib, base64

html = open("보고서.html", encoding="utf-8").read()

raw = re.search(r'<script type="application/json" id="inspection-data">(.*?)</script>', html, re.S).group(1)
data = json.loads(raw)

# 1) 수정 여부 확인
expected = re.search(r'name="inspection-data-sha256" content="([0-9a-f]*)"', html).group(1)
if expected and hashlib.sha256(raw.encode("utf-8")).hexdigest() != expected:
    raise ValueError("데이터 블록이 수정되었습니다")

# 2) 사진 추출 → 파일 저장
for ph in data["photos"]:
    b64 = re.search(r'<img src="data:image/jpeg;base64,([^"]+)" data-photo-id="%s"' % ph["id"], html).group(1)
    jpg = base64.b64decode(b64)
    assert hashlib.sha256(jpg).hexdigest() == ph["sha256"]
    name = f'{data["inspection"]["company"]["id"]}_{data["inspection"]["date"]}_{ph["id"]}.jpg'
    open(name, "wb").write(jpg)
```

## 4. 버전 호환

- `schema`가 `coating-inspection/1`이 아니면 처리하지 말고 알림을 띄워 주세요.
- 필드가 **추가**되는 것은 같은 schema 버전 안에서 일어날 수 있습니다. 모르는 필드는 무시하도록 만들어 주세요.
- 필드의 의미가 바뀌거나 삭제될 때만 schema 버전을 올립니다.
