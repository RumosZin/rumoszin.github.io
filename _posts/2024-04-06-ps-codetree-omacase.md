---
title: "[CodeTree/C++] 코드트리 오마카세"
author:
date: 2024-04-06 14:28:00 +0900
categories: [Computer Science, Problem Solving]
tags: [Problem Solving, C++, 시뮬레이션]
---

## **문제**

[[코드트리/C++] 코드트리 오마카세 (삼성 SW 역량테스트 2023 하반기 오후 2번 문제)](https://www.codetree.ai/training-field/frequent-problems/problems/codetree-omakase/description?page=1&pageSize=20)

## **문제 해석**

원형의 초밥 벨트와 L개의 의자가 놓여있다. 회전 초밥은 각 의자 앞에 여러 개 놓일 수 있으며 초밥 벨트는 1초에 한 칸씩 시계방향으로 회전합니다.

- 초밥 벨트의 마지막 위치 L-1의 다음 위치는 처음 위치인 0이다.
- 초밥 벨트가 회전하므로 위치 x에 놓인 초밥은 매 시간 t마다 달라진다.

주방장의 초밥 만들기

- 같은 위치 x에 여러 초밥이 올라갈 수 있다.
- 같은 name의 초밥이 여러 개 올라갈 수 있다.
- 회전 초밥 벨트가 회전함에 따라, 위치 x의 초밥들은 1초 뒤 위치 x + 1로 이동한다.

손님 입장

- 시각 t에 위치 x에 있는 의자로 가서 착석한다. 초밥 벨트에 자신의 이름이 적혀있으면 먹을 수 있고, 그렇지 않으면 자신의 이름이 적힌 초밥이 위치 x에 올 때까지 기다려야 한다.
- 자신의 이름이 적힌 초밥을 n개 먹고 자리를 떠난다. n개를 전부 먹을 때까지 자리를 지킨다.

사진 촬영

- 시간의 흐름에 따른 회전 초밥 벨트가 회전한다. 그에 따라 초밥들의 위치도 변경된다. 이때 손님의 위치에 손님의 이름이 적힌 초밥이 있다면, 초밥은 사라진다.
- 사진이 촬영되는 순간에는, 손님에게 먹힐 예정인 초밥과, 손님이 아직 오지 않은 초밥, n개를 먹기 위해 기다리는 손님이 존재한다.

위와 같이 문제를 해석한 결과, 필요한 정보들은 다음과 같다.

- 명령어 목록
- 손님 목록
- 손님과 관련된 명령어
- 손님의 위치
- 손님의 입장 시간

## **구현 방향**

명령어 Q가 들어올 때마다 초밥 만들기, 손님 입장, 사진 촬영을 고려하는 것은 비효율적이다.

핵심은 **시간 t에, 손님이 자신의 초밥을 먹었는지** 여부이므로, **손님의 입장 시간**, **초밥이 먹히고 없어지는 시간**, **손님이 n개의 초밥을 먹은 후 퇴장 시간**을 알고 있는 것이 중요하다.

주어진 명령은 초밥 생성, 손님 입장, 사진 촬영으로 3개이다. **그런데 사진 촬영은 사실상 초밥 소멸, 손님 퇴장을 알아야 하므로, 명령을 추가하여 관리한다.**

### **변수 선언**

```cpp
#include <iostream>
#include <vector>
#include <map>
#include <set>
#include <algorithm>

using namespace std;

int L, Q;

class Query {
    public :
        int cmd, t, x, n;
        string name;

        Query(int cmd, int t, int x, string name, int n) {
            this->cmd = cmd;
            this->t = t;
            this->x=x;
            this->name = name;
            this->n = n;
        }
};

vector<Query> queries; // 명령어 저장
set<string> names; // 등장한 사람 목록
map<string, vector<Query>> p_queries; // 손님과 관련된 초밥 명령
map<string, int> entry_time; // 손님의 입장 시간
map<string, int> position; // 손님의 위치
map<string, int> exit_time; // 손님의 퇴장 시간
```

- `#include <~>`는 변수 선언 시 사용하는 라이브러리가 있다면 어렵지 않게 적을 수 있다.
- 명령의 형식이 비슷하므로, class로 선언한다. typedef struct로 선언해도 괜찮지만, **문제에 주어진 명령 외에 2개의 명령을 추가할 예정이므로 class로 선언하고 생성자를 만든다.**

- 아래 3개의 선언은 문제에서 주어지기 때문에 쉽게 작성할 수 있다.

```cpp
vector<Query> queries; // 명령어 저장
map<string, int> entry_time; // 손님의 입장 시간
map<string, int> position; // 손님의 위치
```

- 초밥 소멸, 손님 퇴장 명령도 관리해야 하기 때문에, name과 관련된 명령어를 따로 관리해아 한다.

```cpp
map<string, vector<Query>> p_queries; // 손님과 관련된 초밥 명령
map<string, int> exit_time; // 손님의 퇴장 시간
```

### **input : O(Q)**

```cpp
int main() {
    // input
    cin >> L >> Q;
    for(int i = 0; i < Q; i++) {
        int cmd = -1;
        int t = -1, x = -1, n = -1;
        string name;
        cin >> cmd;
        if(cmd == 100) cin >> t >> x >> name;
        else if(cmd == 200) cin >> t >> x >> name >> n;
        else cin >> t;

        queries.push_back(Query(cmd, t, x, name, n)); // 명령어 저장

        // 손님의 초밥 명령 관리
        if(cmd == 100) p_queries[name].push_back(Query(cmd, t, x, name, n));

        // 손님이 입장한 시간과 위치 관리
        else if(cmd == 200) {
            names.insert(name); // 손님 목록
            entry_time[name] = t; // 손님의 입장 시간
            position[name] = x; // 손님의 위치
        }
    }
    // ...
}
```

### **손님마다 초밥을 먹는 시간 계산 : O(Q)**

- 초밥의 소멸은 항상 손님이 초밥을 먹는 경우에 이루어진다. 따라서 초밥의 생성과 손님의 입장 시간의 순서만 고려하면 된다.
- 초밥 생성 -> 손님 입장 : 손님이 입장했을 때 초밥의 위치를 구하고, 두 위치의 차이를 구한다.
- 손님 입장 -> 초밥 생성 : 초밥의 생성 위치와 손님의 위치 차이를 구한다.
- `p_queries`에 손님의 초밥에 대한 정보가 주어지므로, 손님의 모든 초밥에 대해 먹는 시간을 계산한다. **모든 초밥은 손님에 의해 전부 없어진다고 주어졌으므로, 초밥을 n개 먹는지 확인할 필요가 없다. p_queries의 명령어를 전부 수행한 것이 초밥을 정확히 n개 먹는 것이다.**

```cpp
int main() {
    // input

    for(string name : names) { // 모든 손님들에 대해 계산
        // 퇴장 시간
        exit_time[name] = 0;

        // 손님의 초밥 명령 목록
        for(Query q : p_queries[name]) {
            // 초밥이 손님 등장 전 주어진 상황
            int time_to_removed = 0;
            if(q.t < entry_time[name]) {
                // 손님의 입장 시간에 초밥의 위치 구하기
                int t_sushi_x = (q.x + (entry_time[name] - q.t)) % L;

                // 몇 초 지나야 초밥을 만나는지 계산
                int additional_t = (position[name] - t_sushi_x + L) % L;

                time_to_removed = entry_time[name] + additional_t;
            }

            // 초밥이 손님 등장 이후 주어진 상황
            else {
                int additional_t = (position[name] - q.x + L) % L;
                time_to_removed = q.t + additional_t;
            }

            // 초밥이 사라지는 시간 중 가장 늦은 시간 업데이트
            exit_time[name] = max(exit_time[name], time_to_removed);

            // 초밥이 사라지는 111번 쿼리 추가
            queries.push_back(Query(111, time_to_removed, -1, name, -1));

        }
    }
}

```

### **손님마다 퇴장 시간 명령어 추가 : O(Q) + O(QlogQ)**

- 앞서 구한 `exit_time`에 손님의 떠난 시간이 저장되므로, 명령어에 손님이 오마카세를 떠난 시간을 추가한다.
- 초밥 소멸 시간, 손님의 퇴장 시간 명령어를 추가했으므로 `queries`를 정렬한다.
- **시간 t의 사진 촬영은 시간 t의 초밥 회전, 손님의 식사 이후에 이루어지므로, t가 동일할 때 300 명령어가 뒤로 가도록 정렬한다.**

```cpp
bool cmp(Query q1, Query q2) {
    if(q1.t != q2.t) return q1.t < q2.t;
    return q1.cmd < q2.cmd;
}
```

```cpp
    // 손님이 마지막으로 초밥을 먹은 시간 t 계산
    // 손님이 t 시간에 오마카세를 떠난 222 Query 추가
    for(string name : names) {
        queries.push_back(Query(222, exit_time[name], -1, name, -1));
    }

    // 전체 쿼리를 시간 순으로 정렬
    // t가 동일하다면 300이 가장 뒤로 가도록 정렬
    sort(queries.begin(), queries.end(), cmp);
```

### **사진 촬영**

- 명령어 종류에 따라 초밥 생성/소멸, 손님 입장/퇴장을 알 수 있기 때문에, 시간 t에서 남은 초밥과 남은 손님을 알 수 있다.

```cpp
    int people_num = 0;
    int sushi_num = 0;
    for(int i = 0; i < queries.size(); i++) {
        if(queries[i].cmd == 100) // 초밥 추가
            sushi_num++;
        else if(queries[i].cmd == 111) // 초밥 제거
            sushi_num--;
        else if(queries[i].cmd == 200) // 사람 추가
            people_num++;
        else if(queries[i].cmd == 222) // 사람 제거
            people_num--;
        else // 사진 촬영
            cout << people_num << " " << sushi_num << "\n";
    }
    return 0;
```

## **전체 코드**

[[코드트리/C++] 코드트리 오마카세 / 풀이 코드](https://github.com/RumosZin/algorithm-study/blob/main/CodeForce/Samsung_codetree_omacase.cpp)

<script src="https://utteranc.es/client.js"
        repo="RumosZin/rumoszin.github.io"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
