---
layout: post
title: "Red Black Tree - case3_LEFT and LEFT "
date: 2025-10-23 02:17:07 +0900
categories: [KRAFTON JUNGLE]
tags: [Programming]
---
## LL Case란?

연속된 빨간 노드가 왼쪽-왼쪽으로 발생한 경우 → 오른쪽 회전 + 색상 교환으로 해결
```
      GP(B)                    P(B)
      /   \                   /   \
    P(R)  uncle     =>      z(R)  GP(R)
    /                              \
  z(R)                            uncle
```
#### 접근 방법 
화살표를 바꾼 다는 생각 
Tree는 화살표 방향에 따라 부모와 자식이 나눠짐
5 -> 6 은 5가 부모 6이 자식이지만 5 <- 6 은 6이 부모 5가 자식임
이러한 원리를 통해 부모 자식간의 관계를 쉽게 바꿀 수 있고 Red Black Tree에서 말하는 회전도 쉽게 구현 할 수 있음.

#### 코드의 단점 : 
증조부 관련 케이스를 원래 1개로 해결 할 수 있는 것을 2개로 나눔

```C
node_t* case3_LL_Match(node_t* z, rbtree *t) { //테스트 케이스3 왼쪽 부모의 왼쪽 노드가 빨간일 경우 
    node_t * parent = z->parent;  // z의 부모를 parent로 지정
    node_t * grandparent = z->parent->parent;  // 할아버지 노드 추가
    
    grandparent->left = parent->right; // 부모의 오른쪽을 할아버지의 왼쪽으로 지정
    if (parent->right != t->nil) {//부모의 오른쪽이 비어있지 않을 경우
      parent->right->parent = grandparent; // 부모의 오른쪽 노드의 부모를 할아버지로 지정
    }
    parent->right = grandparent;  // 할아버지를 부모의 오른쪽으로
    parent->parent = grandparent->parent;  // 부모의 부모를 증조부모로 연결
    
    
    if (grandparent->parent != t->nil) { // 증조부모가 있다면
        if (grandparent->parent->left == grandparent) {//할아버지의 위치가 증조부의 왼쪽이였다면
            grandparent->parent->left = parent; //부모를 증조부의 왼쪽으로
        }
        else {
          grandparent->parent->right = parent; //아니라면 부모를 증조부의 오른쪽으로
        }      
    }
    grandparent->parent = parent;  // 할아버지의 부모를 parent로

    // 색 변경
    parent->color = RBTREE_BLACK;
    grandparent->color = RBTREE_RED;  
    if (parent->parent == t->nil) { //회전이 완료되어 할아버지 아래 노드가 없으면
      t->root = parent; //부모를 새로운 트리로 
    }

    return parent; // 새로운 서브트리 루트
}
```

