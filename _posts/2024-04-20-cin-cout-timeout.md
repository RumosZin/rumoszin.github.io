---
title: "[C++] cin, cout 시간 초과 해결 방법"
author:
date: 2024-04-20 18:28:00 +0900
categories: [Computer Science, Problem Solving]
tags: [Tips, C++]
---

### **cin, cout 사용 시 시간 초과 발생**

알고리즘의 시간 복잡도를 O(N)으로 맞게 구현 했는데도 시간 초과가 발생할 때가 있다. C++로 문제를 풀 때 cin, cout을 사용해서 문제가 발생한 것이다. **cin, cout은 scanf, printf보다 컴파일 속도가 느리다.**

코딩 테스트 시 scanf, printf와 cin, cout을 섞어서 사용하면 안되기 때문에, 무조건 cin과 cout을 사용해서 입출력을 처리해야 한다면 코드에 아래와 같은 선언을 작성하자.

```cpp
ios::sync_with_stdio(false);
cin.tie(NULL);
```

### **ios::sync_with_stdio(false);**

C++에서 사용하는 `#include <iostream>`과 C에서 사용하는 `#include <stdio.h>`를 동기화 시킨다. 따라서 `iostream`, `stdio`의 버퍼를 모두 사용하기 때문에 지연이 발생한다. **`false`로 설정해서 동기화를 하지 않고 C++만의 버퍼를 사용하도록 한다.**

### **cin.tie(NULL);**

`cin`은 `cout`에 바인딩 되어 있어서 cin에서 입출력을 수행하기 전에 `flush`를 호출해서 I/O 부담이 증가한다고 한다. 따라서 바인딩을 해제해서 사용한다.

### **"\n"으로 개행 사용**

`cout`에서 `endl`은 개행 문자를 출력하면서 출력 버퍼를 비우기 때문에 속도가 느리다. `"\n"`으로 개행을 처리해야 한다.

### **참고 자료**

- [https://wondangcom.tistory.com/631](https://wondangcom.tistory.com/631)
- [https://algospot.com/forum/read/2496/](https://algospot.com/forum/read/2496/)
- [https://wondangcom.tistory.com/631](https://wondangcom.tistory.com/631)

<script src="https://utteranc.es/client.js"
        repo="RumosZin/rumoszin.github.io"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
