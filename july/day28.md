# 헌마 교육 중 중요내용 정리

## RIP에 대하여
### RIP : CPU가 다음에 실행할 명령어의 주소를 가리키는 레지스터 / x86-64에서 사용하는 어셈블리어

RIP = 0x401100이라면 cpu는 0x401100에 있는 기계어 명령을 실행한다

예를 들어서
main();
vuln();
puts("DONE");
일 때 main()이 vuln()dmf 호출할 때, vuln()실행이 끝난 후 돌아올 주소를 stack에 저장한다. 이게 savedRIP, 즉 반환주소이다.
  ┌────────────────────┐
  │ saved RIP          │ → vuln 종료 후 실행할 주소
  ├────────────────────┤
  │ saved RBP          │
  ├────────────────────┤
  │ char buf[64]       │
  └────────────────────┘
정상적인 경우는
savedRIP = main함수의 다음 명렁어 주소
공격 측면에서 바라보면
savedRIP = win() {bin/sh/(공격 바이너리, 함수)} 로 이동한다.
함수의 ret실행 시  cpu는 win()으로 이동

프로그램에 
void win() {
  system("/bin/sh");
}
가 있다고 가정할 때,
win()주소가 0x401169라면 payload를 
\x90이나 A X savedRIP까지의 거리0x401169인데
ret이 실행되면 
RIP = 0x401169가 되어 win()이 실행된다.
---

공격자가 일반적으로 직접 현재 RIP에 값을 쓰는 것은 아닙니다. 먼저 Stack에 저장된 saved RIP를 덮는다.

  Buffer Overflow로 saved RIP 변경
                ↓
  함수에서 ret 실행 -- ret실행이 무슨 말?
                ↓
  saved RIP가 실제 RIP로 들어감
                ↓
  실행 흐름 변경

  따라서 RIP가 중요한 이유는 다음과 같습니다.

--> RIP를 제어한다는 것은 CPU가 다음에 어떤 코드를 실행할지 제어한다는 의미이기 때문.

## Shellcode
shellcode는 공격자가 실행하려는 기계어 코드

흔히 아는 기능:
- /bin/sh 실행
- 파일 열기 및 읽기
- 네트워크 연결
- Reverse shell 연결
등이 있다.

예를 들어서 C코드로 기능 구현을 해보면
execve("/bin/sh", ...);

이를 CPU가 직접 실행할 수 있는 기계어 바이트로 만들면 다음과 같은 형태가 됩니다.

shellcode = b"\x48\x31\xf6\x56\x48..."

문자열처럼 보이지만 실제로는 CPU 명령어이다.

연장선으로
### Stack shellcode
Stack shellcode는 기계어 코드를 Stack의 버퍼에 넣고 그 주소로 실행 흐름을 보내는 방식
- Stack
  ┌──────────────────────────┐
  │ saved RIP = buf 주소     │
  ├──────────────────────────┤
  │ saved RBP                │
  ├──────────────────────────┤
  │ shellcode가 들어간 buf   │
  └──────────────────────────┘

payload작성 구조
-  [shellcode][padding][buf 주소]
실행 과정 :
1) 입력을 통해 buf에 shellcode저장
2) Overflow로 saved RIP를 buf 주소로 변경
3) 함수에서 ret 실행
4) RIP가 buf 주소로 변경
5) CPU가 buf의 바이트를 명령어로 실행
6) /bin/sh 실행

### NX와 관게


  
