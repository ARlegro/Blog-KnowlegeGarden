---
{"dg-publish":true,"permalink":"/DevStudy/DB/Postgre's TYPE/Postgre's VARCHAR vs TEXT/","noteIcon":"","created":"2025-12-09T17:19:47.463+09:00","updated":"2025-12-13T09:26:31.510+09:00"}
---



 **character varying(n), varchar(n)**
 - 제한이 있는 가변길이 (최대 길이 n 까지)
 - varchar(10)에 'abc'가 들어가면 3바이트의 크기를 가짐
 - 길이 지정을 하지 않으면 (n을 명시하지 않으면) 문자 길이 제한 없음으로 간주

**text**
 - 제한없이 가변 길이

varchar, char는 db엔진에 올라가는 반면에 text는 하드에 데이터를 저장한다.
그래서 varchar, char는 데이터에 대한 링크를 가지고 있기 때문에 인덱스가 가능한 반면, text는 그렇지 않기 때문에 인덱스가 불가능 하다. 상황에 따라서 잘 쓰도록

