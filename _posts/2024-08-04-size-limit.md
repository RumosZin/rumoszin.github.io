---
title: "첨부 파일 크기로 인한 Body Size Limit, File Size Limit 문제 해결하기"
author:
date: 2024-08-04 20:00:00 +0900
categories: [Infra & DevOps, Web]
tags: [인턴, Size limit]
---

<br>

인턴십에서 진행하는 프로젝트 개발을 마치고 사용자 피드백을 받으며 애자일하게 이슈를 처리하고 있는데, 첨부 파일과 관련해서 Body Size Limit, File Size Limit 문제가 발생했다. 어떤 오류 로그를 가지고 있고 어떻게 해결했는지 정리해두자!

## **Body Size Limit**

SvelteKit application adapter-node에서 설정이 작아서 발생하는 문제이다. 서버에서 설정한 최대 허용 페이로드의 크기를 넘었음을 의미한다.

```
2024-08-07 00:51:22:SS +00:00: SvelteKitError: Content-length of 4706173 exceeds limit of 524288 bytes.
2024-08-07 00:51:22:SS +00:00:     at Object.start (file:///app/build/handler.js:984:19)
2024-08-07 00:51:22:SS +00:00:     at setupReadableStreamDefaultController (node:internal/webstreams/readablestream:2435:23)
2024-08-07 00:51:22:SS +00:00:     at setupReadableStreamDefaultControllerFromSource (node:internal/webstreams/readablestream:2467:3)
2024-08-07 00:51:22:SS +00:00:     at new ReadableStream (node:internal/webstreams/readablestream:278:7)
2024-08-07 00:51:22:SS +00:00:     at get_raw_body (file:///app/build/handler.js:973:9)
2024-08-07 00:51:22:SS +00:00:     at getRequest (file:///app/build/handler.js:1054:7)
2024-08-07 00:51:22:SS +00:00:     at Array.ssr (file:///app/build/handler.js:1248:19)
2024-08-07 00:51:22:SS +00:00:     at handle (file:///app/build/handler.js:1318:23)
2024-08-07 00:51:22:SS +00:00:     at file:///app/build/handler.js:1318:40
2024-08-07 00:51:22:SS +00:00:     at Array.<anonymous> (file:///app/build/handler.js:1237:4)
2024-08-07 00:51:22:SS +00:00:     at handle (file:///app/build/handler.js:1318:23)
2024-08-07 00:51:22:SS +00:00:     at file:///app/build/handler.js:1318:40
2024-08-07 00:51:22:SS +00:00:     at Array.<anonymous> (file:///app/build/handler.js:682:28)
2024-08-07 00:51:22:SS +00:00:     at handle (file:///app/build/handler.js:1318:23)
2024-08-07 00:51:22:SS +00:00:     at Array.<anonymous> (file:///app/build/handler.js:1324:10)
2024-08-07 00:51:22:SS +00:00:     at loop (file:///app/build/index.js:224:63) {
2024-08-07 00:51:22:SS +00:00:   status: 413,
2024-08-07 00:51:22:SS +00:00:   text: 'Payload Too Large'
2024-08-07 00:51:22:SS +00:00: }
```

[SvelteKit 공식 문서](https://kit.svelte.dev/docs/adapter-node#environment-variables-bodysizelimit)의 BODY_SIZE_LIMIT을 확인해보니 기본이 512KB이다. 공식 문서에 등장한대로 애플리케이션 배포 시 환경 변수에 Infinity를 설정하여 해결할 수 있었다.

**첨부 가능 파일 크기 제한을 두지 않고 Infinity로 무작정 설정하면 문제가 생기지 않을까? 보안 문제나 처리 속도가 느려지는 등의 문제를 생각했으나, 협력업체의 애플리케이션 사용자들에게는 파일 크기 제한 없이 첨부하는 기능이 중요하기 때문에 제한을 두지 않기로 했다.**

<br>

## **File Size Limit**

supabase storage-api에서 파일 크기 제한을 기본 값인 52.4MB로 해놓았는데, 이 제한에 걸려서 파일이 제대로 업로드 되지 못하는 문제이다.

```
FileRepository Upload Method Error
{
  statusCode: '413',
  error: 'Payload too large',
  message: 'The object exceeded the maximum allowed size'
}
```

[Supabase Self-Hosting 공식 문서](https://supabase.com/docs/guides/self-hosting/storage/config) / [Supabase Storage .env 예시 코드](https://github.com/supabase/storage/blob/master/.env.sample)에서 따로 설정이 가능함을 알게 되었다.

**처음에는 문제의 원인을 파악하기 어려웠는데, 52MB 정도의 파일에서는 문제가 안생기는데 66MB의 파일이 저장되지 않는다는 피드백에서 힌트를 얻어 FILE_SIZE_LIMIT을 524MB로 변경하고, 변경 사항을 반영하기 위해서 storage-api 컨테이너를 재시작해서 파일 크기 문제를 해결할 수 있었다.**

<br>

빠르게 해결해야 했던 문제였는데, 공식 문서에 관련 내용이 잘 정리되어 있어서 수월히 해결할 수 있었다. 특히나 Supabase 셀프 호스팅의 경우는 사용자가 많이 없어서 공식 레포지토리의 Issue를 많이 의지해야 했는데, 여러 컨트리뷰터들의 상세한 설명들을 참고해서 이 문제에도 해결 방법을 적용해볼 수 있었다.

**만약 Supabase Self-Hosting에 난항을 겪고 있다면, [공식 레포지토리의 self-hosted 라벨](https://github.com/supabase/supabase/labels/self-hosted)에 있는 모든 이슈들을 확인하고 답변들을 읽어보는 것을 추천한다! 당장 문제를 해결할 수 없어도 힌트를 얻는데 큰 도움이 된다.**

<br>
<br>

<script src="https://utteranc.es/client.js"
        repo="RumosZin/rumoszin.github.io"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
