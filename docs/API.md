# 방명록 CRUD API 문서

방명록의 모든 데이터 조작은 Supabase RPC(원격 함수 호출) API로 이루어집니다.
테이블에 직접 접근하는 것은 차단되어 있으며, 아래 7개 함수만 호출할 수 있습니다.

> 최종 수정: 2026-08-08

---

## 기본 정보

| 항목 | 값 |
|---|---|
| Base URL | `https://jcspkkaszsqlwnokqpmz.supabase.co/rest/v1/rpc` |
| HTTP 메서드 | 모든 요청 `POST` |
| 인증 | 공개 키(anon key) — 로그인 불필요 |
| 요청 본문 | JSON |
| 응답 본문 | JSON |

### 공통 요청 헤더

```
Content-Type: application/json
apikey: sb_publishable_4mUadSjAbg1N41XjfH1ZjA_Rdcru2gq
Authorization: Bearer sb_publishable_4mUadSjAbg1N41XjfH1ZjA_Rdcru2gq
```

### 공통 응답 형식

조회(`guestbook_list`)를 제외한 모든 API는 아래 형식으로 응답합니다.

```jsonc
// 성공
{ "ok": true }

// 실패 (비밀번호 불일치, 검증 실패 등)
{ "ok": false, "error": "비밀번호가 일치하지 않아요." }
```

HTTP 상태 코드는 성공/실패 모두 `200`입니다. 반드시 응답 본문의 `ok` 값으로 판단하세요.
(`404`, `400` 등이 반환되는 경우는 함수명 오타, 파라미터 누락 등 요청 자체가 잘못된 경우입니다.)

### 입력값 제한

| 항목 | 제한 |
|---|---|
| 이름/별명 (`p_name`) | 1~20자 |
| 방명록 내용 (`p_content`) | 1~1,000자 |
| 답글 내용 (`p_content`) | 1~500자 |
| 비밀번호 (`p_password`) | 4자 이상 |

---

## 1. 목록 조회 — `guestbook_list`

방명록 전체 목록을 답글 포함으로 반환합니다. 글은 최신순, 답글은 오래된 순으로 정렬됩니다.

```
POST /rest/v1/rpc/guestbook_list
```

**요청 본문**: `{}` (파라미터 없음)

**응답 예시**

```json
[
  {
    "id": "3f2b1c9a-...",
    "name": "주희",
    "content": "안녕하세요!",
    "created_at": "2026-08-08T00:07:43.123456+00:00",
    "updated_at": null,
    "replies": [
      {
        "id": "8e4d2f1b-...",
        "name": "방문자A",
        "content": "반가워요!",
        "created_at": "2026-08-08T00:08:03.123456+00:00",
        "updated_at": null
      }
    ]
  }
]
```

- `updated_at`이 `null`이 아니면 수정된 글입니다.
- `password_hash`는 응답에 포함되지 않습니다.

**curl 예시**

```bash
curl -X POST "https://jcspkkaszsqlwnokqpmz.supabase.co/rest/v1/rpc/guestbook_list" \
  -H "Content-Type: application/json" \
  -H "apikey: sb_publishable_4mUadSjAbg1N41XjfH1ZjA_Rdcru2gq" \
  -H "Authorization: Bearer sb_publishable_4mUadSjAbg1N41XjfH1ZjA_Rdcru2gq" \
  -d '{}'
```

---

## 2. 방명록 작성 — `guestbook_add_entry`

```
POST /rest/v1/rpc/guestbook_add_entry
```

**요청 본문**

| 파라미터 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `p_name` | string | ✅ | 이름 또는 별명 (1~20자) |
| `p_content` | string | ✅ | 내용 (1~1,000자) |
| `p_password` | string | ✅ | 비밀번호 (4자 이상) — 수정·삭제 시 필요 |

```json
{
  "p_name": "주희",
  "p_content": "안녕하세요!",
  "p_password": "my-secret"
}
```

**응답**: `{ "ok": true }`

**에러 케이스**

| 상황 | `error` 메시지 |
|---|---|
| 비밀번호 4자 미만 | `비밀번호는 4자 이상이어야 해요.` |
| 이름/내용 길이 초과·공백 | `이름(1~20자)과 내용(1~1000자)을 확인해 주세요.` |

---

## 3. 방명록 수정 — `guestbook_update_entry`

작성 시 입력한 비밀번호가 일치해야 수정됩니다. 내용만 수정 가능하며, 이름은 변경할 수 없습니다.

```
POST /rest/v1/rpc/guestbook_update_entry
```

**요청 본문**

| 파라미터 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `p_id` | uuid | ✅ | 수정할 글의 id (`guestbook_list`로 조회) |
| `p_password` | string | ✅ | 작성 시 입력한 비밀번호 |
| `p_content` | string | ✅ | 새 내용 (1~1,000자) |

```json
{
  "p_id": "3f2b1c9a-...",
  "p_password": "my-secret",
  "p_content": "수정된 내용입니다."
}
```

**응답**: `{ "ok": true }` — 성공 시 `updated_at`이 현재 시각으로 기록됩니다.

**에러 케이스**

| 상황 | `error` 메시지 |
|---|---|
| 존재하지 않는 id | `글을 찾을 수 없어요.` |
| 비밀번호 불일치 | `비밀번호가 일치하지 않아요.` |
| 내용 길이 초과·공백 | `내용은 1~1000자여야 해요.` |

---

## 4. 방명록 삭제 — `guestbook_delete_entry`

⚠️ 글을 삭제하면 그 글에 달린 **답글도 모두 함께 삭제**됩니다 (cascade). 복구할 수 없습니다.

```
POST /rest/v1/rpc/guestbook_delete_entry
```

**요청 본문**

| 파라미터 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `p_id` | uuid | ✅ | 삭제할 글의 id |
| `p_password` | string | ✅ | 작성 시 입력한 비밀번호 |

```json
{
  "p_id": "3f2b1c9a-...",
  "p_password": "my-secret"
}
```

**응답**: `{ "ok": true }`

**에러 케이스**

| 상황 | `error` 메시지 |
|---|---|
| 존재하지 않는 id | `글을 찾을 수 없어요.` |
| 비밀번호 불일치 | `비밀번호가 일치하지 않아요.` |

---

## 5. 답글 작성 — `guestbook_add_reply`

```
POST /rest/v1/rpc/guestbook_add_reply
```

**요청 본문**

| 파라미터 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `p_entry_id` | uuid | ✅ | 답글을 달 원본 글의 id |
| `p_name` | string | ✅ | 이름 또는 별명 (1~20자) |
| `p_content` | string | ✅ | 내용 (1~500자) |
| `p_password` | string | ✅ | 비밀번호 (4자 이상) |

```json
{
  "p_entry_id": "3f2b1c9a-...",
  "p_name": "방문자A",
  "p_content": "반가워요!",
  "p_password": "reply-pw"
}
```

**응답**: `{ "ok": true }`

**에러 케이스**

| 상황 | `error` 메시지 |
|---|---|
| 비밀번호 4자 미만 | `비밀번호는 4자 이상이어야 해요.` |
| 원본 글이 이미 삭제됨 | `원본 글이 삭제되었어요.` |
| 이름/내용 길이 초과·공백 | `이름(1~20자)과 내용(1~500자)을 확인해 주세요.` |

---

## 6. 답글 수정 — `guestbook_update_reply`

```
POST /rest/v1/rpc/guestbook_update_reply
```

**요청 본문**

| 파라미터 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `p_id` | uuid | ✅ | 수정할 답글의 id |
| `p_password` | string | ✅ | 작성 시 입력한 비밀번호 |
| `p_content` | string | ✅ | 새 내용 (1~500자) |

```json
{
  "p_id": "8e4d2f1b-...",
  "p_password": "reply-pw",
  "p_content": "수정된 답글입니다."
}
```

**응답**: `{ "ok": true }`

**에러 케이스**

| 상황 | `error` 메시지 |
|---|---|
| 존재하지 않는 id | `답글을 찾을 수 없어요.` |
| 비밀번호 불일치 | `비밀번호가 일치하지 않아요.` |
| 내용 길이 초과·공백 | `내용은 1~500자여야 해요.` |

---

## 7. 답글 삭제 — `guestbook_delete_reply`

```
POST /rest/v1/rpc/guestbook_delete_reply
```

**요청 본문**

| 파라미터 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `p_id` | uuid | ✅ | 삭제할 답글의 id |
| `p_password` | string | ✅ | 작성 시 입력한 비밀번호 |

```json
{
  "p_id": "8e4d2f1b-...",
  "p_password": "reply-pw"
}
```

**응답**: `{ "ok": true }`

**에러 케이스**

| 상황 | `error` 메시지 |
|---|---|
| 존재하지 않는 id | `답글을 찾을 수 없어요.` |
| 비밀번호 불일치 | `비밀번호가 일치하지 않아요.` |

---

## JavaScript 호출 예시

`index.html`에서 실제로 사용하는 방식과 동일합니다.

```js
const SUPABASE_URL = "https://jcspkkaszsqlwnokqpmz.supabase.co";
const SUPABASE_KEY = "sb_publishable_4mUadSjAbg1N41XjfH1ZjA_Rdcru2gq";

async function rpc(fn, params = {}) {
  const res = await fetch(`${SUPABASE_URL}/rest/v1/rpc/${fn}`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "apikey": SUPABASE_KEY,
      "Authorization": `Bearer ${SUPABASE_KEY}`,
    },
    body: JSON.stringify(params),
  });
  if (!res.ok) throw new Error(`요청 실패 (${res.status})`);
  return res.json();
}

// 사용 예
const entries = await rpc("guestbook_list");

const result = await rpc("guestbook_add_entry", {
  p_name: "주희",
  p_content: "안녕하세요!",
  p_password: "my-secret",
});
if (!result.ok) alert(result.error);
```

---

## 보안 참고사항

- **공개 키는 노출되어도 안전하게 설계**되어 있습니다. 테이블 직접 읽기/쓰기는 RLS로 전면 차단되어 있고, 위 7개 함수를 통해서만 접근할 수 있습니다.
- 비밀번호는 서버(DB 함수) 안에서 bcrypt로 해시되어 저장·검증됩니다. 평문 비밀번호는 저장되지 않으며, API 응답에 해시도 포함되지 않습니다.
- 비밀번호 검증·글자 수 검증이 모두 DB 함수 내부에서 실행되므로, 클라이언트를 조작해도 우회할 수 없습니다.
- 비밀번호를 잊으면 API로는 수정·삭제할 수 없습니다. [Supabase 대시보드](https://supabase.com/dashboard/project/jcspkkaszsqlwnokqpmz) → Table Editor에서 직접 관리해야 합니다.

## 관련 문서

- 프로젝트 개요·화면·배포: [`../README.md`](../README.md)
- DB 스키마: Supabase `todo-app` 프로젝트의 `guestbook_entries`, `guestbook_replies` 테이블 (마이그레이션명: `create_guestbook`)
