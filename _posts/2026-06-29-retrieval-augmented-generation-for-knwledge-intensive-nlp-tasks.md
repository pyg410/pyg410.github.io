---
layout: post
title: Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks
categories: [ai-engineering]
---

2021.04.12 Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks 논문 리뷰

## 0. Abstract

사전학습된 매개변수(parametric) 메모리와 비매개변수(non-parametric) 메모리를 결합하여 언어를 생성하는 모델인 검색 증강 생성(RAG, Retrieval-Augmented Generation)을 위한 범용 미세 조정 방법을 탐구함.
> \* Prametric Memory: pretrained된 seq2seq 모델 <br>
> \* Non-parametric Memory: pretrained된 NLP를 통해 접근하는 문서(ex. 위키피디아)의 Dense vector index

<br>

해당 연구의 RAG 방식
1. 전체 생성된 시퀀스에서 동일한 검색된 구절을 조건으로 사용하는 방식
2. 토큰별로 다른 구절을 사용할 수 있는 방식

## 1. Introduction

기존 Pre-trained neural language model의 한계
- Memory를 쉽게 확장하거나 수정할 수 없다.
- 예측에 대한 insight를 직접적으로 제공할 수 없다.
- Hallucination을 유발한다.

<br>

Parametric + Non-parametric(검색기반) 기억을 결합한 하이브리드 모델(REALM)
- 지식을 직접 수정 및 확장이 가능
- 접근한 지식을 검토 및 해석 가능

<br>
해당 연구에서는 Seq2Seq 모델에 하이브리드 매개변수 및 비매개변수 Memory를 도입함.

<br>

\[연구방식\] <br>
Pre-trained된 파라미터 메모리 생성 모델에 일반적인 미세 조정 접근 방식인 RAG를 통해 비 파라미터 메모리를 부여함.

<br>
이 구성요소를 결합하여 End-to-end로 학습된 확률 모델을 만듦. <br>
DPR(Dense Passage Retriever)은 입력에 따라 조건화된 잠재문서를(latent documents) 제공하며, Seq2seq 모델(BART)은 이러한 잠재 문서와 입력을 함께 조건화 하여 출력을 만듦.

> \* 잠재문서(latent documents): 단어들의 단순한 빈도배열 대신, 문서 속에 숨겨진 주제(topic)나 의미 구조를 수학적/통계적 공간에 압축하여 표현한 개념

<br>
top-k 근사치를 사용하여 latent documents을 주변화(marginalize)한다. 
모든 토큰이 동일한 문서에 생성된다고 가정할 때, 이는 출력별, 토큰별로 수행됨.
(서로 다른 문서는 서로 다른 토큰이 생성한다고 가정)<br>
T5 또는 BART와 마찬가지로 RAG는 모든 seq2seq 작업에 대해 미세조정(파인튜닝)할 수 있으며, 이 과정에서 generation(생성기)와 retriever(검색기)가 함께 학습된다.

> \* marginalize: 공식에서 시그마로 나타남. 상관없는 z(문서)에 대한 정보를 summarize함으로서 제거하는 방법

<br>
예전에는 특정 작업을 위해 처음부터 학습된 Non-parametric memory로 시스템을 강화하는 아키텍처를 제안하는 폭넓은 연구가 있었음. <br>


> \* e.g. Memory networks, stack-augmented network, memory layers

<br>

반대로 이 연구에서는 매개변수적 및 비매개변수적 메모리 구성 요소 모두가 사전학습되고, 광범위한 지식으로 사전로드된 환경을 연구함.<br>
중요한건 사전 학습된 접근 메커니즘을 사용함으로써 추가 학습 없이도 지식에 접근 가능.

연구에서 사용 검증: Natural Questions, WebQuestions, CuratedTrec, FEVER

![Figure1](https://arxiv.org/html/2005.11401v4/x1.png)
\[Figure1\] <br>

1. 쿼리 x에 대해 최대 내적 검색(MIPS, Maximum Inner Product Search)를 사용하여 top-K개 문서 z_i를 찾는다.
2. 최종 예측값 y를 얻기위해, z를 잠재변수로 간주하고, 서로 다른 문서에 대한 seq2seq 예측값을 주변화(marginalize)한다.

## 2. Method

RAG모델에서 입력 시퀀스 x를 사용하여 텍스트 문서 z를 검색하고(retrieve), 이를 추가적인 Context로 활용하여 target 시퀀스 y를 생성한다.

Figure1에서 보는 것과 같이 해당 연구의 모델은 두 가지 구성요소를 활용한다.
1. 쿼리 x가 주어졌을 때, 텍스트 구절에 대한 top-k가 잘린 분포를 반환하는 매개변수 η를 갖는 검색기 $p_{\eta}(z \mid x)$
2. 이전 $i-1$개의 토큰 $y_(1:i-1)$, 원래 입력 및 검색된 구절 $z$의 컨텍스트를 기반으로 현재 토큰을 생성하는 생성기 $p_{\theta}(y_{i} \mid x,z,y_{1:i-1})$ ($\theta$로 파라미터화 됨)

Retriever, Generator를 End-to-end로 학습시키기 위해, 검색된 문서를 잠재 변수로 취급한다.
생성된 텍스트에 대한 분포를 만들기 위해 잠재 문서에 대해 서로 다른 방식으로 주변화하는 두 가지 모델을 제안 한다.
첫 번째 접근 방식인 RAG-Sequence는 동일한 문서를 사용하여 각 목표 토큰을 예측한다.
두 번째 접근 방식인 RAG-Token은 서로 다른 문서를 기반으로 각 목표 토큰을 예측할 수 있다.

## 2.1 Models

### RAG-Sequence Model

RAG-Sequence 모델은 검색된 문서를 사용해서 전체 시퀀스를 생성한다.
기술적으로는, retrieved 문서를 단일 잠재 변수로 취급하고, 이를 주변화하여 top-k개 근사법(approximation)을 통해 seq2seq 확률 $p(y \mid x)$를 얻는다.
구체적으로는 retriever를 사용하여 top-k개 문서를 검색하고, generator는 각 문서에 대한 출력 시퀀스 확률 생성한 다음 이를 주변화한다.

요약하자면 질문(x)에 대해 가장 쓸만한 문서 top-k개(z)를 뽑은 뒤, 각 문서를 통째로 참고해서 답변(y)를 처음부터 끝까지 만들었을 때의 확률을 구하고, 이를 문서의 검색 점수와 곱해서 모두 더한 값이다.

### RAG-Token Model

RAG-Token 모델에서는 각 대상 토큰에 대해 서로 다른 잠재 문서(intent documents)를 생성하고 그에 따라 주변화(marginalize)할 수 있다.

이를 통해 Generator가 답변을 생성할 때 여러 문서에서 내용을 선택할 수 있다.

구체적으로, Retriever를 사용하여 top-k문서를 검색한 다음, generator는 각 문서에 대한 다음 출력한 토큰의 분포를 생성하고 곱한다.
그다음 이를 주변화한(z에 대한 시그마) 후 다음 출력 토큰으로 프로세스를 반복한다.

마지막으로, RAG는 target class 길이가 1인 목표 시퀀스로 간주하는 경우, 시퀀스 분류작업에서 사용할 수 있다.
이 경우 RAG-Sequence와 RAG-Token은 동일하다.
왜냐하면 정답이 단어 딱 한 개(길이가 1)인 문제를 풀 때는, RAG-Sequence, RAG-Token 방식이던 수학적으로 완전히 동등한 모델이 되기 때문이다.
조금 더 풀어보자면, RAG-Sequence는 답변 전체를 만드는 동안 처음 고른 문서 1개를 끝까지 고정해서 사용한다.
RAG-Token방식은 답변의 매 단어(토큰)을 적을 때마다 문서를 계속 새로 검색하거나 다른 문서로 갈아타며 참조한다.
즉, RAG-Sequence도 첫 단어를 만들기 전에 문서 하나 고르고 단어 하나를 만들게 되고, RAG-Token도 첫 단어 만들기 전에 문서 하나 고르고 단어 하나를 만들고 다음 단어가 없기 때문에 RAG-Sequence와 동일하게 끝난다.

즉, 길이가 1인경우 Token방식의 문서를 바꿀 기회가 주어지지 않기 때문에 바로 종료되어 Sequence방식과 동일하게 된다.

## 2.2. Retriever: DPR

이어서 계속..