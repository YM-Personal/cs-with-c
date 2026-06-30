# Stage 4-(4) — 몸 그리기와 사과/몸 겹침 방지

> 자라난 몸을 `o`로 그려서 드디어 **눈에 보이는 길이**를 만든다. 그리고 새 사과가 몸 위에 떨어지지 않도록 **다시 뽑기(rejection sampling)** 를 도입한다.

## 오늘의 목표

- 매 프레임 `body[1..length-1]`을 `o`로 보드에 박는다 → 몸이 화면에 나타남.
- 사과를 뽑을 때 **머리·몸과 겹치면 다시 뽑는다**.
- 첫 사과 배치에도 같은 규칙 적용 (일관성).
- 게임오버 / 자기 몸 충돌은 다음 stage(5).

기대 화면 (몇 번 먹은 뒤):
```
##################
#................#
#.....*..........#
#......OoO.......#       ← 어 잠깐 'O'가 두 개?
#................#
```

→ 'O'가 두 개로 보이면 안 됨. 머리만 `O`, 나머지는 `o`. 정확한 예시:
```
##################
#................#
#.....*..........#
#......Ooooooo...#       ← 머리(O) + 몸 여러 칸(o)
#................#
```

---

## 배경 이야기

### 그리는 순서가 곧 우선순위

보드 한 칸엔 글자 하나만 들어가니, **나중에 박는 게 이긴다**(덮어쓴다). 그래서 매 프레임 박는 순서가 곧 시각적 우선순위:

```
1. 꼬리 지우기 ('.')   ← 가장 먼저
2. 사과 박기 ('*')
3. 몸 박기 ('o')        ← body[1..length-1]
4. 머리 박기 ('O')      ← 가장 마지막 = 최상위
```

머리는 항상 보여야 하니 마지막. 사과와 몸은 rejection sampling으로 안 겹치게 보장되지만, **혹시 그 보장이 깨지면 어떤 증상이 나오는지**도 순서로 정할 수 있다 — "사과를 먼저, 몸을 나중"으로 두면 만약 겹치면 **사과가 안 보이는** 증상이 나옴(몸이 사과를 덮음). 디버그 친화적인 방향.

### 몸 렌더링은 단순 순회

```c
for (int i = 1; i < length; i++) {
  board[body[i].r][body[i].c] = 'o';
}
```

`i = 0`은 머리니까 별도 처리, `i = 1`부터가 몸. 각 segment의 `(r, c)`에 `'o'` 한 글자.

> "큐의 가시화"가 처음으로 일어나는 순간. Stage 4-(1)부터 `body[]`에 좌표를 쌓아뒀지만, 이걸 화면에 그려본 적은 없었다.

### 사과가 몸 위에 떨어지면 안 되는 이유

겹치면 그 칸의 글자는 그리는 순서에 따라 `o` 또는 `*` — 한 글자만 표시됨. 사용자 입장에선 "사과를 먹으려고 갔는데 사과가 안 보이거나, 몸이 안 보이거나" 가 됨. 게다가 사과 위로 몸이 지나가면 **사과를 우연히 못 먹게** 되어 게임 진행 자체가 막힘.

해결: 사과를 박기 전에 "그 좌표에 이미 뱀이 있나" 검사. 있으면 **다시 뽑는다**. 만족할 때까지 반복.

### Rejection sampling이란

"임의로 뽑고, 조건 안 맞으면 버리고, 다시 뽑기" 라는 일반 패턴. 통계·시뮬레이션·게임에서 다 쓴다:

- **랜덤 맵 생성**: 보스 방을 뽑았는데 시작 방과 너무 가까우면 다시.
- **카드 셔플**: 마지막 뽑은 카드와 같으면 다시(연속 방지).
- **확률 변환**: 균등 분포 난수에서 정규 분포 뽑기(전형적 알고리즘 중 하나).

장점: **알고리즘이 단순**. 단점: 조건 통과 확률이 낮으면 평균 시도 횟수가 늘어남 (극단적으론 무한 루프).

스네이크에선 보드가 거의 가득 차면 빈 칸 찾기가 점점 어려워지지만, **학습 단계에선 length가 그렇게 커지지 않으니** 문제없음. (이론적 함정은 함정 절에 한 줄.)

### `do-while` — "일단 한 번은 하고 본다"

```c
do {
  /* 본문 */
} while (조건);
```

`while`과 비슷하지만 **조건 검사 전에 본문을 무조건 한 번 실행**한다. 사과 뽑기처럼 "일단 한 번 뽑아보고, 조건 안 맞으면 다시"라는 패턴에 자연스럽게 맞음.

비교:
```c
// while 버전 — 첫 시도 전에 조건이 어떻게 평가될지 애매
collide = 1;
while (collide) {
  /* 뽑기 + 검사 */
}

// do-while 버전 — 첫 뽑기는 무조건 실행, 그 결과로 조건 결정
do {
  /* 뽑기 + 검사 */
} while (collide);
```

후자가 의도를 더 명확히 드러냄. C에서 흔히 보는 관용.

### 코드 중복은 일단 수용

첫 사과 배치와 먹은 후 새 사과 배치 — **같은 `do-while` 블록이 두 번** 들어간다. 깔끔하진 않다. 정석은 함수로 빼는 것:

```c
void place_apple(Cell *apple, Cell *body, int length) { /* ... */ }
```

이번 stage는 함수 도입을 안 한다(stage 5에서 다룸). 일단 인라인으로 두 번 쓰고, **"여기 함수가 들어갈 자리"** 라는 감각만 가져간다. **중복을 보면 함수가 들어갈 자리가 보이기 시작** 한다는 게 다음 단계 학습 포인트.

---

## 새 CS 개념

- **Rejection sampling**: 조건 만족할 때까지 다시 뽑는 일반 샘플링 패턴. 단순함과 시도 횟수 사이의 트레이드오프.
- **렌더링 우선순위 (z-order)**: 같은 좌표에 그릴 게 여러 개일 때 "나중에 그리는 게 위"라는 규칙. 게임/GUI 어디서나 같음.

---

## 새 C 문법

### `do { ... } while (조건);`
- 본문을 **최소 한 번 실행** 후 조건 평가. 거짓이 될 때까지 반복.
- 끝의 세미콜론 `;`을 빼먹기 쉽다 (`while` 단독엔 없으니까). 주의.

### `break` (이미 본 것의 새 용도)
- `switch`뿐 아니라 **반복문 한 단계를 즉시 빠져나갈 때도** 쓴다.
- 사과/몸 충돌 검사에서 "겹치는 segment를 찾았으면 더 볼 필요 없음" 표시.

---

## 코드

`01-snake/snake.c`를 다음으로 덮어쓴다.

```c
#include <windows.h>
#include <stdio.h>
#include <conio.h>
#include <stdlib.h>
#include <time.h>

#define ROWS 8
#define COLS 18

typedef struct {
  int r;
  int c;
} Cell;

#define MAX_LEN (ROWS * COLS)

int main(void) {
  SetConsoleOutputCP(65001);

  srand(time(NULL));

  char board[ROWS][COLS];

  int dr = 0, dc = 1;

  for (int r = 0; r < ROWS; r++) {
    for (int c = 0; c < COLS; c++) {
      if (r == 0 || r == ROWS - 1 || c == 0 || c == COLS - 1) {
        board[r][c] = '#';
      } else {
        board[r][c] = '.';
      }
    }
  }

  Cell *body = malloc(sizeof(Cell) * MAX_LEN);
  int length = 1;

  body[0].r = ROWS / 2;
  body[0].c = COLS / 2;

  Cell prev_tail = body[0];

  Cell apple;
  // (수정) 첫 사과 배치도 몸과 안 겹치게 — rejection sampling.
  // 시작 시 length = 1이라 body[0](머리)만 검사하면 됨.
  // 정의 좌표를 뽑고 → 머리와 겹치는지 확인 → 겹치면 다시.
  // apple.r = 1 + rand() % (ROWS - 2);
  // apple.c = 1 + rand() % (COLS - 2);
  int apple_collide;
  do {
    apple.r = 1 + rand() % (ROWS - 2);
    apple.c = 1 + rand() % (COLS - 2);
    apple_collide = 0;
    for (int i = 0; i < length; i++) {
      if (body[i].r == apple.r && body[i].c == apple.c) {
        apple_collide = 1;
        break;
      }
    }
  } while (apple_collide);

  int ate = 0;

  int running = 1;

  printf("\033[?1049h");

  while (running) {
    printf("\033[H");

    // === 차분 렌더링 ===
    // 그리는 순서가 곧 우선순위: 꼬리 지우기 → 사과 → 몸 → 머리.
    if (!ate) {
      board[prev_tail.r][prev_tail.c] = '.';
    }
    ate = 0;

    board[apple.r][apple.c] = '*';

    // (추가) 몸 그리기 — body[1..length-1]을 'o'로.
    // i=0은 머리(별도, 마지막에 'O'). i=1부터가 몸 segment.
    for (int i = 1; i < length; i++) {
      board[body[i].r][body[i].c] = 'o';
    }

    board[body[0].r][body[0].c] = 'O';

    for (int r = 0; r < ROWS; r++) {
      for (int c = 0; c < COLS; c++) {
        putchar(board[r][c]);
      }
      putchar('\n');
    }

    printf("이동: wasd, 종료: q\n");

    if (_kbhit()) {
      int key = _getch();
      switch (key) {
        case 'w': dr = -1, dc =  0; break;
        case 's': dr =  1, dc =  0; break;
        case 'a': dr =  0, dc = -1; break;
        case 'd': dr =  0, dc =  1; break;
        case 'q': running = 0; break;
      }
    }

    prev_tail = body[length - 1];

    for (int i = length - 1; i > 0; i--) {
      body[i] = body[i - 1];
    }

    body[0].r += dr;
    body[0].c += dc;

    if (body[0].r < 1)        body[0].r = 1;
    if (body[0].r > ROWS - 2) body[0].r = ROWS - 2;
    if (body[0].c < 1)        body[0].c = 1;
    if (body[0].c > COLS - 2) body[0].c = COLS - 2;

    if (body[0].r == apple.r && body[0].c == apple.c) {
      length++;
      body[length - 1] = prev_tail;

      // (수정) 새 사과 배치도 머리·몸과 안 겹치게 — 같은 do-while.
      // 같은 패턴이 두 번 나옴 → 다음 stage에서 함수로 뺄 자리.
      // apple.r = 1 + rand() % (ROWS - 2);
      // apple.c = 1 + rand() % (COLS - 2);
      do {
        apple.r = 1 + rand() % (ROWS - 2);
        apple.c = 1 + rand() % (COLS - 2);
        apple_collide = 0;
        for (int i = 0; i < length; i++) {
          if (body[i].r == apple.r && body[i].c == apple.c) {
            apple_collide = 1;
            break;
          }
        }
      } while (apple_collide);

      ate = 1;
    }

    Sleep(200);
  }

  free(body);

  printf("\033[?1049l");

  return 0;
}
```

---

## 돌려보기

```sh
gcc 01-snake/snake.c -o 01-snake/snake
./01-snake/snake
```

확인 사항:
- 사과를 먹으면 뱀이 **눈에 보이게** 한 칸씩 길어지는가?
- 머리는 `O`, 몸 segments는 `o`로 구분되는가?
- 새 사과가 **몸 위에 떨어지지 않는가**? (눈으로 확인하기 어려우니, 몸을 길게 키운 다음 사과가 빈 칸에만 뜨는지 보기)
- 벽 clamp, `q` 종료 정상 작동?

길어진 뱀이 자기 몸을 통과하는 게 보일 텐데 — 그게 다음 stage의 일감 (자기 몸 충돌 = 게임오버).

---

## 흔히 빠지는 함정

- **몸 그리는 루프 `i = 0`부터** → body[0](머리) 자리에 `o`가 박힌 뒤 마지막 머리 `O` 덮어쓰기로 결과는 같지만 의도가 흐려짐. `i = 1`부터가 정석.
- **사과를 몸보다 **뒤**에 그림** → 몸과 겹치는 자리에 사과가 떨어졌을 때(혹시 rejection이 깨졌다면) **사과만 보이고 몸 한 칸이 사라짐**. 디버그가 더 어렵다.
- **`do-while`의 끝 세미콜론 빠뜨림** → 컴파일 에러. `}` 다음 `while (...)` 다음 `;`까지가 한 문장.
- **`break;`를 빼먹음** → 충돌하는 segment를 찾고도 루프를 끝까지 돔. 동작은 똑같이 되지만(`apple_collide`가 한 번 1로 바뀌면 다시 0이 안 됨) 느림. 정석은 찾자마자 `break`.
- **첫 사과는 그냥 한 번만 뽑음** → length=1이라 충돌 확률 1%. 가끔 머리 위에 사과가 떠서 "왜 안 먹어도 사과가 사라지지?" 같은 신비 현상. 일관성을 위해 같은 do-while.
- **MAX_LEN까지 자라면 새 사과 do-while이 무한 루프** → 빈 칸이 0개. 이론상 게임 클리어 상태. 처리는 stage 5에서 (지금은 게임을 그렇게까지 길게 안 함).
- **자기 몸을 통과해도 멀쩡함** → 정상. 자기 충돌 판정과 게임오버는 stage 5에서.

---

## 체크 질문

소리 내어 답해보기:

1. 박는 순서를 "꼬리 지우기 → 머리 → 몸 → 사과"로 바꾸면 화면이 어떻게 망가질까? 어느 글자가 안 보이지?
2. Rejection sampling의 장점과 한계를 한 문장으로? 어떤 경우에 시간이 폭증할까?
3. 몸 그리는 루프를 `for (int i = 0; i < length; i++)`로 바꾸면 시각적으로 뭐가 달라지지? 왜?
4. `do-while`과 `while`의 차이를 사과 뽑기 예시로 설명한다면?
5. 첫 사과 do-while과 새 사과 do-while이 거의 같다는 사실은 무엇을 시사하는가? (다음 stage 예고)
6. 사과 충돌 검사의 `break;`를 빼면 동작은 같은데 왜 비효율적인가?
7. length가 MAX_LEN까지 가면 새 사과 do-while에 무슨 일이 일어나지?

---

## 다음 단계 예고

[Stage 5](stage-5.md) — **충돌·게임오버·점수.** 머리가 자기 몸과 부딪히면 게임 종료. `length - 1`을 점수로 표시. 그리고 이번 단계에서 두 번 등장한 사과 배치 코드를 **함수로 분리** — 처음으로 `main` 외부에 함수를 만든다. 게임 상태 전이("플레이 중" → "게임 오버")의 개념도.
