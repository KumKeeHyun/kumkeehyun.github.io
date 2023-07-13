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
- [Godis Controller](#godis-controller)
- [마치면서](#마치면서)
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

<img width="455" alt="image" src="https://github.com/KumKeeHyun/raspi-cluster/assets/44857109/9855d1c0-5245-4ef1-a0ae-1ded1149b8d4">

`etcd`, `API Server`, `Scheduler`, `Controller Manager`는 쿠버네티스 컨트롤 플레인의 가장 기본적인 구성요소 입니다. 여기서 `API 서버`의 역할은 다음과 같습니다.

- RBAC를 통해 인가된 API 요청인지 판단
- 요청 payload가 유효한지 검사
- 요청에 따라 etcd에 저장 및 조회
- 리소스가 변경되면 변경된 사항을 클라이언트들에게 전달

문서의 요약문에는 API 서버는 쿠버네티스에서 중추 역할을 한다고 설명되어 있어서, kubectl로 포드 생성을 요청하면 API Server에서 특정 노드의 kubelet에게 포드 생성 요청을 보낼 것이라고 생각할 수 있습니다.

하지만 API 서버가 정작 하는 일은 사용자 요청에 따라 etcd에 CRUD 작업을 수행하고, 특정 리소스 변경을 구독하는 클라이언트들에게 변경 사항을 알려주는 것이 전부입니다. 즉, API 서버는 etcd에 새로운 포드의 정보를 저장하고, 포드 변경 사항 이벤트를 구독하고있는 클라이언트들에게 전달하는 작업만 담당합니다. 그렇다면 kubelet은 어떻게 포드를 생성하는 걸까요?

<img width="1126" alt="image" src="https://github.com/KumKeeHyun/raspi-cluster/assets/44857109/1cba075c-0a81-44e9-9966-523a400c896b">

kubelet이 새로 생성할 포드 정보를 받는 과정을 간단하게 정리한 그림입니다.

0. 스케줄러와 kubelet은 포드 변경 사항을 전달받기 위해 API 서버에 Watch 요청
1. API 서버가 새로운 포드 생성 요청을 받고 etcd에 새로운 포드 정보를 저장
2. API 서버가 새로운 포드 정보를 Watch 클라이언트인 스케줄러와 kubelet에게 전달
3. 스케줄러가 새로운 포드를 보고 할당될 노드를 선택한 뒤에 포드 정보를 업데이트
4. API 서버는 스케줄러가 요청한 업데이트 정보를 etcd에 저장하고 변경 사항을 Watch 클라이언트들에게 전달
5. kubelet은 새로 할당된 포드를 보고 컨테이너 생성

이처럼 쿠버네티스는 Watch 메커니즘을 통해서 각 컴포넌트는 새로 선언된 정보를 전달받고 해당 정보에 수렴하도록 다른 리소스들을 조정하는 흐름으로 작동합니다. API 서버는 이런 컴포넌트들에게 일관된 진입점을 제공해주고 추가적으로 권한 검사, 유효성 검사, 일관성 검사(낙관적 락) 등의 작업도 수행합니다.

## Controller Manager

쿠버네티스에는 포드 이외에 ReplicaSet, Deployment, Service... 등의 여러 리소스가 있습니다. Deployment를 생성하면 ReplicaSet, Pod가 자동으로 생성되는데 이처럼 현재 상태가 선언된 상태로 수렴하도록 조정하는 작업을 수행하는 것이 컨트롤러입니다.

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

- 리소스 조회시 API 서버로 요청을 하는 것이 아니라 Informer(Watch)에 의해 관리되는 캐시를 사용
- 이벤트를 전달할 때 리소스 전체를 전달하는 것이 아니라 key만 전달

한다는 것입니다.

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

SyncHandler를 작성할때 주의해야 하는 점은 리소스 조회시 API 서버가 아닌 캐시를 조회한다는 것입니다. API 서버의 최신 상태가 반영되지 않은 상태에서 조정 작업을 수행하면 뜻하지 않은 작업이 수행될 수 있습니다.

예를 들어, 아래 그림처럼 API 서버에는 spec에 맞게 3개의 포드가 생성된 상태이지만 아직 캐시에 반영이 되지 않아서 불필요한 포드를 생성하는 상황이 생길 수 있습니다.

<img width="1172" alt="image" src="https://github.com/KumKeeHyun/raspi-cluster/assets/44857109/480e2fe3-3d39-4c3f-a96d-f402c2e00cdd">

사실 시간이 지나면 다시 캐시가 API 서버와 동기화되면서 불필요한 포드가 생성된 것을 확인하고 해당 포드를 제거할 수 있기 때문에, 일반적인 경우에는 큰 문제가 되지 않습니다. 하지만 이런 불필요한 생성/삭제 작업이 어플리케이션의 특성에 따라서 기존 포드들에 큰 영향(데이터베이스 -> id 충돌, scale out 조정 작업...)을 끼칠 수도 있습니다.

쿠버네티스에서는 Create 작업같이 멱등성이 없는 작업을 위해서 Expectations이라는 유틸을 사용합니다. 사용하는 방법은 다음과 같습니다.

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

ReplicaSet 컨트롤러의 SyncHandler를 보면 조정작업을 하기 전에 `expectations.SatisfiedExpectations()`을 통해 생성/삭제했지만 아직 확인되지 않는 포드가 있다면 추가적인 조정 작업을 수행하지 않는 것을 확인할 수 있습니다.

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

하지만 이 Expections는 생성/삭제만 처리가 가능하고 상태 변경 등의 작업은 커버할 수 없다는 한계가 있습니다. 또 `ExpectCreations`시 기댓값을 횟수로만 지정할 수 있어서 특정 UID를 기반으로 설정할 수 없다는 문제점도 있습니다(delete는 있음). 
[LINE에서 선언형 DB as a Service를 개발하며 얻은 쿠버네티스 네이티브 프로그래밍 기법 공유](https://2022.openinfradays.kr/session/3) 세션 뒷부분을 보면 Expectations를 확장해서 기댔값을 처리한 사례가 소개되어있습니다.

# Godis Controller

# 마치면서