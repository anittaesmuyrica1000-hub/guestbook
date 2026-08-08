# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

이 문서는 Claude Code가 이 저장소에서 작업할 때 참고하는 안내서입니다.

## 프로젝트 개요

비밀번호 기반 공개 방명록. 프런트엔드는 `index.html` 단일 파일(순수 HTML/CSS/JS, 빌드 없음)이고, 백엔드 로직은 전부 Supabase(PostgreSQL) 안의 RPC 함수로 존재한다. **이 저장소의 파일만 봐서는 코드의 절반(검증·비밀번호 로직)이 보이지 않는다** — 나머지 절반은 Supabase DB에 있다.

## 실행

```bash
python3 -m http.server 8931 -d .   # http://localhost:8931
```

빌드·테스트·린트 없음. 로컬과 배포 사이트가 같은 DB를 공유하므로 로컬에서 글을 쓰면 실서비스에도 보인다. 테스트로 남긴 글은 지울 것.

## 아키텍처

```
index.html ──POST /rest/v1/rpc/guestbook_*──▶ Supabase (guestbook 전용 프로젝트)
```

- **Supabase 프로젝트**: `guestbook` (`sxrorbvuhdbavfbaibuj`, 서울) — 방명록 전용. 테이블 `guestbook_entries`·`guestbook_replies`, RPC 함수 7개 (`guestbook_list`, `guestbook_add/update/delete_entry`, `guestbook_add/update/delete_reply`). 2026-08-08에 구 `todo-app` 프로젝트(`jcspkkaszsqlwnokqpmz`)에서 분리했고, todo-app은 무료 플랜 2개 제한 때문에 **일시중지(paused)** 상태다 — 되살리려면 다른 프로젝트를 정리해야 한다. 구 프로젝트의 guestbook 테이블은 분리 시점 데이터로 남아 있고(`backup-2026-08-08.json`과 동일), 새 글은 새 프로젝트에만 쌓인다.
- **보안 모델 (변경 시 반드시 유지)**: 두 테이블은 RLS on + 정책 없음 + anon/authenticated revoke로 직접 접근이 전면 차단되어 있고, 모든 접근은 SECURITY DEFINER RPC 함수를 통해서만 이루어진다. 비밀번호는 pgcrypto `crypt()`/`gen_salt('bf')`로 해시 저장·검증하며, 함수 안에서만 다룬다. `guestbook_list`는 `password_hash`를 응답에 포함하지 않는다. 새 기능을 추가할 때도 테이블에 직접 정책을 열지 말고 같은 RPC 패턴을 따를 것. Supabase 보안 어드바이저의 "anon can execute SECURITY DEFINER function" 경고는 의도된 설계다(로그인 없는 공개 방명록).
- **응답 규약**: 쓰기 함수는 모두 HTTP 200에 `{ok: true}` 또는 `{ok: false, error: "한국어 안내문"}`을 반환한다. 에러 문구는 사용자에게 그대로 노출되므로 해요체 한국어로 쓴다.
- **DB 변경 방법**: Supabase MCP의 `apply_migration` 사용 (기존 마이그레이션명: `create_guestbook`). 함수를 수정하면 `docs/API.md`의 파라미터·에러 메시지 표도 함께 갱신할 것.

## 프런트엔드 규약

- `index.html` 안에서 사용자 입력은 반드시 `esc()`로 이스케이프한 뒤 innerHTML에 넣는다 (XSS 방지).
- 확인·입력 UI는 브라우저 `alert`/`confirm`/`prompt` 대신 인라인 폼(`openReplyForm`/`openEditForm`/`openDeleteForm` 패턴)을 쓴다.
- UI 문구는 해요체 한국어.

## 배포

GitHub `anittaesmuyrica1000-hub/guestbook` → 사용자 본인 Vercel 계정(Juhee's projects, 팀 슬러그 `juhee-team`)의 `guestbook` 프로젝트 → https://guestbook-juhee-team.vercel.app

- **git 연동 자동 배포**: `main`에 push하면 자동 재배포된다. 배포는 push로 끝난다.
- 푸시 인증: remote에 토큰을 저장하지 않는다. 사용자에게 GitHub 토큰을 받아 푸시 URL에 일회성으로만 사용하고, `.git/config`에 토큰이 남지 않았는지 확인할 것.
- git 커밋 identity는 이 저장소에 local로 설정되어 있다 (GitHub noreply 이메일). global 설정을 바꾸지 말 것.
- Vercel MCP(`deploy_to_vercel`)는 **다른 계정**(jayjaewoongchoi-2571 팀)에 연결되어 있으므로 이 프로젝트 배포에 쓰지 않는다. 그 계정에 있는 구 프로젝트 `guestbook`·`juhee-guestbook`(juhee-guestbook.vercel.app 포함)은 폐기 대상이다.
- Vercel은 새 프로젝트에 Vercel Authentication(배포 보호)을 기본으로 켠다. 새 프로젝트가 302로 SSO 리다이렉트되면 프로젝트 Settings → Deployment Protection에서 꺼야 한다.

## 문서

- `README.md` — 개요·스키마·운영(데이터 확인/초기화는 Supabase 대시보드)
- `docs/API.md` — RPC 7개의 파라미터·응답·에러 메시지 상세. API 동작을 바꾸면 여기를 갱신
