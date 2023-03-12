---
title: Portfolio
permalink: portfolio
layout: single
classes: wide
---

## About Me



## Experience

#### 공군 정보체계관리단
2021.04 ~ 2023.01

소프트웨어개발병

###### 공군 인트라넷 웹서비스 백엔드 개발 및 유지보수
- 수사단 업무 지원 체계 개발
  - 결재 라인(결재자 등록, 결재/기각/전결) 기능 개발
  - 직급(6개) 및 소속(10개 이상)에 따른 자원 접근 권한 기능 개발
  - 2개월간 기능 개발 + 취약성 검사 + 기능 평가를 끝내고 성공적으로 목표했던 날짜에 서비스 론칭
  - Spring Boot 2 + JPA + Oracle

---

- 인사 업무 지원 체계 유지보수
  - Log4Shell 취약점 대응
  - 비지니스 로직 관련 버그 수정
  - 용이한 유지보수를 위해 Maven 프로젝트로 변환 작업
  - JSP + Spring 5 + Mybatis + Oracle

---

- 서비스별로 중복 개발해서 사용하던 기능들을 정리 & 라이브러리 형태로 개발
  - SSO 인증 기능
  - 조직도 트리 형태 조회 기능
  - 파일 업로드/다운로드 기능
  
###### 국방부인사정보체계 데이터 연동 관리
- 국인체에서 일일 연동되는 공군인사자력을 가공하여 공군 자체 인사DB에 반영
  - 국인체 고도화 사업에 따른 연동 프로시저 재작성
  - 연동 테이블 약 30개, 타겟 테이블 약 15개
  - 일일 연동 데이터 평균 30,000개 
  - PL/SQL + Oracle

## Skills

> 3: 내부 동작 방식을 알고있음, 트러블슈팅을 할 수 있음<br>2: 기본적인 동작 방식을 알고있음, 기본적인 기능 구현 가능<br>1: 코드를 읽을 수 있음, 약간의 변경사항 추가 가능

#### Language

<figure class="third">
    <ul>
        <li><span><span class="btn btn--info btn--small">2.5</span> Golang</span></li>
    </ul>
    <ul>
        <li><span><span class="btn btn--info btn--small">2</span> Java</span></li>
    </ul>
    <ul>
        <li><span><span class="btn btn--inverse btn--small">1</span> Python</span></li>
    </ul>
</figure>
  
#### Backend

<figure class="third">
    <ul>
        <li><span><span class="btn btn--success btn--small">2</span> Gin</span></li>
        <li><span><span class="btn btn--success btn--small">1.5</span> Gorm</span></li>
    </ul>
    <ul>
        <li><span><span class="btn btn--success btn--small">2</span> Spring MVC</span></li>
        <li><span><span class="btn btn--info btn--small">2.5</span> Spring Security</span></li>
    </ul>
    <ul>
        <li><span><span class="btn btn--success btn--small">2</span> Jpa</span></li>
        <li><span><span class="btn btn--inverse btn--small">1</span> Mybatis</span></li>
    </ul>
</figure>

#### DB

<figure class="third">
    <ul>
        <li><span><span class="btn btn--success btn--small">2</span> SQL(Oracle)</span></li>
        <li><span><span class="btn btn--success btn--small">1.5</span> Redis</span></li>
    </ul>
    <ul>
        <li><span><span class="btn btn--success btn--small">2</span> Kafka</span></li>
        <li><span><span class="btn btn--inverse btn--small">1</span> Elasticsearch</span></li>
    </ul>
    <ul>
        <li><span><span class="btn btn--inverse btn--small">1</span> Etcd</span></li>
    </ul>
</figure>

#### Data Processing

<figure class="third">
    <ul>
        <li><span><span class="btn btn--success btn--small">1.5</span> Kafka Streams</span></li>
    </ul>
</figure>

#### Container

<figure class="third">
    <ul>
        <li><span><span class="btn btn--success btn--small">2</span> Docker</span></li>
    </ul>
    <ul>
        <li><span><span class="btn btn--success btn--small">1.5</span> Kubernetes</span></li>
    </ul>
</figure>

## Projects

### perisco
2022.06 ~ 현재

eBPF를 이용해서 쿠버네티스 파드 네트워크를 모니터링하는 솔루션

- 개인 프로젝트
- [github](https://github.com/KumKeeHyun/perisco)
- Cilium/ebpf, Kubernetes

### gstream
2022.11 ~ 2023.01

Kafka Streams DSL을 모방한 Golang 채널 스트림 처리 라이브러리

- 개인 프로젝트
- [github](https://github.com/KumKeeHyun/gstream)
- Golang, BoltDB

### ToIoT
2020.07 ~ 2020.11

센서 데이터를 수집, 사용자 정의 가공, 시각화하는 IoT 플랫폼

🎂수상🎂 2020 공개SW 개발자대회 학생부문 동상

- 4인 프로젝트
- [github](https://github.com/SSU-NC/toiot)
- 맡은 일
  - 센서 등록, 조회 API 개발 (Gin + Gorm)
  - 센서 데이터를 가공하는 Kafka Consumer 개발 (Kafka + Shopify/sarama)

## Article

- [(2023.03.10) etcd raft 모듈 사용해보기](https://kumkeehyun.github.io/posts/using-etcd-raft)
- [(2021.03.28) etcd raft 모듈 분석](https://kumkeehyun.github.io/posts/etcd-raft-insides)

## Education

#### 숭실대학교

2018.03 ~ 2024.02

AI융합학부 학사 졸업(예정)

#### 세종 한솔고등학교

2015.03 ~ 2018.02

자연계 졸업
