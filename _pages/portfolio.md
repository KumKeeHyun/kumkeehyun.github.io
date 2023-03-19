---
title: Portfolio
permalink: portfolio
layout: single
classes: wide
---

<a href="mailto:kbzjung359@gmail.com" rel="me" class="u-email">
    <meta itemprop="email" content="kbzjung359@gmail.com" />
    <i class="fas fa-fw fa-envelope-square" aria-hidden="true"></i><span class="label">Email</span>
</a>

<a href="https://github.com/KumKeeHyun" rel="nofollow noopener noreferrer me" itemprop="sameAs">
    <i class="fab fa-fw fa-github" aria-hidden="true"></i>
    <span class="label">GitHub</span>
</a>

<a href="https://www.linkedin.com/in/%EA%B8%B0%ED%98%84-%EA%B8%88-a36b141ba/" rel="nofollow noopener noreferrer me" itemprop="sameAs">
    <i class="fab fa-fw fa-linkedin" aria-hidden="true"></i>
    <span class="label">LinkedIn</span>
</a>



## About Me

Spring 프레임워크로 백엔드 개발 경험이 있습니다. Golang으로 백엔드, 인프라 관련 프로젝트하는 것을 즐깁니다.

데이터 처리, 분산 환경에 관심이 많습니다. 
새로운 지식을 공부할 때는 깊게 파고들어가는 것을 선호합니다. 
때로는 내부구조 파악을 위해 소스 코드를 분석하고, 정리한 내용을 글로 남깁니다.

소스 코드 분석을 기반으로 내부 구조를 파악하고 글로 정리하는 등의 활동 경험이 있습니다.

Trade-off 관계에서 Pareto optimality를 찾아가는 사고 과정을 좋아합니다. 
모든 상황에 맞는 하나의 정답은 없다고 생각하고 다양한 관점, 고려 사항을 기반으로 최적의 선택을 하려 노력합니다.

## Experience

#### 공군 정보체계관리단
2021.04 ~ 2023.01

소프트웨어개발병

###### 공군 인트라넷 웹서비스 백엔드 개발 및 유지보수
- 수사단 업무 지원 체계 개발
  - 결재 라인(결재자 등록, 결재/기각/전결) 기능 구현
  - 직급(6개) 및 소속(10개 이상)에 따른 자원 접근 권한 기능 구현
  - SSO 커스텀 인증 필터 구현
  - 2개월간 기능 개발, 취약성 검사, 시험 평가를 마치고 성공적으로 목표했던 날짜에 서비스 론칭
  - Spring Boot 2, JPA, Oracle

---

- 인사 업무 지원 체계 유지보수
  - Log4Shell 취약점 대응
  - 비지니스 로직 관련 버그 수정
  - 용이한 유지보수를 위해 로컬 라이브러리 형식에서 Maven 프로젝트로 변환
  - JSP, Spring 5, Mybatis, Oracle

---

- 각 체계별로 중복 개발해서 사용하던 기능들을 정리 & 라이브러리 형태로 개발
  - SSO 인증 기능
  - 조직도 트리 형태 조회 기능
  - 파일 업로드/다운로드 기능
  
###### 국방부인사정보체계 데이터 연동 관리
- 국인체에서 일일 연동되는 공군인사자력을 가공하여 공군 자체 인사DB에 반영
  - 국인체 고도화 사업에 따른 연동 프로시저 재작성
  - 연동 테이블 약 30개, 타겟 테이블 약 15개
  - 일일 연동 데이터 평균 30,000개 
  - PL/SQL, Oracle

## Skills

> 3: 내부 동작 방식을 알고있음, 트러블슈팅을 할 수 있음<br>2: 기본적인 동작 방식을 알고있음, 기본적인 기능 구현 가능<br>1: 코드를 읽을 수 있음, 약간의 변경사항 추가 가능

#### Language

<figure class="third">
    <ul>
        <li><span><span class="btn btn--info btn--small">3</span> Golang</span></li>
    </ul>
    <ul>
        <li><span><span class="btn btn--success btn--small">2</span> Java</span></li>
    </ul>
    <ul>
        <li><span><span class="btn btn--inverse btn--small">1</span> Python</span></li>
    </ul>
</figure>

---
  
#### Backend

<figure class="third">
    <ul>
        <li><span><span class="btn btn--success btn--small">2</span> Gin</span></li>
        <li><span><span class="btn btn--success btn--small">2</span> Gorm</span></li>
    </ul>
    <ul>
        <li><span><span class="btn btn--success btn--small">2</span> Spring MVC</span></li>
        <li><span><span class="btn btn--success btn--small">2</span> Spring Security</span></li>
    </ul>
    <ul>
        <li><span><span class="btn btn--success btn--small">2</span> Jpa</span></li>
        <li><span><span class="btn btn--inverse btn--small">1</span> Mybatis</span></li>
    </ul>
</figure>

---

#### DB

<figure class="third">
    <ul>
        <li><span><span class="btn btn--success btn--small">2</span> SQL</span></li>
        <li><span><span class="btn btn--inverse btn--small">1</span> Redis</span></li>
    </ul>
    <ul>
        <li><span><span class="btn btn--success btn--small">2</span> Kafka</span></li>
        <li><span><span class="btn btn--inverse btn--small">1</span> Elasticsearch</span></li>
    </ul>
    <ul>
        <li><span><span class="btn btn--inverse btn--small">1</span> Etcd</span></li>
    </ul>
</figure>

---

#### Data Processing

<figure class="third">
    <ul>
        <li><span><span class="btn btn--inverse btn--small">1</span> Kafka Streams</span></li>
    </ul>
</figure>

#### Container

---

<figure class="third">
    <ul>
        <li><span><span class="btn btn--success btn--small">2</span> Docker</span></li>
    </ul>
    <ul>
        <li><span><span class="btn btn--success btn--small">2</span> Kubernetes</span></li>
    </ul>
</figure>

## Projects

### perisco
2022.06 ~ 현재

eBPF를 이용해서 쿠버네티스 파드 네트워크를 모니터링하는 솔루션

- 개인 프로젝트
- k8s 특정 cni에 종속적이지 않은 네트워크 모니터링을 만들어보자는 목표로 시작
- [github](https://github.com/KumKeeHyun/perisco)
- ebpf를 이용하여 socket_send/recv 관련 함수에서 송수신 데이터 복사
- raw bytes에서 http1, http2 등 7계층 프로토콜 파싱
- pods 관련 메타데이터 보강 후 저장 및 시각화
- Cilium/ebpf, Kubernetes

### gstream
2022.11 ~ 2023.01

Kafka Streams DSL을 모방한 Golang 채널 스트림 처리 라이브러리

- 개인 프로젝트
- Golang에도 Java의 스트림 API같은 라이브러리가 있으면 좋겠다는 생각 & 마침 generic이 새로 추가되어서 시작
- [github](https://github.com/KumKeeHyun/gstream)
- Golang으로 Kafka Stream의 KStream, KTable 개념 구현
- filter, map, groupby 등 함수형 유틸리티 구현
- Golang, BoltDB, Kafka Streams

### ToIoT
2020.07 ~ 2020.11

센서 데이터를 수집, 가공, 시각화하는 IoT 플랫폼

🏆수상🏆 2020 공개SW 개발자대회 학생부문 동상

- 4인 프로젝트(FE 1명, BE 2명, 센서 네트워크 1명)
- 대규모 센서 데이터(미세먼지)를 쉽게 수집, 가공, 시각화하는 플랫폼을 목표로 개발
- [github](https://github.com/SSU-NC/toiot)
- 맡은 일
  - 센서 등록, 조회 API 개발
  - 사용자 정의 데이터 처리를 위한 API 개발
  - 센서 데이터를 가공하는 Kafka Consumer 개발
- Golang, Gin, Gorm, MySQL, Kafka, Elasticsearch

## Article

- [(2023.03.10) etcd raft 모듈 사용해보기](https://kumkeehyun.github.io/posts/using-etcd-raft)
- [(2022.09.09) 함수형 프로그래밍 모나드 ppt 발표](https://speakerdeck.com/kums/gonggun-jeongbocegyegwanridan-jeonsanhanmadang-monadeu)
- [(2021.03.28) etcd raft 모듈 분석](https://kumkeehyun.github.io/posts/etcd-raft-insides)

## Education

#### 숭실대학교

2018.03 ~ 2024.02

AI융합학부 학사 졸업(예정)

#### 세종 한솔고등학교

2015.03 ~ 2018.02

자연계 졸업
