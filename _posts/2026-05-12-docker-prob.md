---
layout: post
title: 폐쇄망 Docker Network Subnet 충돌로 인해 특정 서버 SSH/Ping Timeout 발생한 장애 분석
categories: [infra]
---
개발서버에서 Docker compose를 이용해 container를 띄운 후 부터, 갑자기 Local → 개발서버로의 모든 SSH/PING/HTTP 요청에 대해 Timeout이 발생했다.
<br>
처음에는 방화벽 문제라고 생각했지만, 실제 원인은 Docker Network Subnet 충돌로 인해 호스트 라우팅 테이블이 꼬인 문제였다.

이번 글에서는 다음 순서로 정리한다.

* 문제 상황
* 문제 원인
* 원인 분석 과정
* 해결 방법
* 재발 방지 방법

---

# 문제 상황

운영 중인 서버에서 갑자기 다음과 같은 현상이 발생했다.

## 증상

* Docker compose 컨테이너를 띄운 서버만 SSH timeout 발생
* ping timeout 발생
* 다른 서버들은 정상 접속 가능
* 다른 서버 → 문제 서버 SSH, HTTP 가능
* 로컬 PC → 문제 서버만 접속 불가
* Docker 컨테이너 포트는 LISTEN 상태
* sshd 프로세스 정상 동작

예시:

```bash
ssh user@server -p 7002
```

결과:

```text
Connection timed out
```

ping도 동일하게 timeout 발생:

```bash
ping server-ip
```

---

# 처음 의심했던 원인들

처음에는 아래 항목들을 의심했다.

* 운영 방화벽 whitelist 누락
* fail2ban 차단
* sshd 장애
* Docker iptables 충돌
* Security Group 문제
* 서버 네트워크 장애

하지만 이상한 점이 있었다.

## 이상했던 점

다른 서버에서는 문제 서버로 SSH 및 HTTP 요청이 정상 동작했다.

즉

```text
다른 서버 -> 문제 서버 : 정상
내 PC -> 문제 서버 : timeout
```

이 상태였다.

즉 서버 자체는 살아있고,
특정 source network에서만 문제가 발생하는 상황이었다.

---

# 원인 분석

문제 서버에서 Docker Network 목록을 확인했다.

```bash
docker network ls
```

이후 특정 네트워크의 subnet을 확인했다.

```bash
docker network inspect 네트워크명
```

그리고 문제를 발견했다.

## 문제의 Docker Subnet

Docker Network가 다음과 같은 subnet을 사용 중이었다.

```text
aaa.bbb.0.0/16
```

그런데 내 로컬 PC IP도 동일한 대역이었다.

예:

```text
aaa.bbb.x.x
```

---

# 실제로 발생한 문제

Docker는 Network를 생성할 때 호스트 OS의 라우팅 테이블에 route를 추가한다.

호스트 라우팅 테이블 확인

```bash
ip route
```

문제 상황에서는 이런 route가 추가되어 있었다.

```text
aaa.bbb.0.0/16 dev br-xxxxx
```

의미:

```text
aaa.bbb.* 대역은 Docker Bridge Network로 보내라
```

즉 서버는 내 PC IP를 외부 네트워크가 아니라 Docker 내부 네트워크 주소로 오인하게 되었다.

---

# 왜 timeout이 발생했는가?

패킷 흐름은 다음과 같았다.
```text
내 PC -> 서버
```
패킷은 정상적으로 서버까지 도착한다.
```text
서버 -> 내 PC 응답
```
문제는 여기서 발생한다.

서버는

```text
aaa.bbb.0.x 는 Docker Network 대역
```

이라고 판단하여 응답 패킷을 실제 NIC(eth0)가 아니라 Docker Bridge(br-xxxx)로 보내버렸다.

즉 응답 패킷이 잘못된 인터페이스로 라우팅되면서

* ping timeout
* ssh timeout
* http timeout

이 발생한 것이다.

---

# 확인 방법

### 현재 라우팅 테이블 확인

```bash
ip route
```

### 특정 IP가 어디로 라우팅되는지 확인

```bash
ip route get 내PC_IP
```

문제 상황에서는

```text
aaa.bbb.0.106 dev br-xxxxx
```

처럼 Docker Bridge로 라우팅되고 있었다.

---

# 해결 방법

문제가 되는 Docker Network를 삭제했다.

```bash
docker network rm 네트워크명
```

삭제 직후 모든 문제가 즉시 해결되었다.

* ping 정상화
* SSH 정상화
* HTTP 정상화

---

# 재발 방지 방법

가장 중요한 건 Docker Network 대역을 명시적으로 관리하는 것이다.

특히 폐쇄망/사내망 환경에서는 반드시 필요하다.

---

# 방법 1. Docker Default Address Pool 고정

`/etc/docker/daemon.json`

```json
{
  "default-address-pools": [
    {
      "base": "172.30.0.0/16",
      "size": 24
    }
  ]
}
```

적용 후 아래 명령어 실행

```bash
sudo systemctl restart docker
```

---

# 방법 2. Docker Compose에서 Subnet 명시

```yaml
networks:
  backend:
    ipam:
      config:
        - subnet: aaa.bbb.10.0/24
```

---

# 운영 환경에서 주의할 점

사내망/VPN 환경에서는 아래 대역이 이미 사용 중인 경우가 많다.

```text
10.x.x.x
192.168.x.x
172.16.x.x ~ 172.31.x.x
```

Docker 자동 subnet 할당을 그대로 사용하면 실제 네트워크와 충돌할 수 있다.

---

# 결론

이번 장애의 핵심 원인은 

> Docker Network Subnet이 실제 네트워크 대역과 충돌하면서
> 호스트 라우팅 테이블이 Docker Bridge를 우선하게 된 것

이었다.

Docker Network는 단순히 컨테이너 내부만 사용하는 것이 아니라,
호스트 OS의 라우팅에도 직접 영향을 준다.

특히 폐쇄망/사내망 환경에서는 반드시 Docker Subnet 정책을 명시적으로 관리해야 한다.
