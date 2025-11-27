---
layout: post
title: "2025-11-25(pintos)"
date: 2025-11-26 00:56:45 +0900
categories: [KRAFTON JUNGLE]
tags: [Programming]
---
---

# Today's Log(2025-11-25)
>오늘은 Week11의 화요일이다. 이번 주차부터 pintos 프로젝트에 들어간다.

---

# 오늘 한 일
- argument passing 파싱 구현


# TIL

```c
char file_name_copy[256];     // 파싱용 복사본 버퍼
char *argv[64];               // 최대 64개 인자 지원 (Pintos 권장)
int argc = 0;
char *token, *save_ptr;

/* 1. 원본 보호를 위한 안전 복사 */
strlcpy(file_name_copy, file_name, sizeof(file_name_copy));

/* 2. 공백 기준 토큰화하면서 argv에 저장 */
for (token = strtok_r(file_name_copy, " ", &save_ptr);
     token != NULL;
     token = strtok_r(NULL, " ", &save_ptr)) {
    argv[argc++] = token;     // token 주소 그대로 저장
}

/* 3. C 표준 준수를 위한 NULL 종료 – 반드시 필요! */
argv[argc] = NULL;
코드 동작 상세 분석
``` 

| 단계 | 코드 | 핵심 동작 및 이유 |
| - | -| - |
1 | strlcpy(...) | 원본 file_name 절대 건드리지 않음. 항상 \0 종료 보장 → 안전
2 | strtok_r(file_name_copy, ...) | 복사본만 파괴 (\0 삽입). save_ptr로 재진입 안전하게 상태 유지
3 | argv[argc++] = token; | 별도 strdup 없이 복사본 내부 주소 재사용 → 메모리 효율 최고
4 | argv[argc] = NULL; | execve 계열 함수와 C 언어 표준이 요구하는 종료 조건
실행 예시
text입력 file_name : "/bin/echo hello world 123"

파싱 후 file_name_copy 내부:
"/bin/echo\0hello\0world\0123\0..."

최종 argv 상태:
argv[0] → "/bin/echo"
argv[1] → "hello"
argv[2] → "world"
argv[3] → "123"
argv[4] → NULL
argc = 4

---



# 코드 / 실습
```c
    strlcpy(file_name_copy, file_name, 255);
    for (token = strtok_r (file_name_copy, " ", &save_ptr); token != NULL; token = strtok_r (NULL, " ", &save_ptr)) {
		argv[argc] = token;
		argc++;
    	printf ("'%s'\n", token);
    }
```
