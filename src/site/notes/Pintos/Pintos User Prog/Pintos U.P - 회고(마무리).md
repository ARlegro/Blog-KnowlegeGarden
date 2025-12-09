---
{"dg-publish":true,"permalink":"/Pintos/Pintos User Prog/Pintos U.P - 회고(마무리)/","noteIcon":"","created":"2025-12-03T14:52:53.019+09:00","updated":"2025-12-03T16:55:40.696+09:00"}
---




## 1.  느낀점 

### 1.1.  시작, 인자 전달 
Project 1에서 스레드·락 같은 개념을 만졌을 때는 그냥 “아 운영체제 느낌이 이렇군” 정도였다. 근데 User Program을 시작하자마자 느낌이 확 달라졌다.

Argument Passing을 처음 구현할 때는 “도대체 왜 문자열을 역순으로 박고, 왜 word-align을 맞추고, 왜 fake return address를 넣어야 되지?” 이런 생각이 계속 들었다.

근데 스택을 한 줄씩 따라가고, System V ABI 문서를 눈으로 훑고, 디버그로 rsp 변화를 보면서 **‘아 이게 이렇게 돼야 main(argc, argv)이 살아서 실행되는구나’** 이걸 온몸으로 이해하게 됐다. 

### 1.2.  대충 넘어가면 터진다.
> OS라는 민감한 자식 때문에 얼마나 의미 있게 코드를 짜야될지 느겼다.

조금만 틀려도 절대 눈감아 주지 않는다. C라는 불안정한 언어로 OS를 구현하면서 한칸만 틀려도 Page Fault, 잠깐만 예외 처리 안해도 커널이 무너지고, 0xCCCC…로 뒤덮이고, RIP이 엉뚱한 곳을 가리키고, push에서 죽어버리는 등 너무 너무 오류가 많이 떴다.
덕분에 “운영체제가 얼마나 정교한 구조인지”는 걸 몸으로 배웠다.


### 1.3.  Multi OOM 
[[Pintos/Pintos User Prog/Multi-OOM 테스트 코드 심층 분석 및 약간 수정\|Multi-OOM 테스트 코드 심층 분석 및 약간 수정]]
[[Pintos/Pintos User Prog/Multi-OOM 테스트 Trouble(1) - 정상 흐름 속 page fault + 레지스터 오염\|Multi-OOM 테스트 Trouble(1) - 정상 흐름 속 page fault + 레지스터 오염]]
[[Pintos/Pintos User Prog/Multi-OOM 테스트 Trouble(2) - sema 교체\|Multi-OOM 테스트 Trouble(2) - sema 교체]]


Multi-OOM은 개인적으로 User Program 전체 중 가장 난이도가 높았고, 결정적으로 최종 통과는 못 했다. 그래도 이 테스트를 붙잡고 있었던 시간이 오히려 제일 많이 배운 시간이었다.

너무 많은 오류들이 있었다. 일부만 얘기하자면 
- 자식이 죽으면서 남긴 세마포어나 락이 오염되어 page fault가 터지고
- 레지스터가 쓰레기 값(예: `0xCCCC…`)으로 오염되고
- 커널 컨텍스트에서 `page fault`가 나면서 그대로 패닉이 발생하고
- `fork` 동기화가 조금만 어긋나도 부모가 손자의 exit을 기다리거나, 서로 잘못된 세마포어를 사용해 교착되는 일까지 생겼다
  
특히 마지막 즈음에는 노트북에서는 계속 실패하고 데스크탑에서는 통과하는 기괴한 경험도 있었다. 환경 차이, 컨텍스트 스위칭 타이밍, 메모리 정렬… OS 세계에서는 이런 작은 변수 하나가 결과를 갈라놓을 수 있다는 걸 배운 순간이었다.

Multi-OOM은 최종적으로는 실패한 챌린지였지만, 원인을 잡아내는 과정이 은근히 중독적이었고 가장 크게 배운 테스트 였던 것 같다.

---
## 2.  자료 

### 2.1.  공부 및 트러블 슈팅 

- [[Pintos/Pintos User Prog/Pintos U.P - 사전 분석\|Pintos U.P - 사전 분석]]
- [[Pintos/Pintos User Prog/Pintos U.P - Argument Passing\|Pintos U.P - Argument Passing]]
- [[Pintos/Pintos User Prog/Pintos U.P - Exec 시스템 콜 구현 및 트러블 슈팅\|Pintos U.P - Exec 시스템 콜 구현 및 트러블 슈팅]]
- [[Pintos/Pintos User Prog/Pintos U.P - Fork 개념 정리 + 좀비 프로세스\|Pintos U.P - Fork 개념 정리 + 좀비 프로세스]]
- [[Pintos/Pintos User Prog/Pintos U.P - Fork 구현 흐름 정리 및 트러블 슈팅 정리\|Pintos U.P - Fork 구현 흐름 정리 및 트러블 슈팅 정리]]
- [[Pintos/Pintos User Prog/Pintos U.P - read_normal 테스트\|Pintos U.P - read_normal 테스트]]
- [[Pintos/Pintos User Prog/Pintos U.P - System Call\|Pintos U.P - System Call]]
- [[Pintos/Pintos User Prog/Multi-OOM 테스트 코드 심층 분석 및 약간 수정\|Multi-OOM 테스트 코드 심층 분석 및 약간 수정]]
- [[Pintos/Pintos User Prog/Multi-OOM 테스트 Trouble(1) - 정상 흐름 속 page fault + 레지스터 오염\|Multi-OOM 테스트 Trouble(1) - 정상 흐름 속 page fault + 레지스터 오염]]
- [[Pintos/Pintos User Prog/Multi-OOM 테스트 Trouble(2) - sema 교체\|Multi-OOM 테스트 Trouble(2) - sema 교체]]]

### 2.2.  번역본 or 외부자료 
- [[Pintos/Pintos User Prog/가이드 및 번역본/User Prog - System Call 가이드 번역본\|User Prog - System Call 가이드 번역본]]
- [[Pintos/Pintos User Prog/가이드 및 번역본/User Prog 가이드 번역본 (intro)\|User Prog 가이드 번역본 (intro)]]
- [[Pintos/Pintos User Prog/가이드 및 번역본/User Prog 인자 전달 가이드 번역본\|User Prog 인자 전달 가이드 번역본]]



