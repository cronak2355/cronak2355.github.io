---
layout: post
title: "Linked List - Q1"
date: 2025-10-16 02:00:58 +0900
categories: [KRAFTON JUNGLE]
tags: [Programming]
---
### 정수를 입력받아, 해당 값을 오름차순으로 정렬된 연결 리스트에 삽입하는 C 함수를 작성하는 문제
#### 문제
1. 함수는 새 항목이 삽입된 인덱스 위치를 반환한다.
2. 삽입이 실패한 경우(중복된 값 등)에는 -1을 반환해야 한다.
3. 연결 리스트는 이미 정렬되어 있거나 비어 있다고 가정한다.

#### 접근
값을 하나하나 비교하며 중복 값과 정렬되어야 할 위치를 찾는다.

### 트러블 슈팅
#### 반복되는 값을 잡아내지 못하는 문제
#### 해결 : temp라는 노드에서 확인하게 하여 해결함

#### 마지막 값을 넣지 못하고 인덱스만 증가하는 문제
#### 해결 : 노드의 다음 값이 NULL인 상태로 끝났다 = 다음 값이 없으며 중복되거나 현재 값보다 더 큰 값이 없다. 고로 다음 값에 현재 값 삽입



```c
int insertSortedLL(LinkedList *ll, int item)
{
	int index = 0;
	ListNode *newnode = malloc(sizeof(ListNode)); //malloc을 통해 ListNode의 byte 크기 만큼 메모리 할당

	if(newnode == NULL) //메모리가 할당되지 않으면 -1 출력
	{ 
		return -1;
	}

	newnode->item = item; //입력 받은 수를 newnode 안에 있는 item에 넣음
	newnode->next = NULL; //다음 수는 아직 모르므로 NULL로 설정

	if(ll->head == NULL) { //List head안에 아무것도 없을 경우 newnode주소를 head에 넣은 후 List크기를 1로 설정해줌 index값은 0 반환
		ll->head = newnode;
		ll->size = 1;
		return 0;
	}
	ListNode *temp = ll->head; //List head(첫 번째 값)을 temp에 저장

	if(item < ll->head->item) { //만약 입력 받은 item값이 List의 head의 item 값보다 작을 경우 기존 head값을 newnode의 next로 바꾸고 head를 newnode로 설정 후 size에 +1
		newnode->next = ll->head;
		ll -> head = newnode;
		ll->size += 1;
		return 0;
	}

	while(temp -> next != NULL) { //next값이 NULL일 때까지 반복
		if(temp->item == item) { //만약 입력 받은 값이랑 중복되는 값이 있을 경우 -1반환
			return -1;
	}

		if(temp->next->item > newnode->item) { //첫 번째 값의 다음 item이 현재 itme보다 클 경우 만약 입력 받은 값이 기존 값보다 작을 경우 현재 값을 next로 넣고 입력 받은 값을 newnode로 넣음
			newnode-> next = temp -> next; //temp의 next를 newnode의 다음으로 설정 
			temp -> next = newnode; //temp의 next는 newnode로 설정 
			ll->size += 1;
			index += 1;
			return index;
		}
		temp = temp->next; //한칸씩 리스트를 이동 
		index+=1;
	}
	temp-> next = newnode; //다 돌아도 넣을 수가 없으면 마지막에 삽입
	ll->size += 1;
	index +=1;
	return index;
}
```

### 결과 
반례가 있음

### 교훈 
작성한 코드를 더 효율적으로 만들 궁리를 해야한다.




























