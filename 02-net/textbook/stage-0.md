# Stage 0 — 첫 소켓: 에코 서버

> 게임을 잠시 떠나, **포트를 열고 손님을 기다리는 프로그램**을 직접 짠다. nginx·sshd·MySQL이 전부 이 한 가지 골격의 변주였다 — 오늘 그 골격을 손으로 만든다.

## 들어가기 전 — WSL 준비 (한 번만)

지금까지는 Windows + MSYS2(gcc)로 했지만, **네트워크부터는 WSL(Windows 속 리눅스)에서** 한다. 이유는 ① 교재·자료(Beej's, CS50)가 전부 리눅스 소켓(POSIX) 기준이라 머릿속 번역이 필요 없고, ② 실제 운영 서버(리눅스)와 같은 환경이며, ③ 나중 Phase 3(미니셸)도 어차피 WSL이 필요하기 때문.

PowerShell에서 (관리자 권한):

```powershell
wsl --install
```

재부팅 후 Ubuntu가 뜨면 사용자명/비번 설정. 그 다음 **Ubuntu 터미널 안에서**:

```sh
sudo apt update
sudo apt install -y gcc netcat-openbsd
```

- `gcc` — 컴파일러 (MSYS2의 그것과 같은 역할, 리눅스판)
- `netcat-openbsd` — 오늘 서버를 테스트할 **클라이언트 도구**(`nc`). 우리가 클라이언트 코드를 짜기 전, 이걸로 먼저 접속해본다.

> 💡 윈도우 파일은 WSL에서 `/mnt/c/Users/...` 로 보인다. 이 프로젝트 폴더는 `cd /mnt/c/Users/bluec/Documents/01_YM/basic-cs` 로 접근. (단, 빌드/실행 속도는 WSL 홈 `~/` 안이 더 빠르다 — 신경 쓰이면 나중에 옮겨도 됨.)

---

## 오늘의 목표

- 9000번 포트에서 듣는 **에코 서버**를 만든다 (받은 문자를 그대로 되돌려줌).
- 다른 터미널에서 `nc localhost 9000` 으로 접속해, 내가 친 글자가 메아리쳐 돌아오는 걸 본다.
- 소켓 통신의 5형제 **`socket → bind → listen → accept → recv/send`** 를 손으로 친다.

---

## 배경 이야기 — "창구를 열고 기다린다"

지난 대화에서 정리한 그림을 코드로 옮기는 날이다.

서버 프로그램의 일생은 한 문장이었다: **"포트를 열고(listen), 손님이 올 때까지 잠들었다가(accept), 오면 깨어나 처리하고, 다시 잠든다."** 이걸 가능하게 하는 OS의 도구가 **소켓(socket)** 이다.

소켓은 **"네트워크로 드나드는 통신 창구"** 다. 우리가 `socket()`을 부르면 OS가 창구를 하나 내주고, 그 창구를 가리키는 번호표(=파일 디스크립터)를 돌려준다. 재미있는 건 — 이 번호표가 뱀게임 Stage 2에서 만난 `stdin`(0)/`stdout`(1)/`stderr`(2)와 **같은 종류**라는 점이다. 유닉스는 키보드·화면·파일·네트워크 연결을 전부 "파일처럼 생긴 것(file descriptor)"으로 통일해서 다룬다. 그래서 소켓에도 `read`/`write`를 쓸 수 있다(우리는 네트워크용 별칭 `recv`/`send`를 쓸 거지만 본질은 같다).

순서를 비유로:

| 코드 | 비유 (가게 차리기) |
|---|---|
| `socket()` | 창구(전화기) 한 대 마련 |
| `bind()` | 그 전화기에 **번호(포트 9000)** 배정 |
| `listen()` | 영업 개시 — "전화 받을게" 상태로 |
| `accept()` | **전화벨 울릴 때까지 대기** → 울리면 통화 연결 |
| `recv`/`send` | 통화 (듣고 / 말하고) |
| `close()` | 통화 끊기 |

`accept()`가 바로 "잠들어 기다리는" 그 지점이다. 손님이 없으면 여기서 멈춰 CPU를 쓰지 않고, 접속이 오면 OS가 깨운다.

## 새 CS 개념

### 클라이언트–서버 모델
한쪽(**서버**)은 정해진 포트에서 **계속 기다리고**, 다른 쪽(**클라이언트**)이 먼저 **말을 건다**. 비대칭이다. 오늘 우리는 서버를 만들고, 클라이언트는 `nc`로 대신한다. (다음 단계부터 클라이언트도 직접 짠다.)

### TCP — "끊김 없는 바이트 수도관"
우리가 만드는 건 **TCP** 연결이다. TCP는 접속 시 양쪽이 짧은 인사(3-way handshake)를 주고받아 **신뢰성 있는 연결**을 만든다 — 보낸 순서대로, 빠짐없이 도착하는 걸 OS가 보장한다. 그래서 우리는 "패킷이 유실되면 어쩌지" 같은 걱정 없이 **바이트가 흐르는 수도관** 하나가 깔렸다고 보면 된다. (`SOCK_STREAM`이 "이 수도관 방식으로 줘"라는 뜻.)

### 바이트 순서(endianness)와 `htons`
숫자를 메모리에 저장하는 순서가 CPU마다 다르다(리틀 엔디언/빅 엔디언). 그런데 네트워크는 **약속된 한 가지 순서(빅 엔디언)** 만 쓴다. 그래서 포트 번호 같은 숫자를 네트워크에 넣기 전 `htons()`(**h**ost **to** **n**etwork **s**hort)로 변환해야 한다. 안 하면 9000이 엉뚱한 포트로 둔갑한다. → "내 컴퓨터 안의 표현"과 "밖으로 나가는 표준 표현"이 다를 수 있다는, 네트워크의 첫 교훈.

### 블로킹(다시 만남)
`accept()`와 `recv()`는 **블로킹** 함수다 — 일이 생길 때까지 멈춘다. Stage 3에서 `getchar()`(블로킹) vs `_kbhit()`(논블로킹)을 겪었던 그 개념이 그대로 돌아온다. 오늘은 블로킹으로 단순하게 가고, 캡스톤(네트워크 뱀게임)에서 이걸 논블로킹으로 바꿔 게임 루프에 녹인다.

---

## 새 C 문법

### `#include` 들 (네트워크용)
```c
#include <sys/socket.h>  // socket, bind, listen, accept, recv, send
#include <arpa/inet.h>   // struct sockaddr_in, htons, inet_ntoa, INADDR_ANY
#include <unistd.h>      // close (소켓도 fd라 close로 닫음)
```
Windows의 `<winsock2.h>`에 해당하는, 리눅스의 소켓 헤더들. `windows.h`/`conio.h`는 더 안 쓴다(WSL이니까).

### `int fd = socket(AF_INET, SOCK_STREAM, 0);`
- 소켓을 만들고 **파일 디스크립터(정수)** 를 받는다. 실패 시 `-1`.
- `AF_INET` = IPv4 주소 체계, `SOCK_STREAM` = TCP. (`SOCK_DGRAM`이면 UDP)

### `struct sockaddr_in`
- 주소(IP + 포트)를 담는 구조체. 필드를 직접 채운다.
  - `.sin_family = AF_INET;`
  - `.sin_addr.s_addr = INADDR_ANY;` — "이 컴퓨터의 모든 네트워크 카드에서 받기"(= 0.0.0.0)
  - `.sin_port = htons(9000);` — 포트, **반드시 `htons`로 감싸서**

### `bind(fd, (struct sockaddr*)&addr, sizeof(addr))`
- 소켓에 위 주소(포트)를 묶는다. `(struct sockaddr*)` 캐스팅은 옛 API 관례(원형이 일반 `sockaddr`을 받게 돼 있어서).

### `listen(fd, backlog)`
- "접속 받을게" 상태로 전환. `backlog`(예: 5)는 한꺼번에 몰린 대기 접속을 몇 개까지 줄 세울지.

### `int client = accept(fd, ...)`
- 접속 하나를 받아 **새 소켓(클라이언트 전용 fd)** 을 돌려준다. 이후 그 손님과의 통신은 이 `client` fd로 한다. (원래 `fd`는 계속 다음 손님을 받는 "안내데스크"로 남는다.)

### `recv(client, buf, len, 0)` / `send(client, buf, len, 0)`
- 받기 / 보내기. 반환값은 **실제 처리된 바이트 수**.
- `recv`가 **0**을 반환하면 = 상대가 연결을 끊음. **음수**면 에러.

### `setsockopt(... SO_REUSEADDR ...)`
- (편의) 서버를 껐다 바로 다시 켤 때 나는 "Address already in use" 에러를 막아준다. 지금은 "재시작 편하게 해주는 한 줄"로만 알면 충분.

### `perror("...")`
- 직전 시스템 콜이 왜 실패했는지를 사람이 읽는 메시지로 출력. 네트워크 코드는 실패 지점이 많아 **단계마다 에러 체크**가 습관이 돼야 한다.

---

## 코드

`02-net/echo.c` 에 그대로 친다.

```c
#include <stdio.h>
#include <string.h>      // memset
#include <unistd.h>      // close
#include <arpa/inet.h>   // sockaddr_in, htons, inet_ntoa, INADDR_ANY
#include <sys/socket.h>  // socket, bind, listen, accept, recv, send

#define PORT 9000

int main(void) {
  // 1) 소켓(통신 창구) 생성. AF_INET=IPv4, SOCK_STREAM=TCP.
  //    돌아오는 정수는 stdin/stdout과 같은 종류의 "파일 디스크립터".
  int server_fd = socket(AF_INET, SOCK_STREAM, 0);
  if (server_fd < 0) { perror("socket"); return 1; }

  // (편의) 재시작 시 "Address already in use" 방지. 지금은 관용구로 받아들인다.
  int opt = 1;
  setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

  // 2) 주소 구조체 채우기 — "어느 포트에서 받을지"
  struct sockaddr_in addr;
  memset(&addr, 0, sizeof(addr));        // 쓰레기값 방지: 통째 0으로 초기화
  addr.sin_family = AF_INET;             // IPv4
  addr.sin_addr.s_addr = INADDR_ANY;     // 모든 네트워크 인터페이스(0.0.0.0)
  addr.sin_port = htons(PORT);           // 호스트→네트워크 바이트순서 변환 (필수!)

  // 3) 소켓에 포트 묶기
  if (bind(server_fd, (struct sockaddr*)&addr, sizeof(addr)) < 0) {
    perror("bind"); return 1;
  }

  // 4) 듣기 시작. 5 = 대기 접속 큐 길이.
  if (listen(server_fd, 5) < 0) { perror("listen"); return 1; }
  printf("에코 서버 대기 중... 포트 %d (Ctrl+C로 종료)\n", PORT);

  // 바깥 루프: 손님을 한 명씩 받아 처리하고, 끝나면 다음 손님을 받는다.
  while (1) {
    struct sockaddr_in client;
    socklen_t clen = sizeof(client);

    // 5) 접속 받기. 손님 없으면 여기서 "잠들어" 대기(블로킹).
    int client_fd = accept(server_fd, (struct sockaddr*)&client, &clen);
    if (client_fd < 0) { perror("accept"); continue; }
    printf("접속됨: %s\n", inet_ntoa(client.sin_addr));  // 손님 IP 출력

    // 안쪽 루프: 이 손님이 끊을 때까지 받은 걸 그대로 되돌려준다.
    char buf[1024];
    while (1) {
      int n = recv(client_fd, buf, sizeof(buf), 0);  // 6) 받기
      if (n <= 0) break;            // 0=상대가 끊음, 음수=에러 → 이 손님 종료
      send(client_fd, buf, n, 0);   //    받은 바이트 수(n)만큼 그대로 돌려주기
    }

    // 7) 이 손님과의 통신 종료. 안내데스크(server_fd)는 살아있다.
    close(client_fd);
    printf("연결 종료. 다음 손님 대기...\n");
  }

  close(server_fd);  // (실제로는 Ctrl+C로 끝나서 여기 도달 안 함)
  return 0;
}
```

**왜 두 겹 루프인가?** 바깥 루프 = "손님을 계속 받는다(서버는 안 죽는다)". 안쪽 루프 = "한 손님과 대화가 끝날 때까지 주고받는다". 이 구조가 모든 서버의 기본형이다. 지금은 한 번에 한 손님만(앞 손님이 끊어야 다음 손님) 받지만, 동시 처리는 나중 단계에서.

---

## 돌려보기

**터미널 두 개**가 필요하다 (서버용 / 클라이언트용). 둘 다 WSL(Ubuntu).

**터미널 A — 서버 실행:**
```sh
cd /mnt/c/Users/bluec/Documents/01_YM/basic-cs
gcc 02-net/echo.c -o 02-net/echo
./02-net/echo
```
기대 출력:
```
에코 서버 대기 중... 포트 9000 (Ctrl+C로 종료)
```
(여기서 멈춘 듯 보이면 정상 — `accept`에서 손님을 기다리는 중.)

**터미널 B — 클라이언트로 접속:**
```sh
nc localhost 9000
```
이제 아무 글자나 치고 Enter:
```
hello
hello          ← 서버가 그대로 메아리침!
안녕
안녕
```
→ 터미널 A에는 `접속됨: 127.0.0.1` 이 떠 있을 것이다.

**끝내기:** 터미널 B에서 `Ctrl+C`(또는 `Ctrl+D`)로 접속을 끊으면, 터미널 A에 `연결 종료. 다음 손님 대기...`가 뜨고 다시 대기 상태로. 서버 자체를 끄려면 터미널 A에서 `Ctrl+C`.

> 🎉 방금 **당신이 짠 C 프로그램이 포트를 열고, 접속을 받고, 데이터를 주고받았다.** nginx가 하는 일의 가장 작은 핵이 바로 이것이다.

---

## 흔히 빠지는 함정

- **`bind: Address already in use`** — 방금 끈 서버의 포트가 잠깐 남아 있을 때. 코드의 `SO_REUSEADDR` 한 줄이 대부분 막아준다. 그래도 나면 몇 초 기다리거나 포트 번호(9000)를 바꿔본다.
- **`htons`를 빼먹음** — 포트가 엉뚱하게 잡혀 `nc localhost 9000`이 접속 안 됨. 숫자를 네트워크에 넣을 땐 항상 `htons`.
- **`recv` 반환값을 안 봄** — `recv`가 0(끊김)을 줬는데 계속 루프 돌면 무한 반복/오작동. `if (n <= 0) break;` 필수.
- **버퍼를 문자열로 착각** — `recv`가 채운 `buf`는 끝에 `\0`이 없을 수 있다. 그대로 `printf("%s", buf)` 하면 위험. 우리는 "받은 n바이트만 그대로 send"라 안전하지만, 화면에 글자로 찍고 싶으면 `buf[n] = '\0';`을 먼저 해야 한다(버퍼 크기 주의).
- **터미널 하나로 테스트** — 서버가 `accept`에서 멈춰 있어 같은 터미널에선 `nc`를 못 친다. **반드시 터미널 두 개.**
- **방화벽/포트 점유** — 9000이 이미 다른 프로그램이 쓰는 중이면 bind 실패. 포트를 9001 등으로 바꾼다.

---

## 체크 질문

소리 내어 답해보기:

1. `socket → bind → listen → accept` 네 단계가 각각 "가게 차리기" 비유에서 무엇에 해당하지?
2. `accept()`는 왜 **새 소켓(`client_fd`)** 을 돌려주지? 원래 `server_fd`는 그동안 무슨 역할을 하지?
3. 포트 번호에 왜 `htons()`를 씌워야 하지? 안 씌우면 무슨 일이?
4. `recv()`가 `0`을 반환하면 무슨 뜻이고, 그때 우리 코드는 어떻게 행동하지?
5. 소켓이 `stdin`/`stdout`과 "같은 종류"라는 건 무슨 의미지? (힌트: 파일 디스크립터)
6. 지금 서버는 손님을 한 번에 한 명만 받는다. 왜 그런지 코드의 루프 구조로 설명해보자.

---

## 다음 단계 예고

[Stage 1](stage-1.md) — **숫자 맞히기 게임 (서버가 정답, 클라가 추측).** 단순 메아리를 넘어, "클라가 숫자를 보내면 서버가 위/아래/정답으로 답한다"는 **우리만의 대화 규칙(프로토콜)** 을 처음 설계한다. 그리고 드디어 **클라이언트도 직접 C로** 짠다 (`nc` 졸업).
