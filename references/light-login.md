# 가벼운 로그인 (모양 ③)

`link-track.md` 5단계에서 읽어요. **닉네임(또는 이메일) + 비밀번호**로 들어가서, 어느 기기에서나 같은 기록을 보게 해요. 소셜 로그인, 인증 메일, 비밀번호 찾기는 넣지 않아요.

사용자에게 설명해요.
> "로그인과 저장은 Supabase라는 무료 서비스에 맡겨요. 비밀번호를 안전하게 바꿔서 저장하는 일을 Supabase가 해 줘서, 우리가 직접 만들 때보다 훨씬 안전해요. 카드 등록은 필요 없어요."

## 아이디 방식 고르기 (객관식)

| 보기 | 설명 |
|---|---|
| 닉네임 (추천) | 영문 소문자 · 숫자 · 밑줄 3~20자. 가장 간단해요. 비밀번호를 잊으면 찾을 방법이 없어요 |
| 이메일 | 평소 쓰는 이메일. 인증 메일은 안 보내요. 나중에 비밀번호 찾기를 붙이기 쉬워요 |

어느 쪽이든 "비밀번호는 다른 사이트와 다르게 정하고, 휴대폰 메모나 비밀번호 관리 앱에 적어 두세요"라고 안내해요.

---

## 순서

### 1. Supabase 프로젝트 만들기 (사용자가 직접)

1. supabase.com에서 가입해요 (GitHub 또는 이메일). 플랜은 **Free**, 카드 등록 화면이 나오면 멈춰요.
2. New project를 만들어요.
   - 이름: 영어 이름과 같게
   - 지역: Northeast Asia (Seoul)
   - DB 비밀번호: 사용자가 정하고 따로 적어 둬요. Claude에게 알려 주지 않아도 돼요.
3. 로그인 설정을 바꿔요. 위치는 Supabase 화면이 바뀌면 달라질 수 있어서, 사용자 화면 캡처를 보고 안내해요.
   - Authentication → Sign In / Providers → **Email**: 켜기
   - 같은 곳의 **Confirm email**: 끄기. 인증 메일을 보내지 않게 해요.
   - 비밀번호 최소 길이: **8자**

### 2. 키를 `.env.local`에 넣기 (사용자가 직접 붙여 넣기)

Project Settings → API(또는 API Keys)에서 두 가지를 복사해요.
- Project URL
- 공개용 키: `anon` 또는 `publishable`

`<영어-이름>/.env.local` 파일을 Claude가 빈칸으로 만들어 두면, 사용자가 값을 붙여 넣어요.
```
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
```

사용자에게 설명해요.
> "이 공개용 키는 화면 코드에 들어가도 괜찮은 키예요. 대신 DB에 '자기 것만 보기' 잠금(RLS)을 걸어서 지켜요. 같은 화면에 있는 `service_role`이나 `secret` 키는 관리자 열쇠라서 절대 복사하지 않아요."

`.env.local`이 `.gitignore`에 들어 있는지 확인해요. create-next-app 기본값에 들어 있어요.

### 3. 테이블과 잠금(RLS) 만들기

Claude가 `<영어-이름>/supabase/schema.sql`을 써요. 사용자는 Supabase 대시보드 → SQL Editor에 붙여 넣고 Run을 눌러요. 테이블 이름과 칸은 PRD의 데이터 칸에 맞춰요.

```sql
-- 예: 기록 테이블. 칸은 PRD에 맞게 바꿔요.
create table public.entries (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null default auth.uid() references auth.users (id) on delete cascade,
  content text not null check (char_length(content) between 1 and 2000),
  created_at timestamptz not null default now()
);

-- 잠금 켜기: 이 줄이 없으면 링크를 아는 누구나 모든 기록을 볼 수 있어요
alter table public.entries enable row level security;

-- 로그인한 사람은 자기 기록만 보고, 쓰고, 고치고, 지울 수 있어요
create policy "own rows: select" on public.entries
  for select to authenticated using ((select auth.uid()) = user_id);
create policy "own rows: insert" on public.entries
  for insert to authenticated with check ((select auth.uid()) = user_id);
create policy "own rows: update" on public.entries
  for update to authenticated using ((select auth.uid()) = user_id) with check ((select auth.uid()) = user_id);
create policy "own rows: delete" on public.entries
  for delete to authenticated using ((select auth.uid()) = user_id);
```

`(select auth.uid())`처럼 괄호로 감싸면 Supabase 대시보드의 성능 경고가 뜨지 않아요. 뜻은 `auth.uid()`와 같아요.

- 테이블을 하나 더 만들면 그 테이블에도 `enable row level security`와 네 가지 정책을 똑같이 넣어요.
- `user_id`는 DB가 로그인한 사람으로 자동으로 채워요. 화면 코드에서 보내지 않아요.
- 글자 수 제한(`check`)을 넣어 두면 누가 아주 긴 글을 넣어 DB를 채우는 걸 막아요.

### 4. 코드 붙이기

1. `npm install @supabase/supabase-js` (무료 오픈소스, 이름과 이유를 말해요)
2. `src/lib/supabase.ts`에 `getSupabase()` 함수를 만들어요. 두 값이 비어 있으면 `null`을 돌려주고, 있으면 그때 처음 `createClient`를 한 번만 만들어 돌려줘요. 주소 모양이 틀려도 오류가 나니 `try/catch`로 감싸요.
   - **파일 맨 위에서 바로 `createClient`를 부르지 않아요.** 그러면 키를 넣기 전이나 Vercel 첫 배포에서 `npm run build`가 "supabaseUrl is required"로 멈춰요.
   - `getSupabase()`가 `null`이면 화면에 "Supabase 주소와 키를 `.env.local`에 넣어 주세요 (배포한 링크라면 Vercel 환경 변수에 넣고 다시 배포)"를 띄워요.
3. 로그인 화면은 **가입하기 / 들어가기** 탭 두 개로 만들어요.
   - 닉네임 방식: 입력값이 `^[a-z0-9_]{3,20}$`인지 먼저 확인하고, 내부에서 `<닉네임>@<영어-이름>.local`로 바꿔 `signUp` / `signInWithPassword`에 넣어요. 사용자에게 이 이메일 모양은 보여 주지 않아요.
     - 이 도메인은 `src/lib/nickname.ts` 한 곳에 두고 **한 번 정하면 바꾸지 않아요.** 바꾸면 이미 가입한 계정으로 들어올 수 없어요.
     - 가입이 `email_address_invalid`나 "Email not confirmed"로 막히면 도메인을 바꾸지 말고, Supabase에서 **Confirm email이 꺼져 있는지** 먼저 확인해요. 켜진 채로 가짜 주소에 가입을 반복하면 메일이 반송돼서 프로젝트의 메일 발송이 막힐 수도 있어요.
     - 닉네임 칸에는 `autoCapitalize="none"`, `autoCorrect="off"`를 넣고, 입력하는 대로 소문자로 바꿔요. 아이폰이 첫 글자를 대문자로 바꿔서 규칙에 걸리는 걸 막아요.
   - 이메일 방식: 그대로 넣어요.
   - 비밀번호는 8자 이상인지 화면에서도 먼저 확인하고, 가입할 때는 "비밀번호 한 번 더" 칸을 둬요. 비밀번호 찾기가 없어서 오타로 가입하면 다시 들어올 방법이 없어요.
   - `signUp`이 오류 없이 끝났는데 `data.session`이 비어 있으면 Confirm email이 켜져 있는 거예요. "가입은 됐는데 바로 들어가지 못했어요. Supabase에서 'Confirm email'을 꺼 주세요"를 띄워요.
   - 오류 문구는 쉬운 말로 바꿔요. 예: "닉네임이나 비밀번호가 맞지 않아요", "이미 있는 닉네임이에요".
4. 로그인 상태는 `onAuthStateChange`로 따라가요. 로그인 안 했으면 로그인 화면, 했으면 본 화면을 보여 주고, 위쪽에 "○○님 · 나가기"를 둬요.
   - 콜백 안에서는 세션만 상태에 넣어요. 기록 불러오기는 로그인한 사람이 바뀔 때 도는 별도 `useEffect`에서 해요. 콜백 안에서 다른 Supabase 요청을 기다리면 화면이 멈출 수 있어요.
5. 4단계에서 가짜 데이터로 만든 화면을 `supabase.from('<테이블>')`의 select / insert / update / delete로 바꿔요. 불러오는 동안에는 "불러오는 중…"을, 실패하면 "인터넷 연결을 확인해 주세요"를 보여 줘요.
6. 커밋해요. `.env.local`이 커밋에 들어가지 않았는지 `git status`로 확인해요.

### 5. 잠금이 진짜 걸렸는지 확인하기 (꼭 해요)

1. Claude가 테스트 계정 두 개의 값을 정해 `WORKLOG.md`의 "테스트 계정" 칸에 적어요. 예: `test_a`, `test_b`, 비밀번호는 임의로 만들어요. **가입과 로그인은 사용자가 로컬 링크에서 직접 해요.** 이 소프트웨어 안에서만 쓰는 테스트용이라, 다른 사이트 비밀번호와 겹치지 않게 해요.
2. 사용자가 `test_a`로 기록을 하나 남기고, 나가서 `test_b`로 들어가 **안 보이는지** 확인해요.
3. Claude가 `<영어-이름>` 폴더에서 로그인 없이 공개용 키로 직접 물어봐요. `.env.local`은 Next.js만 읽는 파일이라 먼저 불러와야 해요.
   ```
   set -a; source .env.local; set +a
   curl -s "$NEXT_PUBLIC_SUPABASE_URL/rest/v1/<테이블>?select=*" -H "apikey: $NEXT_PUBLIC_SUPABASE_ANON_KEY"
   ```
   | 결과 | 뜻 |
   |---|---|
   | `[]` | 통과. 로그인 없이는 아무것도 안 보여요 |
   | `permission denied` (42501) | 통과. 막혀 있다는 뜻이에요 |
   | 기록이 보여요 | RLS가 빠졌어요. 배포하면 안 돼요 |
   | `Could not find the table`, `does not exist` | 통과가 아니에요. SQL을 안 돌렸거나 테이블 이름이 달라요 |
   | 아무것도 안 나오거나 `URL rejected` | `.env.local` 값이 비어 있어요 |

   윈도우 PowerShell에서는 위 명령 대신 Claude가 같은 요청을 Node 한 줄 스크립트로 보내요.
4. 확인이 끝나면 사용자가 Supabase 대시보드 → Authentication → Users에서 테스트 계정을 지워요.

사용자에게 결과를 보여 줘요.
> "다른 계정으로 들어가면 내 기록이 안 보이는 걸 확인했어요. 링크를 아는 사람이 가입해도 자기 기록만 보여요."

---

## 이번에 넣지 않는 것

| 기능 | 이유 | 다음 버전에서 |
|---|---|---|
| 카카오 · 구글 로그인 | 각 회사에 앱 등록, 키 관리가 필요해요 | 소셜 로그인 붙이기 |
| 인증 메일 · 비밀번호 찾기 | 메일 발송 설정이 필요해요 | 이메일 방식이면 붙이기 쉬워요 |
| 여러 명이 같은 기록 같이 보기 | 누가 무엇을 볼지 권한 규칙을 따로 짜야 해요 | 함께 쓰기 |
| 회원 탈퇴 버튼 | 지금은 사용자가 대시보드에서 지울 수 있어요 | 탈퇴 기능 |

## 알아 두기

- Supabase 무료 프로젝트는 **7일 동안 아무도 안 쓰면 잠들어요.** 대시보드에서 Restore 버튼 한 번이면 깨어나고, 돈은 들지 않아요.
- 무료 플랜은 프로젝트를 2개까지 만들 수 있어요.
- 이메일 방식이면 이메일도 개인정보예요. 기록 칸에 주민번호, 계좌번호 같은 민감한 정보는 받지 않아요.
- **회사 업무 내용**(거래처 미팅, 고객 문의 등)을 담는다면, 개인 서비스에 저장해도 되는지 회사 규정을 한 번 확인하자고 안내해요. 고객 · 거래처 담당자의 연락처 같은 남의 개인정보는 적지 않게 해요.
- 상사나 동료에게 보여 주고 싶을 때 **내 아이디와 비밀번호를 알려 주지 않아요.** "복사해서 보내기"나 폰 공유 버튼으로 내용만 보내요.
