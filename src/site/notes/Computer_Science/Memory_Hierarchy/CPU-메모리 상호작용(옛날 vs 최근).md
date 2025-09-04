---
{"dg-publish":true,"permalink":"/Computer_Science/Memory_Hierarchy/CPU-메모리 상호작용(옛날 vs 최근)/","noteIcon":"","created":"2025-08-13T09:28:02.761+09:00","updated":"2025-08-18T01:04:50.156+09:00"}
---




![Pasted image 20250813190135.png](/img/user/supporter/image/Pasted%20image%2020250813190135.png)

- 최근 CPU는 **메모리 컨트롤러를 CPU 내부에 통합**  ➡ 메모리와 직접 연결



**옛날 방식**
```mermaid
flowchart LR 
CPU --> bridge[I/O Bridge] --> RAM
RAM --> bridge --> CPU
```

**최근 방식**
```mermaid
flowchart LR 
CPU+Memory_Controller --> RAM
RAM --> CPU+Memory_Controller
```
- CPU 내부에 메모리 컨트롤러를 통합 시킴 ➡ **메모리 접근 속도 향상** 
- I/O 연결 책임은 칩셋이 책임짐




