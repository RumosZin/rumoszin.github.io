---
title: "[SvelteKit/Supabase] Build a User Management App with SvelteKit 공식 문서의 오류"
author:
date: 2024-03-30 22:17:00 +0900
categories: [Frontend, Svelte / SvelteKit]
tags: [SvelteKit, Web, 인턴]
---

## **Build a User Management App with SvelteKit를 시작한 이유**

SvelteKit은 Svelte를 사용하여 웹 애플리케이션을 구축하기 위한 Javascript 기반 프레임워크이다. 인턴십에서 다뤄야 할 프레임워크였기 때문에, 공부하게 되었다.

먼저 [huntabyte의 SvelteKit 강의](https://www.youtube.com/watch?v=EQy-AYhZIlE&list=PLq30BP0TIcqXP149TyFMfRhnMT6T5--e5)를 수강했다. Svelte, SvelteKit의 유용한 점들을 소개하는 내용이 주를 이뤘고, 유용한 점들을 확인하는 간단한 실습을 했다. 처음 배울 때는 이론을 실전에 적용하면서 익히는 것이 낫다고 생각하기 때문에, SvelteKit의 이점을 활용하면서 프레임워크를 익히기 좋은 예제를 찾게 되었다.

이왕이면 데이터베이스도 붙여서 확인해보고 싶었는데, huntabyte의 강의에서 SvelteKit 애플리케이션의 데이터베이스로 supabase를 적극 추천했기 때문에, 데이터베이스는 고민 없이 supabase를 선택했다. SvelteKit-supabase로 검색한 결과, supabase 공식 문서에 내용이 잘 정리되어 있으면서 Full code도 공식 레포지토리에 있는, User management tutorial을 발견하게 되었다.

[Build a User Management App with SvelteKit](https://supabase.com/docs/guides/getting-started/tutorials/with-sveltekit)

[Full code](https://github.com/supabase/supabase/tree/master/examples/user-management/sveltekit-user-management)

공식 문서를 참고한 결과, 결론은 **공식 문서에 나오는대로 해서는 User management 애플리케이션을 만들 수 없다**는 것이다. **따라서 나만의 방식으로 User management 애플리케이션을 만들고 공유하려 한다!**

## **문제 상황**

Issues에 나온 문제가 아닌, 직접 겪은 문제만 먼저 정리한다. 위에 적은 공식 문서, Full code를 직접 해본 결과 만난 문제이다.

**`src/routes/account/+page.svelte`의 import 문제**

공식 문서를 따라하고 Full code를 참고해서 `npm run dev`로 실행한다. `localhost:5173` 으로 접근하면 아래와 같은 결과가 나온다.

![magic link를 입력하는 home 화면](/assets/img/240330-1.png){: width="100%"}

이메일 주소를 입력하고 `SEND MAGIC LINK` 버튼을 누르면 입력한 이메일 주소로 인증이 가능하다. 회원가입을 따로 구현하지 않아도, 이 magic link 인증을 통해 회원가입을 할 수 있다. 로그인 할 때 이 이메일 주소를 이용하면 된다.

그런데 메일에 들어가서 인증을 하고 링크로 돌아오면 localhost:5173/account 경로에서 문제가 생긴다.

`src/routes/account/+page.svelte`의 `import { enhance, type SubmitFunction } from '$app/forms'`에서 SubmitFunction에 대해 parse-error가 발생한다.

## **해결 시도**

reddit에서 [정확히 같은 문제를 겪고 있는 글](https://www.reddit.com/r/Supabase/comments/155jeim/having_trouble_with_the_usermanagement_tutorial/)을 찾았다. 참고해서 두 가지 시도를 했지만 에러는 그대로였다.

### **시도 1 : TypeScript compilerOption 수정**

공식 문서에 다음과 같은 글이 있다.

> If you are using TypeScript the compiler might complain about event.locals.supabase and event.locals.getSession, this can be fixed by updating your src/app.d.ts with the content below.
> {: .prompt-warning }

따라서 compilerOption을 아래처럼 수정했다.

```typescript
"compilerOptions": {
    "strict": true,
    "allowUnreachableCode": false,
    "exactOptionalPropertyTypes": true,
    "noImplicitAny": true,
    "noImplicitOverride": true,
    "noImplicitReturns": true,
    "noImplicitThis": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true
}
```

{: file='tsconfig.json'}

### **시도 2 : SubmitFunction import 수정**

문제가 생긴 import 문은 다음과 같다.

```svelte
import { enhance, SubmitFunction } from '$app/forms';
```

{: file='/account/+page.svelte'}

아래와 같이 수정한다.

```svelte
import { enhance } from '$app/forms';
import type { SubmitFunction } from '@sveltejs/kit';
```

{: file='/account/+page.svelte'}

### **시도 3 : SvelteKit 공식 문서 확인**

`$app/forms`에 `SubmitFunction` 자체가 없는 것 같아 [SvelteKit 공식 문서](https://kit.svelte.dev/docs/modules#$app-forms)를 확인했다. 확인 결과 `$app/forms`에는 `applyAction`, `deserialize`, `enhance`만 존재한다.

SvelteKit 공식 문서에 `SubmitFunction`을 검색한 결과, `use:enhance`를 커스텀할 때 form 제출 전 `SubmitFunction`을 수행하도록 할 수 있다고 한다.

결론적으로 [User management App with SvelteKit 공식 문서](https://supabase.com/docs/guides/getting-started/tutorials/with-sveltekit)에서 다루고 있는 코드와, SvelteKit 공식 문서에 나오는 방법이 다르다. 뿐만 아니라, [튜토리얼 Full code](https://github.com/supabase/supabase/tree/master/examples/user-management/sveltekit-user-management)와도 달랐다. Full code에는 없는 폴더 `schema`에서 `Database`를 import 하는 부분도 존재한다.

## **Issues 확인**

이런 문제에 대해서 분명 Issue가 존재할 것이라고 생각했다. 확인 결과 내가 겪었던 문제들을 포함해 더 많은 오류를 찾아낸 사람이 [Issue](https://github.com/supabase/supabase/issues/17375)를 올렸다. [연결된 Pull Request](https://github.com/supabase/supabase/pull/17376)가 있어서 해결 방법을 찾을 수 있다고 생각했지만, 어떤 이유인지 Closed 상태였다.

결론적으로, [Build a User Management App with SvelteKit](https://supabase.com/docs/guides/getting-started/tutorials/with-sveltekit)를 그대로 따라하면 원하는 결과를 얻을 수 없다.

## **앞으로**

가능한 방법을 찾아서 간단한 User Management를 구현한 후 오픈소스 커뮤니티에 공유할 예정이다. 회원가입/로그인을 구현하다가 지쳐 포기하는 일이 줄어들었으면 한다.

작년 여름부터 생각만 하던 아이디어를 구체화할 것이다. 사용자 관리가 고민이어서 미루고 있었는데, SvelteKit를 알고 나서 걱정을 덜게 되었다. 이 프로젝트도 블로그에 자세히 공유할 예정이다.

<script src="https://utteranc.es/client.js"
        repo="RumosZin/rumoszin.github.io"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
