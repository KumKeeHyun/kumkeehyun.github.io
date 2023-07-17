---
title: "쿠버네티스 컨트롤러로 선언형 DBaaS 만들어보기"
permalink: posts/kubernetes-custom-controller
classes: wide

categories:
  - database
  - kubernetes
tags:
  - distributed 
  - raft
  - kubernetes
  - crd
last_modified_at: 2023-07-07T00:00:00-00:00
---

<div style="display: none">
	<a href="https://hits.seeyoufarm.com"><img src="https://hits.seeyoufarm.com/api/count/incr/badge.svg?url=https%3A%2F%2Fkumkeehyun.github.io%2Fposts%2Fkubernetes-custom-controller&count_bg=%23F9B0B0&title_bg=%23555555&icon=&icon_color=%23E7E7E7&title=hits&edge_flat=false"/></a>
</div>

- 이전 시리즈
  - [etcd raft 모듈 분석해보기](./etcd-raft-insides)
  - [etcd raft 모듈 사용해보기](./using-etcd-raft)

# TOC
<!--ts-->
- [TOC](#toc)
- [서론](#서론)
- [Kubernetes Controller](#kubernetes-controller)
	- [Control Plain](#control-plain)
	- [Controller Manager](#controller-manager)
	- [Controller 기본 구조](#controller-기본-구조)
	- [Expectations](#expectations)
		- [사용법 및 예시](#사용법-및-예시)
	- [Optimistic Locking(resourceVersion)](#optimistic-lockingresourceversion)
- [Godis Controller](#godis-controller)
	- [Godis 가내수공업 배포](#godis-가내수공업-배포)
	- [CRD 및 컨트롤러 설계](#crd-및-컨트롤러-설계)
		- [Replicas](#replicas)
		- [ID](#id)
		- [CRD, Resources](#crd-resources)
		- [initial-cluster](#initial-cluster)
	- [SyncHandler 구현](#synchandler-구현)
		- [Status](#status)
		- [클러스터 생성](#클러스터-생성)
		- [클러스터 확장](#클러스터-확장)
		- [클러스터 축소](#클러스터-축소)
		- [단계적 스케일링](#단계적-스케일링)
	- [결과물](#결과물)
		- [클러스터 생성](#클러스터-생성-1)
		- [클러스터 확장](#클러스터-확장-1)
		- [클러스터 축소](#클러스터-축소-1)
		- [포드 중단 및 자동 복구](#포드-중단-및-자동-복구)
		- [노드 제거 및 자동 복구](#노드-제거-및-자동-복구)
- [마치면서](#마치면서)
	- [Reference](#reference)
<!--te-->

# 서론

4학년 복학, 졸업 작품, 상반기 공채 등등... 바쁜 상반기를 보내면서 못하고 있던 `젤다:왕눈`을 구매하고 열심히 젤다를 구하고 있었습니다. 
그러다 공채 최종면접 탈락 이메일을 받아버려서 하반기도 취업 준비를 해야 한다는 절망을 안고 다시 책상 앞에 앉았습니다... 😭

공부할 주제를 찾고있다가 [쿠버네티스를 활용한 선언형 클라우드 DB 서비스](https://engineering.linecorp.com/ko/blog/declarative-cloud-db-service-using-kubernetes)라는 글을 읽게 되었는데, 자연스럽게 저번 방학때 만들었던 간단한 분산 데이터베이스인 [godis](https://github.com/KumKeeHyun/godis)가 생각났습니다. 
평소 DBaaS에 대해서 궁금한 점도 있었고 쿠버네티스 컨트롤러는 이전에 공부하려 했던 주제여서 godis를 기반으로 커스텀 컨트롤러를 만들어보기로 했습니다.

프로젝트 목표는 yaml 파일 하나로 godis 클러스터를 배포, 확장, 축소하는 커스텀 컨트롤러 개발하는 것으로 세웠습니다. 
구현에 앞서 개념 공부를 위해 `Kubernetes in Action`를 다시 읽었고, [sample-controller](https://github.com/kubernetes/sample-controller), [kubernetes-replicaset-controller](https://github.com/kubernetes/kubernetes/tree/master/pkg/controller/replicaset) 코드 분석을 진행했습니다.

이 글에서는 쿠버네티스 컨트롤러에 대해서 공부한 내용을 정리하고 Godis Controller를 구현한 과정을 풀어서 작성했습니다. 쿠버네티스 컨트롤러에 관심이 있으신 분들께 도움이 됐으면 좋겠습니다.

# Kubernetes Controller

쿠버네티스 컨트롤러가 작동하는 과정을 이해하려면, 먼저 쿠버네티스 컨트롤 플레인이 어떻게 구성되어있는지, 각각의 역할이 무엇인지 살펴볼 필요가 있습니다.

## Control Plain

<img width="455" alt="image" src="https://github.com/KumKeeHyun/raspi-cluster/assets/44857109/6e39be95-e331-447a-bb54-205b65de4557">

`etcd`, `API Server`, `Scheduler`, `Controller Manager`는 쿠버네티스 컨트롤 플레인의 기본적인 구성요소 입니다. 여기서 `API 서버`의 역할은 다음과 같습니다.

- RBAC를 통해 인가된 API 요청인지 판단
- 요청 payload가 유효한지 검사
- 요청에 따라 etcd에 저장 및 조회
- 리소스가 변경되면 변경된 사항을 클라이언트들에게 전달

문서의 요약문에는 API 서버는 쿠버네티스에서 중추 역할을 한다고 설명되어 있어서, kubectl로 포드 생성을 요청하면 API Server에서 특정 노드의 kubelet에게 포드 생성 요청을 보낼 것이라고 생각할 수 있습니다.

하지만 API 서버가 정작 하는 일은 사용자 요청에 따라 etcd에 CRUD 작업을 수행하고, 특정 리소스 변경을 구독하는 클라이언트들에게 변경 사항을 알려주는 것이 전부입니다. 즉, API 서버는 etcd에 새로운 포드의 정보를 저장하고, 포드 변경 사항 이벤트를 구독하고있는 클라이언트들에게 전달하는 작업만 담당합니다. 그렇다면 kubelet은 어떻게 포드를 생성하는 걸까요?

<img width="1126" alt="image" src="https://github.com/KumKeeHyun/raspi-cluster/assets/44857109/813128e0-2f2f-412b-ad1c-4e7620522480">

kubelet이 새로 생성할 포드 정보를 받는 과정을 간단하게 정리한 그림입니다.

0. 스케줄러와 kubelet은 포드 변경 사항을 전달받기 위해 API 서버에 Watch 요청
1. API 서버가 새로운 포드 생성 요청을 받고 etcd에 새로운 포드 정보를 저장
2. API 서버가 새로운 포드 정보를 Watch 클라이언트인 스케줄러와 kubelet에게 전달
3. 스케줄러가 새로운 포드를 보고 할당될 노드를 선택한 뒤에 포드 정보를 업데이트
4. API 서버는 스케줄러가 요청한 업데이트 정보를 etcd에 저장하고 변경 사항을 Watch 클라이언트들에게 전달
5. kubelet은 새로 할당된 포드를 보고 컨테이너 생성

이처럼 쿠버네티스는 Watch 메커니즘을 통해서 각 컴포넌트는 새로 선언된 정보를 전달받고 해당 정보에 수렴하도록 다른 리소스들을 조정하는 흐름으로 작동합니다. API 서버는 이런 컴포넌트들에게 일관된 진입점을 제공해주고 추가적으로 권한 검사, 유효성 검사, 일관성 검사(낙관적 락) 등의 작업도 수행합니다.

## Controller Manager

쿠버네티스에는 포드 이외에 ReplicaSet, Deployment, Service... 등의 여러 리소스 타입들이 있습니다. Deployment를 생성하면 ReplicaSet, Pod가 자동으로 생성되는데 이처럼 현재 상태가 선언된 상태로 수렴하도록 조정하는 작업을 수행하는 것이 컨트롤러입니다.

컨트롤러 매니저는 여러 컨트롤러를 모아놓은 것이고 다음과 같은 컨트롤러들이 있습니다.

- ReplicaSet, DaemonSet, and Job controllers
- Deployment controller
- StatefulSet controller
- Node controller
- Service controller
- Endpoints controller
- Namespace controller
- PersistentVolume controller
- ...

컨트롤러 또한 Watch 메커니즘을 통해 작동하는데 ReplicaSet을 예시로 간단하게 작동 과정을 살펴보겠습니다.

<img width="1215" alt="image" src="https://github.com/KumKeeHyun/raspi-cluster/assets/44857109/deafd314-38de-49ea-a192-aec2197e2929">

ReplicaSet 컨트롤러는 Watch API를 통해 ReplicaSet, Pod 리소스들에 대한 변경 사항을 전달받습니다. 이때 새로운 ReplicaSet이 생성되거나 replicas 스펙이 변경된다면 컨트롤러에서 현재 포드 상태를 기반으로 새로운 포드 생성/기존 포드 삭제 등의 작업을 수행하게 됩니다.

## Controller 기본 구조

아래 그림은 [sample-controller](https://github.com/kubernetes/sample-controller)의 구조를 간단하게 정리한 그림입니다. 

<img width="1612" alt="image" src="https://github.com/KumKeeHyun/raspi-cluster/assets/44857109/a53c1b47-47b0-41ae-8bfe-03b04813bda3">

Watch API를 대신 처리해주는 Informer, API 서버에 조회 요청을 보낼 필요가 없도록 리소스들을 캐싱하고 있는 Lister(Indexer), 원하는 상태가 되도록 조정(reconciling)하는 SyncHandler, 조정 작업이 이뤄지도록 이벤트를 전달하는 Queue로 구성되어있습니다.

특이한 점은 

- 리소스 조회 시 API 서버로 요청을 하는 것이 아니라 Informer(Watch)에 의해 관리되는 캐시를 사용
- 이벤트를 전달할 때 리소스 전체를 전달하는 것이 아니라 key만 전달

한다는 것입니다.

[sample-controller](https://github.com/kubernetes/sample-controller)의 코드를 살펴보겠습니다.

```go
// https://github.com/kubernetes/sample-controller/blob/master/controller.go#L88
func NewController(
	ctx context.Context,
	kubeclientset kubernetes.Interface,
	sampleclientset clientset.Interface,
	deploymentInformer appsinformers.DeploymentInformer,
	fooInformer informers.FooInformer) *Controller {
	// ...

	controller := &Controller{
		kubeclientset:     kubeclientset,
		sampleclientset:   sampleclientset,
		// ...
		foosLister:        fooInformer.Lister(),
		foosSynced:        fooInformer.Informer().HasSynced,
		workqueue:         workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), "Foos"),
		// ...
	}
	// ...

	fooInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: controller.enqueueFoo,
		UpdateFunc: func(old, new interface{}) {
			controller.enqueueFoo(new)
		},
	})

	// ...
	return controller
}

// https://github.com/kubernetes/sample-controller/blob/master/controller.go#L346
func (c *Controller) enqueueFoo(obj interface{}) {
	var key string
	var err error
	if key, err = cache.MetaNamespaceKeyFunc(obj); err != nil {
		utilruntime.HandleError(err)
		return
	}
	c.workqueue.Add(key)
}
```

`fooInformer.Informer().AddEventHandler()`를 보면 생성, 수정 이벤트가 발생할 때마다 queue에 이벤트를 전달하는 것을 볼 수 있습니다. 이때 `enqueueFoo()`를 보면 최신 상태의 리소스 대신 `namespace/name` 형태의 key를 queue에 전달하는 것을 볼 수 있습니다.

```go
// https://github.com/kubernetes/sample-controller/blob/master/controller.go#L192
func (c *Controller) processNextWorkItem(ctx context.Context) bool {
	obj, shutdown := c.workqueue.Get()  
	if shutdown {
		return false
	}

	err := func(obj interface{}) error {
		defer c.workqueue.Done(obj)
		var key string
		var ok bool

		// We expect strings to come off the workqueue. These are of the
		// form namespace/name. We do this as the delayed nature of the
		// workqueue means the items in the informer cache may actually be
		// more up to date that when the item was initially put onto the
		// workqueue.
		if key, ok = obj.(string); !ok {
			c.workqueue.Forget(obj)
			utilruntime.HandleError(fmt.Errorf("expected string in workqueue but got %#v", obj))
			return nil
		}
		// Run the syncHandler, passing it the namespace/name string of the
		// Foo resource to be synced.
		if err := c.syncHandler(ctx, key); err != nil {
			// Put the item back on the workqueue to handle any transient errors.
			c.workqueue.AddRateLimited(key)
			return fmt.Errorf("error syncing '%s': %s, requeuing", key, err.Error())
		}
		// Finally, if no error occurs we Forget this item so it does not
		// get queued again until another change happens.
		c.workqueue.Forget(obj)

		return nil
	}(obj)

	// ...
	return true
}

func (c *Controller) syncHandler(ctx context.Context, key string) error {
	namespace, name, err := cache.SplitMetaNamespaceKey(key)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("invalid resource key: %s", key))
		return nil
	}

	// Get the Foo resource with this namespace/name
	foo, err := c.foosLister.Foos(namespace).Get(name)
  // ...
}
```

`syncHandler`를 보면 queue를 통해 전달받은 key를 통해 Lister(캐시)에서 리소스를 조회하는 것을 볼 수 있습니다.

처음에는 addHandler, updateHandler, deleteHandler를 각각 구현해서 eventHandler에 등록하지 않고, queue에 key만 전달하고 하나의 syncHandler에서 add, update, delete 작업을 모두 처리하는지 의문이었습니다.

~~handler라는 단어를 보고 `http.ServeMux`, `http.HandlerFunc`가 떠올라서 서로 다른 이벤트를 한 핸들러로 처리하는 것이 이질적으로 느껴졌던 것 같습니다.~~
 
쿠버네티스 컨트롤러에서는 syncHandler 함수에서 수행하는 작업을 reconcile(조정하다) 라고 표현합니다. 여기에는 '원하는 상태로 만들기 위한 작업을 수행한다' + '그러기 위해 어떤 작업이 필요한지 판단한다' 라는 의미가 녹아있는 것이 아닐까 하고 추측하고 있습니다(뇌피셜 입니다). 

사실 `processNextWorkItem()`의 주석을 읽어보면 어느정도 그 이유를 추측해볼 수 있습니다.

> We expect strings to come off the workqueue. These are of the form namespace/name. We do this as the delayed nature of the workqueue means the items in the informer cache may actually be more up to date that when the item was initially put onto the workqueue.

3개의 이벤트(Update1, Update2, Update3)가 짧은 시간 안에 발생한 상황을 가정해보면, SyncHandler에서 각 이벤트를 따로 전달받아 3번 처리하는 것보다는 3개의 이벤트를 하나의 이벤트로 합쳐서 한번에 처리하는 것이 더 효과적입니다.

<img width="1877" alt="image" src="https://github.com/KumKeeHyun/raspi-cluster/assets/44857109/4e2bca86-2b2a-47ab-9444-f6c3cdf0da74">

위 그림과 같이 조정 작업을 하기 위해서 지연되는 특성이 있는 queue를 사용하고 key 만 전달하는 것이 합리적인 것 같습니다.

## Expectations

SyncHandler를 작성할때 주의해야 하는 점은 리소스 조회 시 API 서버가 아닌 캐시를 조회한다는 것입니다. API 서버의 최신 상태가 반영되지 않은 상태에서 조정 작업을 수행하면 뜻하지 않은 작업이 수행될 수 있습니다.

예를 들어, 아래 그림처럼 API 서버에는 spec에 맞게 3개의 포드가 생성된 상태이지만 아직 캐시에 반영이 되지 않아서 불필요한 포드를 생성하는 상황이 생길 수 있습니다.

<img width="1172" alt="image" src="https://github.com/KumKeeHyun/raspi-cluster/assets/44857109/480e2fe3-3d39-4c3f-a96d-f402c2e00cdd">

사실 시간이 지나면 다시 캐시가 API 서버와 동기화되면서 불필요한 포드가 생성된 것을 확인하고 해당 포드를 제거할 수 있기 때문에, 일반적인 경우에는 큰 문제가 되지 않습니다. 하지만 이런 불필요한 생성/삭제 작업이 어플리케이션의 특성에 따라서 기존 포드들에 큰 영향(데이터베이스 -> id 충돌, scale out 조정 작업...)을 끼칠 수도 있습니다.

쿠버네티스에서는 Create 작업같이 멱등성이 없는 작업을 위해서 Expectations이라는 유틸을 사용합니다.

### 사용법 및 예시

1. `ExpectCreations()`, `ExpectDeletions()`를 통해 원하는 상태를 설정
2. `SatisfiedExpectations()`를 통해 원하는 상태에 도달했는지 확인
  - 따로 block하지는 않고 그냥 bool으로 반환
  - false라면 조정 작업을 진행하지 않고 다음 이벤트를 기다림
  - expections 내부에 ttl이 있기 때문에 조건을 충족하지 않았더라도 expired 되면 true 반환 
3. `CreationObserved()`, `DeletionObserved()`를 통해 현재 상태 변경
  - 2.으로 이동

```go
// https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/controller_utils.go#L147

// ControllerExpectationsInterface is an interface that allows users to set and wait on expectations.
// Only abstracted out for testing.
// Warning: if using KeyFunc it is not safe to use a single ControllerExpectationsInterface with different
// types of controllers, because the keys might conflict across types.
type ControllerExpectationsInterface interface {
	GetExpectations(controllerKey string) (*ControlleeExpectations, bool, error)
	SatisfiedExpectations(controllerKey string) bool
	DeleteExpectations(controllerKey string)
	SetExpectations(controllerKey string, add, del int) error
	ExpectCreations(controllerKey string, adds int) error
	ExpectDeletions(controllerKey string, dels int) error
	CreationObserved(controllerKey string)
	DeletionObserved(controllerKey string)
	RaiseExpectations(controllerKey string, add, del int)
	LowerExpectations(controllerKey string, add, del int)
}
```

사용 예시로 [kubernetes-replicaset-controller](https://github.com/kubernetes/kubernetes/tree/master/pkg/controller/replicaset) 코드를 살펴보겠습니다.

```go
// https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/replicaset/replica_set.go#L670
func (rsc *ReplicaSetController) syncReplicaSet(ctx context.Context, key string) error {
	// ...
	rs, err := rsc.rsLister.ReplicaSets(namespace).Get(name)
	// ...
	rsNeedsSync := rsc.expectations.SatisfiedExpectations(key)
	// ...

	var manageReplicasErr error
	// SatisfiedExpectations 결과가 false라면 조정 작업을 진행하지 않음
	if rsNeedsSync && rs.DeletionTimestamp == nil {
		manageReplicasErr = rsc.manageReplicas(ctx, filteredPods, rs)
	}

	// ...
	if manageReplicasErr == nil && updatedRS.Spec.MinReadySeconds > 0 &&
		updatedRS.Status.ReadyReplicas == *(updatedRS.Spec.Replicas) &&
		updatedRS.Status.AvailableReplicas != *(updatedRS.Spec.Replicas) {
		rsc.queue.AddAfter(key, time.Duration(updatedRS.Spec.MinReadySeconds)*time.Second)
	}
	return manageReplicasErr
}
```

ReplicaSet 컨트롤러의 SyncHandler를 보면 조정 작업을 하기 전에 `expectations.SatisfiedExpectations()`을 통해 생성/삭제했지만 아직 확인되지 않는 포드가 있다면 추가적인 조정 작업을 수행하지 않는 것을 확인할 수 있습니다.

```go
// https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/replicaset/replica_set.go#L565C1-L665C2
func (rsc *ReplicaSetController) manageReplicas(ctx context.Context, filteredPods []*v1.Pod, rs *apps.ReplicaSet) error {
	diff := len(filteredPods) - int(*(rs.Spec.Replicas))
	// ...
	if diff < 0 {
		diff *= -1
		// ....

		// 총 생성되어야 하는 포드 개수 설정
		rsc.expectations.ExpectCreations(rsKey, diff)
		// ...
		successfulCreations, err := slowStartBatch(diff, controller.SlowStartInitialBatchSize, func() error {
			err := rsc.podControl.CreatePods(ctx, rs.Namespace, &rs.Spec.Template, rs, metav1.NewControllerRef(rs, rsc.GroupVersionKind))
			// ...
			return err
		})

		if skippedPods := diff - successfulCreations; skippedPods > 0 {
			// ...
			for i := 0; i < skippedPods; i++ {
				// 포드 생성에 성공한 만큼 Observed 
				rsc.expectations.CreationObserved(rsKey)
			}
		}
		return err
	} else if diff > 0 {
		// ...
	}

	return nil
}
```

만약 조정 작업 중에 diff 만큼 새로운 포드를 생성해야 한다면 `expectatioins.ExpectCreations()`을 통해 원하는 상태를 지정한 뒤에 생성 요청을 보내는 것을 확인할 수 있습니다.

또 생성된 결과를 바로 확인할 수 있는 포드들에 대해서는 `expectations.CreationObserved()`를 통해 바로 expectations에 반영해주는 것도 확인할 수 있습니다.

만약 여기서 모든 expectations이 해결되지 않는다면 

- pod watch에서 addEvent를 통해 나머지 expectations 해결
- expectations이 만료되어 새로 조정 작업 시작

이 있을 수 있습니다.

```go
func NewBaseController(logger klog.Logger, rsInformer appsinformers.ReplicaSetInformer, podInformer coreinformers.PodInformer, kubeClient clientset.Interface, burstReplicas int,
	gvk schema.GroupVersionKind, metricOwnerName, queueName string, podControl controller.PodControlInterface, eventBroadcaster record.EventBroadcaster) *ReplicaSetController {

  // ...

	podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			rsc.addPod(logger, obj)
		},
		// ...
	})

  // ...
}

// https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/replicaset/replica_set.go#L380C1-L418C2
func (rsc *ReplicaSetController) addPod(logger klog.Logger, obj interface{}) {
	pod := obj.(*v1.Pod)
	// ...

	if controllerRef := metav1.GetControllerOf(pod); controllerRef != nil {
		rs := rsc.resolveControllerRef(pod.Namespace, controllerRef)
		// ...
		rsKey, err := controller.KeyFunc(rs)
		if err != nil {
			return
		}
		// ...

    // 포드 생성 이벤트 Observed
		rsc.expectations.CreationObserved(rsKey)
		rsc.queue.Add(rsKey)
		return
	}

	// ...
}
```

조정 작업에서 바로 포드 생성 응답을 받지 못한 경우에 Watch를 통해 expectations를 해결할 수 있도록 eventHandler에도 `expectations.CreationObserved()`를 사용하는 것을 확인할 수 있습니다.

하지만 이 Expections는 생성/삭제만 처리가 가능하고 상태 변경 등의 작업은 커버할 수 없다는 한계가 있습니다. 또 `ExpectCreations`는 기댓값을 횟수로만 지정할 수 있어서 특정 UID를 기반으로 설정할 수 없다는 문제점도 있습니다(delete는 있음). 
[LINE에서 선언형 DB as a Service를 개발하며 얻은 쿠버네티스 네이티브 프로그래밍 기법 공유](https://2022.openinfradays.kr/session/3) 세션 뒷부분을 보면 Expectations를 확장해서 기댓값을 처리한 사례가 소개되어있습니다.

## Optimistic Locking(resourceVersion)

이 부분은 다른 글들에서도 잘 설명되어 있어서 간단하게만 정리하고 넘어가려 합니다.

모든 쿠버네티스 리소스는 `metadata.resourceVersion`를 포함하고 있고 이 버전 정보를 통해 API 서버에서는 낙관적 동시성 제어(optimistic concurrency control)을 수행합니다.

컨트롤러에서 조정 작업 수행 시 리소스의 상태를 수정하기 위한 절차는 다음과 같습니다.

1. Lister(캐시)에서 리소스 조회
2. 리소스 Deepcopy
   - Informer에 의해 캐싱되어 있는 객체이기 때문에 직접 수정하면 안됨
3. 리소스 수정
4. API 서버로 Update 요청

```go
// https://github.com/kubernetes/sample-controller/blob/master/controller.go#L249
func (c *Controller) syncHandler(ctx context.Context, key string) error {
	// ...	
	foo, err := c.foosLister.Foos(namespace).Get(name)
	// ...
	err = c.updateFooStatus(foo, deployment)
	// ...
}

// https://github.com/kubernetes/sample-controller/blob/master/controller.go#L329
func (c *Controller) updateFooStatus(foo *samplev1alpha1.Foo, deployment *appsv1.Deployment) error {
	// NEVER modify objects from the store. It's a read-only, local cache.
	// You can use DeepCopy() to make a deep copy of original object and modify this copy
	// Or create a copy manually for better performance
	fooCopy := foo.DeepCopy()
	fooCopy.Status.AvailableReplicas = deployment.Status.AvailableReplicas
	
	_, err := c.sampleclientset.SamplecontrollerV1alpha1().Foos(foo.Namespace).UpdateStatus(context.TODO(), fooCopy, metav1.UpdateOptions{})
	return err
}
```

수정/업데이트하는 리소스는 캐시에서 조회한 것이기 때문에 항상 최신값인 것이 아닙니다. 당연히 최신이 아닌 버전의 리소스를 조회하고 수정할 가능성이 있지만, API 서버에서 resourceVersion을 통해서 낙관적 동시성 제어를 해주기 때문에 걱정할 필요는 없습니다. 

# Godis Controller

사실 이 글의 메인은 Godis Controller인데... '앞부분이 너무 긴 것 같네 <-> 개념 정리인데 좀 간략하네'의 무한 굴레에 빠졌었습니다. 그래도 정리하려 했던 내용은 다 쓴 것 같아서 다음으로 넘어가려합니다.

서론에 써두었지만 Godis Controller의 목표는 yaml 파일로 godis 클러스터를 배포, 확장, 축소할 수 있도록 자동화하는 것이었습니다. 처음엔 StatefulSet으로 가능한지 고려해보았지만, Godis에서는 멤버십이 변경될 떄 기존 클러스터에게 요청을 보내야하기 때문에 로직을 자유롭게 작성할 수 있는 커스텀 컨트롤러로 결정했습니다.

## Godis 가내수공업 배포

먼저 가내수공업으로 어떻게 Godis 클러스터를 생성/확장/축소하는지 알아보겠습니다.

먼저 Godis는 다음과 같이 실행합니다.

```bash
# node1
$ godis cluster --id 1 \
--listen-client http://0.0.0.0:6379 \
--listen-peer http://127.0.0.1:6300 \
--initial-cluster 1@http://127.0.0.1:6300,2@http://127.0.0.1:16300,3@http://127.0.0.1:26300 \
--waldir /some/wal/path \
--snapdir /some/sanp/path \
--join false
```

- id : 클러스터 내에서 유니크한 id, 정수값
- initial-cluster : 클러스터 멤버십(id, peerURL 쌍)
- waldir, snapdir : wal, snaphost을 저장할 디렉토리 경로
- join : 기존 클러스터에 참가하면 true, 초기 클러스터 구성은 false

각 노드들은 `initial-cluster`로 전달받은 peer들을 멤버십에 추가한 뒤에 Raft 알고리즘에 따라 리더를 선출합니다. 따로 디스커버리 서비스가 없어서 초기 클러스터 멤버십을 전달하는 방법이 `initial-cluster` 뿐이라 빠트린 것이 없는지 잘 확인해야 합니다.

기존 클러스터에 새로운 노드를 추가 or 기존 노드를 제거할 떄는 클러스터 멤버십을 수동으로 변경해주어야 합니다. 멤버십 변경은 따로 클라이언트에서 `cluster meet`, `cluster forget` 커맨트로 수행합니다.

```bash
# id=4, peerURL=http://127.0.0.1:36300인 새로운 노드 멤버십에 추가 -> 새로운 노드 실행
$ cluster meet 4 http://127.0.0.1:36300

# id=1인 노드 프로세스 중지 -> 멤버십에서 노드 제거
$ cluster forget 1 
```

## CRD 및 컨트롤러 설계

### Replicas

Godis에는 Raft 알고리즘 관련 설정 이외에 성능 관련 설정이 없습니다. 그래서 배포할 때 신경써야 할 것이 실행할 노드 개수밖에 없습니다. 즉 `CRD.spec`에 들어갈 정보가 `replicas`밖에 없습니다...

처음 `.spec.replicas=3`으로 CRD를 생성하면 컨트롤러는 3개의 노드로 구성된 클러스터를 생성하고, 이후 값을 변경하면 노드 추가/제거를 수행하는 컨트롤러를 작성해야 합니다.

노드 추가/제거 시에 주의해야 할 점은 한번에 많은 멤버을 변경하게되면 기존 클러스터가 quorum을 잃을 수 있다는 것입니다. 만약 `replicas=3 -> replicas=6`으로 변경해서 멤버십을 6개의 노드로 변경된 상황이라면 3개의 새로운 노드 프로세스가 실행될때까지 기존 클러스터는 quorum을 잃고 요청을 처리할 수 없게 됩니다. 

그래서 `.spec.replicas`가 3에서 6으로 변경되더라도 `4 -> 5 -> 6`으로 순차적으로 확장되도록 컨트롤러를 작성해야 합니다.

### ID

노드를 생성할 때는 식별자에 ID를 접미사로 붙여서 리소스를 생성하도록 했습니다. ID는 1부터 새로운 노드가 추가될때마다 순차적으로 증가하도록 했는데, 이전에 제거되었던 노드의 ID는 재사용하지 않도록 구성했습니다.

```
ID() --(replicas=3 생성)--> ID(1, 2, 3) --(replicas=2 수정)--> ID(1, 2) --(replicas=4 수정)--> ID(1, 2, 4, 5)
```

만약 ID를 재사용한다면, 제거->생성이 짧은 시간 내에 이뤄진 경우에 해당 리소스가 제거해야 할 리소스인지, 새로 생성된 리소스인지 판단하기 힘듭니다. 결정적으로 raft 로그 복제 도중에 같은 ID로 제거된 멤버십 변경 로그가 복제되어서 자신이 제거된 것이 아닌데도 제거되었다고 판단하고 죽어버리기 때문에 재사용할 수 없었습니다.

### CRD, Resources

쿠버네티스 상에서 Godis 노드를 배포하기 위해서 고려해야 했던 점은

- 포드 재시작
- WAL, Snapshot 파일 저장
- Peer간 통신 엔드포인트

였습니다.

먼저 포드가 예상치 않게 중단된 경우 새로운 포드를 생성해야 하는데 이를 컨트롤러가 직접 제어할지, ReplicaSet을 이용할지 선택해야 했습니다. 

Godis는 WAL, Snapshot을 통한 재시작 매커니즘을 지원하기 때문에 같은 설정값으로 단순하게 재시작을 수행해도 문제가 없었습니다. 그래서 굳이 포드의 중단을 직접 추적하기보단 `replicas=1`인  ReplicaSet을 생성하는 방법을 선택했습니다. 또, 포드가 다시 생성됐을 때 이전 포드가 저장한 WAL, Snapshot 파일을 읽어야 하기 때문에 이를 위한 PVC도 추가했습니다(일반 데이터는 메모리에 저장함). 

Peer간 통신에서도 포드IP를 그대로 사용하지 않고 포드가 재시작되어도 같은 엔드포인트를 사용할 수 있도록 서비스를 추가했습니다.

<img width="687" alt="image" src="https://github.com/KumKeeHyun/godis/assets/44857109/13694197-29e2-49dc-b744-e9e84d0e1ce6">

이 구조로 여러 노드를 생성하게 되면 아래와 같은 형태가 됩니다. 

<img width="1137" alt="image" src="https://github.com/KumKeeHyun/godis/assets/44857109/977b5310-11b9-4ac4-8880-30db3f6c2c16">

GodisCluster는 replicas와 현재 배포된 노드들만 확인하고 새로 노드를 생성할지, 기존 노드를 제거할지만 컨트롤하게 하고싶었는데, 노드에 묶여있는 리소스들이 많아서 원래 역할에 집중하지 못할 것 같았습니다. 

<img width="1302" alt="image" src="https://github.com/KumKeeHyun/godis/assets/44857109/56c1c2bf-895f-4660-a24d-33182e1e402d">

그래서 클러스터 조정과 노드 배포를 분리하기 위해서 중간에 하나의 노드를 대변하는 리소스를 추가했습니다. GodisCluster 이벤트를 수신하는 핸들러는 `.spec.replicas`에 따라 Godis 리소스를 생성/제거하고, Godis가 ReplicaSet, Service, PVC 생성을 책짐지는 구조입니다.

### initial-cluster

Godis를 실행할 때 클러스터 멤버십을 전달하는 `initial-cluster` 설정값은 계속해서 변경됩니다. 그래서 재시작하는 포드는 이전에 전달해주었던 initial-cluster 값을 그대로 사용할 수 없습니다. 

클러스터 맴버십이 변경되더라도 재시작되는 포드에게 옳바른 initial-cluster를 전달할 수 있도록, ConfigMap을 추가했습니다.

<img width="1302" alt="image" src="https://github.com/KumKeeHyun/godis/assets/44857109/93886105-d05c-4c7c-a3cb-f8fd50ece9dc">

## SyncHandler 구현

쿠버네티스 상에서 각 노드를 어떻게 배포할지 정했으니 이제 그걸 컨트롤러로 자동화하는 일만 남았습니다. 조정 작업에서 수행해야 하는 작업은 크게 3가지입니다.

- 새로운 클러스터 생성
- 클러스터 확장
- 클러스터 축소

각 조정 작업이 어떻게 이뤄지는지 보기전에 어떤 조정 작업이 필요한지 판단하는 부분부터 살펴보겠습니다.

### Status

앞서 설명했던 것처럼 컨트롤러에서 리소스 조회는 캐시에서 이뤄지기 때문에 API 서버보다 이전의 상태를 기반으로 조정 작업을 하게 될 수 있습니다. 이 때문에 조정 작업 시 기대했던 것보다 더 많은 리소스가 생성/제거될 수 있습니다. 

Godis는 상태 기반 프로세스이고 멤버십이 변경될 때 오버헤드(스냅샷, 복제 로그 전송)가 발생하기 때문에 최대한 이런 상황을 피하고자 했습니다.

<img width="1828" alt="image" src="https://github.com/KumKeeHyun/godis/assets/44857109/90243783-7ac7-4483-b200-9ce8fea7566f">

먼저 `.spec.replicas`에 따라 Godis를 생성/제거하는 작업은 ReplicaSet 컨트롤러가 하는 일과 비슷해서, 해당 컨트롤러가 사용하는 Expectations를 이용하는 것을 고려했었습니다. 하지만 Godis는 랜덤한 식별자로 생성되는 것이 아니라 특정한 ID로 생성되어야 하고, 생성/제거 시 다른 노드들에게 멤버십 변경을 알려야 하기 때문에 단순히 몇 개가 생성/제거되었는지 판단하는 Expectations는 부적합하다고 판단했습니다.

<img width="681" alt="image" src="https://github.com/KumKeeHyun/godis/assets/44857109/7b9eba59-8a14-4ef6-9177-9e82e62e9838">

결국 GodisCluster의 `.status` 필드에 진행 상황과 해야할 작업(생성/제거할 ID)를 저장하고, 도중에 실패하더라도 다음 루프에서 진행 상황을 읽어서 작업을 이어나갈 수 있도록 구성했습니다. 또 이전 조정 작업이 정상적으로 끝났는지 확인한 후에 다음 조정 작업을 시작하도록 구성했습니다.

이제 남은 것은 각 조정 작업이 멱등성을 만족하는지 확인하는 것 입니다.

### 클러스터 생성

1. status를 Initializing으로 변경
   - 이미 Initializing인 경우 스킵
   - 동시에 여러 수정이 이뤄져도 resourceVersion 덕분에 하나의 요청만 성공함
2. 초기 멤버십 구성을 위한 initial-cluster를 ConfigMap에 등록
   - Create 요청 이후 IsAlreadyExists 검사
3. replicas 값만큼 Godis 생성
   - Create 요청 이후 IsAlreadyExists 검사
4. status를 Running으로 변경
   - 동시에 여러 수정이 이뤄져도 resourceVersion 덕분에 하나의 요청만 성공함

### 클러스터 확장

1. status를 Scaling으로 변경, 새로 생성할 노드의 ID 등록
   - 이미 Scaling인 경우 스킵
   - 동시에 여러 수정이 이뤄져도 resourceVersion 덕분에 하나의 요청만 성공함
   - 이후에 중단되고 다른 루프에서 처리되더라도 새로운 ID를 생성하지 않고 등록한 ID로 노드 생성 진행
2. 새로운 멤버십의 initial-cluster를 ConfigMap에 등록
   - Update라서 여러번 요청해도 문제 없음
3. 기존 클러스터에 멤버십 변경 요청
   - 같은 ID로 여러번 멤버십 변경 요청해도 문제 없음
4. Godis 생성
   - Create 요청 이후 IsAlreadyExists 검사
5. status를 Running으로 변경
   - 동시에 여러 수정이 이뤄져도 resourceVersion 덕분에 하나의 요청만 성공함

### 클러스터 축소

1. status를 Scaling으로 변경, 제거할 노드의 ID 등록
   - 이미 Scaling인 경우 스킵
   - 동시에 여러 수정이 이뤄져도 resourceVersion 덕분에 하나의 요청만 성공함
   - 이후에 중단되고 다른 루프에서 처리되더라도 제거할 ID를 다시 선택하지 않고 등록한 ID로 노드 제거 진행
2. 새로운 멤버십의 initial-cluster를 ConfigMap에 등록
   - Update라서 여러번 요청해도 문제 없음
3. Godis 삭제
   - Delete 요청 이후 IsNotFound 검사
4. 기존 클러스터에 멤버십 변경 요청
   - 같은 ID로 여러번 멤버십 변경 요청해도 문제 없음
5. status를 Running으로 변경
   - 동시에 여러 수정이 이뤄져도 resourceVersion 덕분에 하나의 요청만 성공함

### 단계적 스케일링

앞서 설명했던 것처럼 클러스터 확장/축소는 한번에 하나의 노드만 작업하도록 계획했습니다. 

이를 위해서 확장/축소 작업은 1개 노드 단위로 작업하도록 구현하고, 작업이 끝난 후에도 남아있는 작업이 있다면 queue에 다시 key를 넣어서 다음 루프에서 처리되도록 구현했습니다.

```go
func (c *Controller) syncCluster(ctx context.Context, key string) error {
	// ...

	if requiresInitialize(cluster, configNotExists) {
		// ...
	} else if requiresScaleOut(cluster, godises) {
		// ...
	} else if requiresScaleIn(cluster, godises) {
		// ...
	}
	if err != nil {
		return err
	}
	if cluster.Spec.Replicas != nil && *cluster.Spec.Replicas != cluster.Status.Replicas {
		// ...
		c.clusterQueue.AddAfter(key, requeueDelay)
	}

	return nil
}
```

조정 작업 단위를 1로 고정해 두었기 때문에, 많은 수의 조정 작업을 예약한 경우에 최종 상태까지 도달하는 시간이 많이 걸릴 수 있습니다. 하지만 단일 raft 그룹에서 복제 노드를 10개 이상으로 구성하는 경우는 거의 없기 때문에 문제가 없을 것이라 판단했습니다.

## 결과물

[godis github repo](https://github.com/KumKeeHyun/godis)

### 클러스터 생성

이제 3개의 노드로 구성된 클러스터를 생성해보겠습니다. 클러스터 생성은 아래와 같이 GodisCluster 리소스를 선언해주면 끝입니다!

```yml
apiVersion: kumkeehyun.github.com/v1
kind: GodisCluster
metadata:
  name: example-godis
spec:
  name: example-godis
  replicas: 3
```

![initializ-cluster](https://github.com/KumKeeHyun/godis/assets/44857109/646ad7a3-87bd-4beb-9728-dccfc3c2d1e0)

GIF를 유심히 보시면 `example-godis-1-cqmgs`, `example-godis-2-x2cj8`, `example-godis-3-6vcn2` 총 3개의 포드가 생성된 것을 볼 수 있습니다. 

마지막에 출력된 로그에서는 ID가 2인 `example-godis-2-x2cj8`가 리더로 선출된 것도 확인할 수 있습니다.

### 클러스터 확장

클러스터를 수동으로 확장하려면 기존 클러스터에 멤버십 변경 요청을 보내고 새로운 노드에 설정값을 신경써서 넣어준뒤 실행시켜야 합니다. 이제는 번거로운 작업 없이 `.spec.replicas`를 증가시켜주기만 하면 됩니다.

![scaleout](https://github.com/KumKeeHyun/godis/assets/44857109/634ea8fc-6b2e-45e9-a642-caf1b10739f2)

GIF를 유심히 보시면 스펙을 3에서 4로 수정했더니 `example-godis-4-8bmzs` 포드가 생성된 것을 확인할 수 있습니다. describe 명령으로 출력된 정보의 마지막 이벤트 부분을 보면 ID=4인 Godis를 생성했다는 로그도 볼 수 있습니다.

### 클러스터 축소

클러스터 노드 수를 줄이고 싶다면 확장 때와 마찬가지로 `.spec.replicas`를 감소시켜주기만 하면 됩니다.

![scalein](https://github.com/KumKeeHyun/godis/assets/44857109/397de1d2-295e-4a25-94cd-a026d1b7a02f)

GIF를 유심히 보시면 스펙을 4에서 3로 수정했더니 `example-godis-1-cqmgs` 포드가 제거된 것을 확인할 수 있습니다. describe 명령으로 출력된 정보의 마지막 이벤트 부분을 보면 ID=1인 Godis를 제거했다는 로그도 볼 수 있습니다. 실행중인 포드의 로그도 클러스터 멤버십이 (2,3,4)로 변경된 것을 볼 수 있습니다.

### 포드 중단 및 자동 복구

만약 어떤 이유로 포드가 제거된 경우 ReplicaSet에 의해 자동으로 복구됩니다. 이때 새로 생성된 포드는 이전 포드의 WAL, Snapshot을 복구한 뒤에 기존 클러스터에 참가하게 됩니다.

![pod-autohealing](https://github.com/KumKeeHyun/godis/assets/44857109/b0802f8b-9019-4864-9337-025f6d7ed1fd)

GIF를 유심히 보시면 kubectl delete를 이용해서 `example-godis-4-8bmzs` 포드를 제거했더니 `example-godis-4-24dr7` 포드가 생성된 것을 볼 수 있습니다. 해당 포드의 로그를 보면 이전 포드의 WAL을 잘 복구하고 최종 멤버십이 (2,3,4)로 조정된 것을 확인할 수 있습니다.

이때 Godis 컨트롤러는 ReplicaSet이 알아서 조정해줄 것을 기대하고 아무런 작업을 하지 않습니다.

### 노드 제거 및 자동 복구

만약 수동으로 특정 노드를 제거하고 싶다면 kubectl delete로 해당 Godis 리소스를 제거하면 됩니다. 그럼 Godis 컨트롤러가 해당 노드가 제거된 것을 감지하고 `.spec.replicas`와 같은 수의 노드를 실행시키기 위해 새로운 Godis를 생성합니다.

![godis-autohealing](https://github.com/KumKeeHyun/godis/assets/44857109/db9a535b-45cb-47e6-ae63-8eaf4a88dbd9)

GIF를 유심히 보시면 kubectl delete를 이용해서 `example-godis-3` Godis 리소스를 제거했더니 `example-godis-5` Godis와 `example-godis-5-wzs6h` 포드가 생성된 것을 볼 수 있습니다. `example-godis-5-wzs6h` 포드의 로그를 보면 최종 멤버십이 (2,4,5)로 잘 조정된 것도 확인할 수 있습니다.

# 마치면서

이전에 쿠버네티스 컨트롤러에 대해서 개념만 공부했을 때는 간단하다고 생각했었는데, 막상 직접 구현하려고 해보니 고려해야 할 부분이 너무 많아서 좀 당황했었습니다. 과연 완성은 할 수 있을지 걱정했었는데, 어찌어찌 구현하고 의도했던 대로 작동하는 컨트롤러를 확인하고 나름의 성취감을 느꼈습니다.

![고인물짤](https://i.namu.wiki/i/3Rdx9AjAMNhgvidfsjbEECkyeRsHOSkTSUO6LG-etXjn70DXNywOXvD9hmLY2uIwvTHQdGki_lZ-8U4eozhE3POxk6tsgUGUKI16eN5EwIZymiGGugqqhvP6VU8xAaTcLL4zIPq-px9NpbQDphzmfQ.webp)

또 컨트롤러 작성에서 adoption, orphaning 관련된 부분도 다루려 했는데 풀어내지 못해 아쉬움이 남습니다.

그래도 구현에 필요한 개념들을 공부하면서 어렴풋이 알고 있던 개념들을 정리해볼 수 있었고, 단순히 컨테이너 오케스트레이션으로만 생각하고 있던 쿠버네티스를 좀 더 넓은 시각으로 바라볼 수 있게 된 계기가 되었습니다.

글을 읽어주셔서 감사합니다.

## Reference

- https://www.oreilly.com/library/view/kubernetes-in-action/9781617293726/
- https://engineering.linecorp.com/ko/blog/declarative-cloud-db-service-using-kubernetes
- https://www.getoutsidedoor.com/2020/05/09/kubernetes-controller-%EA%B5%AC%ED%98%84%ED%95%B4%EB%B3%B4%EA%B8%B0/
- https://www.youtube.com/watch?v=SWD__6nhLic&t=1770s
- https://learnk8s.io/etcd-kubernetes