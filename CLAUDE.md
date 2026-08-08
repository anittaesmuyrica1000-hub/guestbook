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
index.html ──POST /rest/v1/rpc/guestbook_*──▶ Supabase (todo-app 프로젝트)
```

- **Supabase 프로젝트**: `todo-app` (`jcspkkaszsqlwnokqpmz`, 서울). 무료 플랜 프로젝트 수 제한 때문에 무관한 todo 테이블들과 한 프로젝트를 공유한다. 방명록 소유는 `guestbook_` 접두사가 붙은 것들뿐: 테이블 `guestbook_entries`·`guestbook_replies`, RPC 함수 7개 (`guestbook_list`, `guestbook_add/update/delete_entry`, `guestbook_add/update/delete_reply`). `users`·`todos`·`categories` 등은 건드리지 말 것.
- **보안 모델 (변경 시 반드시 유지)**: 두 테이블은 RLS on + 정책 없음 + anon/authenticated revoke로 직접 접근이 전면 차단되어 있고, 모든 접근은 SECURITY DEFINER RPC 함수를 통해서만 이루어진다. 비밀번호는 pgcrypto `crypt()`/`gen_salt('bf')`로 해시 저장·검증하며, 함수 안에서만 다룬다. `guestbook_list`는 `password_hash`를 응답에 포함하지 않는다. 새 기능을 추가할 때도 테이블에 직접 정책을 열지 말고 같은 RPC 패턴을 따를 것. Supabase 보안 어드바이저의 "anon can execute SECURITY DEFINER function" 경고는 의도된 설계다(로그인 없는 공개 방명록).
- **응답 규약**: 쓰기 함수는 모두 HTTP 200에 `{ok: true}` 또는 `{ok: false, error: "한국어 안내문"}`을 반환한다. 에러 문구는 사용자에게 그대로 노출되므로 해요체 한국어로 쓴다.
- **DB 변경 방법**: Supabase MCP의 `apply_migration` 사용 (기존 마이그레이션명: `create_guestbook`). 함수를 수정하면 `docs/API.md`의 파라미터·에러 메시지 표도 함께 갱신할 것.

## 프런트엔드 규약

- `index.html` 안에서 사용자 입력은 반드시 `esc()`로 이스케이프한 뒤 innerHTML에 넣는다 (XSS 방지).
- 확인·입력 UI는 브라우저 `alert`/`confirm`/`prompt` 대신 인라인 폼(`openReplyForm`/`openEditForm`/`openDeleteForm` 패턴)을 쓴다.
- UI 문구는 해요체 한국어.

## 배포

Vercel 프로젝트 **`juhee-guestbook`** (팀 `team_nM4xor6LKVFp2G6SNOTA2Lrs`) → https://juhee-guestbook.vercel.app

- git 연동이 아니라 Vercel MCP `deploy_to_vercel`로 `index.html` 내용을 직접 업로드하는 방식이다. `index.html`을 수정해도 자동 배포되지 않으므로, 수정 후 같은 프로젝트명으로 다시 `deploy_to_vercel`을 호출해야 한다.
- 이 팀은 새 프로젝트에 Vercel Authentication(배포 보호)이 기본으로 켜진다. 새로 배포한 프로젝트가 302로 SSO 리다이렉트되면 `update_project_deployment_protection`으로 `ssoProtection: {enabled: false}` 처리할 것.
- 구 프로젝트 `guestbook`(긴 주소)은 폐기 대상이며 재사용하지 않는다.

## 문서

- `README.md` — 개요·스키마·운영(데이터 확인/초기화는 Supabase 대시보드)
- `docs/API.md` — RPC 7개의 파라미터·응답·에러 메시지 상세. API 동작을 바꾸면 여기를 갱신
