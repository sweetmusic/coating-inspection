# HTML 점검보고서 데이터 형식 (coating-inspection/1)

앱의 **🖨️ 보고서 → 📤 HTML** 로 받은 파일에는, 사람이 보는 보고서와 별도로
**사내 웹앱이 읽기 위한 데이터 블록**이 들어 있습니다. 화면·인쇄에는 보이지 않습니다.

- 샘플 파일: [`sample-report.html`](sample-report.html) (가상의 협력사로 앱에서 실제 생성)
- 이 파일은 PC 크롬에서 열어 노란 칸(비고·조치 계획·사진 설명·Remark)을 고친 뒤
  **💾 수정본 저장**으로 다시 저장할 수 있습니다. 이때 **데이터 블록도 함께 갱신**되고 `edited_at`이 기록됩니다.
- 협력사에 보내는 파일이므로 **등급(A~D)은 넣지 않습니다.** 필요하면 `summary.normalized_score`와 설정 파일의 `grades`로 계산합니다.

## 1. 파일 안에서의 위치

```html
<head>
  <meta name="generator" content="coating-inspection">
  <meta name="inspection-data-sha256" content="(64자리 16진수)">
  <script type="application/json" id="inspection-data">{ ...JSON... }</script>
</head>
...
<img src="data:image/jpeg;base64,..." alt="현장사진 1" data-photo-id="P01">
```

| 요소 | 용도 |
|---|---|
| `script#inspection-data` | 점검 데이터(JSON). `<` 문자는 `<`로 저장되므로 블록이 중간에 끊기지 않으며, 일반 JSON 파서로 그대로 읽으면 원래 문자로 복원됩니다. |
| `meta[name=inspection-data-sha256]` | 데이터 블록 **텍스트 그대로(UTF-8)** 의 SHA-256. 앱이나 [수정본 저장] 외의 방법(메모장 등)으로 고쳤는지 확인하는 용도입니다(보안 서명은 아님). 비어 있으면 생성 환경에서 계산할 수 없었던 것입니다. |
| `img[data-photo-id]` | 사진 원본(JPEG). `data-photo-id`가 JSON의 `photos[].id`와 같습니다. |

> 업로드된 HTML은 브라우저로 열지 말고 **텍스트로 읽어서** 필요한 부분만 꺼내 주세요.

## 2. JSON 구조

```jsonc
{
  "schema": "coating-inspection/1",       // 데이터 형식 버전. 구조가 바뀌면 /2 로 올림
  "inspection_uid": "9bdfbc42-…",         // 점검 1건의 고유 ID. 같은 점검의 보고서를 다시 받거나 수정본을 저장해도 동일 → 중복 업로드 판별용
  "generated_at": "2026-10-06T13:31:40+09:00",  // 앱에서 보고서를 만든 시각
  "edited_at": null,                      // 내보낸 파일에서 [수정본 저장]한 시각 (없으면 null). 같은 uid면 최신 것을 사용
  "app_build": "b13",                     // 앱 빌드 번호
  "checklist": { "version": "v1.0", "updated_at": "2026-03-29 17:55:48" },  // 사용한 체크시트 버전
  "inspection": {
    "company": { "id": "C001", "name": "삼녹㈜" },  // 협력사는 id 기준으로 관리 권장
    "date": "2026-10-06",
    "type": "정기",                          // "정기" | "상하반기"
    "inspector": "홍길동",
    "reply_due": "2026-10-13",               // 조치 회신 기한
    "remark": "종합 Remark (줄바꿈 포함)"
  },
  "summary": {
    "raw_score": 76, "max_score": 105, "normalized_score": 72,  // 원점수 / 만점 / 환산점수(100점)
    "answered": 50, "unanswered": 0, "excluded": 1,             // 항목 상태별 개수
    "findings": 16,                                             // 지적사항 수
    "photos": 2
  },
  "majors": [                               // 대분류별 점수
    { "id": "2", "name": "품질문서관리", "score": 19, "max_score": 28, "pct": 68 }
  ],
  "answers": [                              // 세부항목별 결과 (사용 중인 항목 전체)
    {
      "id": "2-1-2",
      "major_id": "2", "major_name": "품질문서관리",
      "item_id": "2-1", "item_name": "문서(도면, Spec, 절차서) 개정/폐기 관리 현황",
      "status": "answered",                 // "answered" | "unanswered" | "excluded"
      "selected_index": 2,                  // 선택한 선택지 순번(0부터). 미선택이면 null
      "selected_text": "문서 최신본 미보유",
      "score": 0,                           // 미선택·제외 항목은 null (합계에서는 0점 처리)
      "max_score": 3,
      "finding": true,                      // 만점 미달 = 지적사항
      "memo": "점검자 코멘트 · 조치 계획",
      "options": [                          // 작성 당시의 선택지 전체 (체크시트가 바뀌어도 재현 가능)
        { "text": "문서 최신본 보유 및 관리 양호", "score": 3 },
        { "text": "최신본 사용중이나 이전 문서 미폐기", "score": 1 },
        { "text": "문서 최신본 미보유", "score": 0 }
      ]
    }
  ],
  "photos": [
    {
      "id": "P01",                          // img[data-photo-id]와 연결
      "no": 1,                              // 보고서에 표시되는 "사진 1"
      "file": "IMG_1234.jpg",               // 촬영 당시 파일명
      "mime": "image/jpeg",
      "width": 1200, "height": 900,          // 앱에서 압축된 크기 (긴 변 최대 1200px)
      "bytes": 25231,                       // 디코딩한 JPEG 크기
      "sha256": "…",                        // 디코딩한 JPEG 바이트의 SHA-256
      "added_at": "2026-10-06T13:31:40+09:00",  // 앱에 추가한 시각 (이전 버전에서 추가한 사진은 null)
      "caption": "도장 전 오일 미제거 상태",   // 사진 설명
      "linked_to": "2-1-2"                  // 연결한 점검항목 id (없으면 null)
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
- 점수는 앱에서만 바뀝니다. 내보낸 파일에서 고칠 수 있는 것은 `memo`, `photos[].caption`, `inspection.remark` 뿐입니다.

## 3. 읽기 예시 (Python)

```python
import re, json, hashlib, base64

html = open("보고서.html", encoding="utf-8").read()

raw = re.search(r'<script type="application/json" id="inspection-data">(.*?)</script>', html, re.S).group(1)
data = json.loads(raw)

# 1) 수정 여부 확인
expected = re.search(r'name="inspection-data-sha256" content="([0-9a-f]*)"', html).group(1)
if expected and hashlib.sha256(raw.encode("utf-8")).hexdigest() != expected:
    raise ValueError("데이터 블록이 앱 밖에서 수정되었습니다")

# 2) 사진 추출 → 파일 저장
for ph in data["photos"]:
    b64 = re.search(r'<img src="data:image/jpeg;base64,([^"]+)"[^>]*data-photo-id="%s"' % ph["id"], html).group(1)
    jpg = base64.b64decode(b64)
    assert hashlib.sha256(jpg).hexdigest() == ph["sha256"]
    name = f'{data["inspection"]["company"]["id"]}_{data["inspection"]["date"]}_{ph["id"]}.jpg'
    open(name, "wb").write(jpg)
```

## 4. 버전 호환

- `schema`가 `coating-inspection/1`이 아니면 처리하지 말고 알림을 띄워 주세요.
- 필드가 **추가**되는 것은 같은 schema 버전 안에서 일어날 수 있습니다. 모르는 필드는 무시하도록 만들어 주세요.
- 필드의 의미가 바뀌거나 삭제될 때만 schema 버전을 올립니다.
