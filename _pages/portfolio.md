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

Golang으로 백엔드, 인프라 관련 프로젝트하는 것을 즐깁니다.

분산 환경과 쿠버네티스 컴포넌트 등에 관심이 많습니다.
새로운 지식을 공부할 때는 깊게 파고들어가는 것을 선호합니다. 인터넷 자료보다는 책을 통해 공부하는 것을 선호합니다. 때로는 내부구조를 이해하기 위해 소스코드 분석도 진행합니다.

Trade-off 관계에서 Pareto optimality를 찾아가는 사고 과정을 좋아합니다. 
모든 상황에 맞는 하나의 정답은 없다고 생각하고 다양한 관점, 고려 사항을 기반으로 최적의 선택을 하려 노력합니다.


## Projects

### perisco
2022.06 ~ 2022.10, 2023.03 ~ 2023.05

eBPF를 활용하여 마이크로서비스 네트워크를 모니터링하는 솔루션

- 개인 프로젝트
- 서비스에 네트워크 로그 저장과 관련된 코드가 없어도 자동으로 로그 생성이 가능합니다
- 특정한 k8s cni에 종속되지 않고 flannel, callico, cilium 등의 cni들에서 잘 작동합니다
- 불필요한 이벤트 수신을 방지하기 위해 Circuit Breaker 패턴을 적용했습니다
- 추후 다른 프로토콜을 추가할 수 있도록 확장성 있는 설계를 위해서 Composite, Factory 등의 패턴을 적용했습니다
- HTTP/2 로그 생성을 위해서 프로토콜의 Stream, Frame, Hpack 등 세부 구조에 대해 공부했습니다
- cilium/ebpf, Kubernetes, Grafana
- [github link](https://github.com/KumKeeHyun/perisco)

### godis
2023.01 ~ 2023.03, 2023.07

Raft 알고리즘 기반 복제 기능을 지원하는 간단한 분산 데이터베이스

- 개인 프로젝트
- etcd-raft 라이브러리를 이용하여 Raft 알고리즘 기반 복제 기능이 있는 분산 데이터베이스 구현
- Redis 문서를 참고하여 Redis Serialization Protocol v2 를 구현했습니다.
- etcd-raft 라이브러리를 사용하여 복제 기능 및 스냅샷 기능을 구현했습니다.
- 쿠버네티스 커스텀 컨트롤러를 구현하여 선언적으로 배포, 확장할 수 있는 기능을 구현했습니다.
- Golang, etcd/raft, Kubernetes
- [github link](https://github.com/KumKeeHyun/godis)


### gstream
2022.11 ~ 2023.01

Kafka Streams DSL을 모방한 Golang 채널 스트림 처리 라이브러리

- 개인 프로젝트
- Go 언어의 채널을 기반으로 Kafka Streams 의 KStream, KTable 개념을 모방하는 스트림 구현
- KStream, KTable 개념을 구현하기 위해 실제 Kafka Streams의 소스코드를 분석했습니다
- map, flatMap, reduce 등 함수형 유틸리티 구현을 위해서 모나드 개념을 공부했습니다
- stream backpressure를 해결하기 위해 채널 버퍼 조절, 고루틴 추가 생성 등의 기능을 구현했습니다
- Golang, BoltDB, Kafka Streams
- [github link](https://github.com/KumKeeHyun/gstream)

### ToIoT
2020.07 ~ 2020.11

센서 데이터를 수집, 가공, 시각화하는 IoT 플랫폼

🏆수상🏆 2020 공개SW 개발자대회 학생부문 동상

- 4인 프로젝트(FE 1명, BE 2명, 센서 네트워크 1명)
- 센서 데이터(미세먼지)를 쉽게 수집, 가공, 시각화할 수 있는 플랫폼을 목표로 진행했습니다
- 미세먼지 측정 범위가 지역구 단위가 아니라 더 세밀한 단위로 측정되면 좋겠다고 생각하여 대규모 처리 요구사항을 추가했습니다
- [github link](https://github.com/SSU-NC/toiot)
- 맡은 일
  - 센서 등록, 조회 API 개발
  - 사용자 정의 데이터 처리를 위한 API 개발
  - 센서 데이터를 가공하는 Kafka Consumer 개발
- Golang, Gin, Gorm, MySQL, Kafka, Elasticsearch

## Article

- [(2023.03.10) etcd raft 모듈 사용해보기](https://kumkeehyun.github.io/posts/using-etcd-raft)
  - etcd의 raft 라이브러리를 사용해서 간단한 Distributed key-value store를 구현해본 경험을 정리한 아티클입니다
  - 해당 활동의 결과로 수정 사항을 찾아서 직접 Pull Request를 보내는 활동을 했습니다
- [(2022.09.09) 함수형 프로그래밍 모나드 ppt 발표자료](https://www.slideshare.net/ssuser094f811/pdf-258182147)
  - 공군 정보체계관리단에서 진행한 개발 관련 세미나에서 모나드를 주제로 세션을 진행한 발표자료입니다
- [(2021.03.28) etcd raft 모듈 분석해보기](https://kumkeehyun.github.io/posts/etcd-raft-insides)
  - etcd에서 복제 기능에 사용하고 있는 raft 알고리즘 구현 라이브러리 코드를 분석하고 정리한 아티클입니다
  - 정리한 글을 Golang Korea 커뮤니티에 게재하여 경험을 공유하는 활동을 했습니다

## Experience

#### 공군 정보체계관리단
2021.04 ~ 2023.01

소프트웨어개발병

###### 공군 인트라넷 웹서비스 백엔드 개발 및 유지보수

---

- 수사단 업무 지원 체계 개발
  - 프로젝트 설계, 구현, 릴리즈, 운영 등 모든 과정에 참여
    - 2개월간 설계, 기능 개발, 취약성 검사, 시험 평가를 마치고 성공적으로 목표했던 날짜에 서비스 론칭
  - 직급(6개) 및 소속(50개 이상)에 따른 복잡한 자원 접근 권한 기능을 구현
    - 개념 정리, 객체 설계, 디자인 패턴 적용, 세세한 테스트 케이스 작성 등의 작업을 함
    - 릴리즈 이후 운영 시점에서 권한 관련 이슈가 발행하지 않았음
  - Spring Boot 2, JPA, Oracle

---

- 인사 업무 지원 체계 유지보수
  - 레거시 프로젝트를 받아서 코드를 분석하고 유지보수에 참여
    - 지속적으로 오류가 발생했던 문제에 대해서 버그를 찾고 수정하는 작업을 함
    - 유지보수 요청에 따라 기능을 수정하거나 추가하는 작업을 함
    - 추후 용이한 유지보수를 위해 프로젝트 구조를 개선하는 작업을 함
    - Log4Shell 취약점 대응
  - JSP, Spring 5, Mybatis, Oracle

---

- 다른 팀원들이 서비스 개발시 중복 개발해서 사용하던 기능들을 정리 & 라이브러리 형태로 개발
  - 지속적인 유지보수를 위해서 객체, 패키지 의존성을 깔끔하게 유지하려 노력
  - 라이브러리 사용사례에 따라서 autoconfiguration, 빈 등록 등의 형태로 개발
  - 사용 편의성과 추후 유지보수를 위해서 문서를 상세하게 작성함
  - 실제로 다른 팀원들의 서비스 개발 속도가 향상되었음
  - SSO 인증 필터, 조직도 트리 형태 조회 기능, 파일 업로드/다운로드 API
  
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
        <li><span><span class="btn btn--inverse btn--small">1</span> Spark(pyspark)</span></li>
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

## Education

#### 숭실대학교

2018.03 ~ 2024.02

AI융합학부 학사 졸업(예정)

#### 세종 한솔고등학교

2015.03 ~ 2018.02

자연계 졸업
