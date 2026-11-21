---
title: "Kubernetes v1.37, 갱 스케줄링을 자체 내장으로 마쳤다 — Workload·PodGroup API 베타와 CompositePodGroup 분석"
date: "2026-11-21"
keywords: ["Kubernetes v1.37", "갱 스케줄링", "PodGroup", "CompositePodGroup", "AI 분산학습"]
lang: "ko"
description: "Kubernetes v1.37에서 Workload·PodGroup API가 베타로 승격되고 CompositePodGroup로 다단계 갱 스케줄링이 가능해졌다. Volcano·Kueue 의존 없이 쓰는 네이티브 갱 스케줄링의 현재 상태와 실전 YAML을 정리한다."
---

# Kubernetes v1.37, 갱 스케줄링을 자체 내장으로 마쳤다 — Workload·PodGroup API 베타와 CompositePodGroup 분석

분산 학습 잡 하나를 Kubernetes에 올린 적 있다면 이 장면을 알 것이다. 워커 8개 중 7개만 노드에 할당되고 나머지 1개가 Pending에 걸리면, 이미 떠 있는 7개가 자원만 잡아먹은 채 하염없이 대기한다. 학습은 시작조차 못 하고, 클러스터 자원만 고갈된다. 이 문제를 푸는 "전부 아니면 전무(all-or-nothing)" 스케줄링, 즉 갱 스케줄링(gang scheduling)은 지금까지 Kubernetes 코어가 아니라 Volcano, Kueue, KAI Scheduler 같은 외부 프로젝트의 영역이었다.

그 경계가 무너지고 있다. Kubernetes는 v1.35에서 Workload API를 알파로 처음 선보인 뒤, v1.36(2026년 4월)에서 PodGroup API와 전용 스케줄링 사이클을 도입했고, v1.37(2026년 8월 26일 릴리스, 코드명 Garhwal)에서 마침내 Workload·PodGroup API와 갱 스케줄링을 **베타로 승격**했다. 이 글에서는 이 "워크로드 인지 스케줄링(Workload-Aware Scheduling, WAS)"이 세 버전에 걸쳐 어떻게 자리를 잡았는지, 무엇이 쓸 만해지고 무엇이 아직 아닌지를 정리한다.

## 왜 코어에 갱 스케줄링이 필요했나

기존 kube-scheduler는 파드를 하나씩 큐에서 꺼내 노드에 배치한다. 파드 단위 최적화는 일반 워크로드엔 충분하지만, 모든 멤버가 동시에 떠야 의미가 있는 분산 학습·배치 잡에는 치명적이다. 부분 할당이 반복되면 자원 파편화와 데드락이 교차하며 클러스터 처리량이 급락한다.

그래서 실무에서는 Volcano의 `PodGroup`이나 Kueue의 큐 관리로 이를 메워왔다. 다만 이 접근은 스케줄러 외부에서 상태를 조작하는 방식이라, 선점(preemption), 토폴로지 배치, GPU 자원 할당(DRA) 같은 코어 기능과의 정합성을 맞추기 어려웠다. WAS 작업 그룹(WG WAS)의 목표는 이것을 코어 스케줄러 안으로 끌어들이는 것이었다.

## v1.36이 세운 기반: 템플릿과 런타임의 분리

v1.36의 핵심 설계 결정은 역할 분리다. `Workload` API는 **정적 템플릿**, `PodGroup` API는 **런타임 상태**를 담당한다. 컨트롤러는 Workload에 정의된 `podGroupTemplates`로부터 실제 PodGroup 객체를 찍어내고, 파드는 `spec.schedulingGroup.podGroupName`으로 자기 그룹을 가리킨다. 스케줄러는 Workload를 볼 필요 없이 PodGroup만 읽으면 되므로 상태 갱신도 그룹 단위로 샤딩할 수 있어 확장성이 좋아진다.

같은 릴리스에서 kube-scheduler에 **PodGroup 전용 스케줄링 사이클**이 들어왔다. 큐에서 그룹 멤버를 꺼내면 나머지 멤버를 함께 모아 정렬하고, 클러스터 상태 스냅샷 하나를 기준으로 그룹 전체의 배치를 원자적으로 시도한다. 성공하면 만족한 멤버 전체가 한 번에 바인딩되고, 실패하면 그룹 전체가 백오프 후 재시도한다. 갱 스케줄링은 이 사이클 위에서 `minCount`(최소 동시 실행 파드 수)를 검증하는 방식으로 동작한다.

토폴로지 인지 스케줄링(TAS)과 워크로드 인지 선점도 이 버전에서 첫 모습을 보였다. TAS는 `schedulingConstraints.topology`에 랙 같은 토폴로지 키를 지정해 그룹을 특정 도메인 안에 함께 배치하고, 워크로드 인지 선점은 그룹 전체를 하나의 선점 주체로 취급해 여러 노드에 걸쳐 희생자를 동시에 확보할 수 있게 한다.

## v1.37의 도약: 베타 승격과 세 가지 실무적 변화

v1.37에서 Workload·PodGroup API는 `v1beta1`로 승격됐다. GA까지 한 단계 남은 셈이다. 공식 블로그가 강조하는 변화 중 실무에 직접 닿는 것만 추리면 다음과 같다.

**첫째, PodGroup이 큐의 1급 시민이 됐다.** 이전에는 그룹 소속 파드라도 개별적으로 큐에 올랐다. 이제는 최상위 PodGroup 객체만 큐에 들어가므로 그룹원 전체가 동일한 큐잉 동작을 보장받는다.

**둘째, `minCount`가 가변이 됐다.** 갱의 최소 크기를 중간에 줄이거나 늘릴 수 있어, 이미 스케줄된 파드를 중단하지 않고 탄력적으로 규모를 조정할 수 있다. 학습 도중 노드 이탈로 최소 요구치를 낮춰야 하는 상황에서 유용하다.

**셋째, `disruptionMode` 필드명이 정리됐다.** `PodGroup`은 `all`로, `Pod`는 `single`로 바뀌었다. 일반 선점(default preemption)도 이제 그룹의 disruptionMode를 존중해서, `all: {}`로 지정된 그룹의 파드 하나만 뽑아가는 사고가 사라졌다. 단, 알파 버전을 쓰던 클러스터는 `v1alpha2`가 통째로 `v1alpha3`으로 교체되는 파괴적 변경을 감수해야 한다.

Job 컨트롤러 통합도 눈에 띈다. v1.36의 통합은 `parallelism > 1`, `completionMode: Indexed`, `completions == parallelism`이라는 "모양이 확정된" 잡에만 자동 적용됐다면, v1.37에서는 Job에 `.spec.scheduling` 필드가 생겨 정책을 명시적으로 선언할 수 있다:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: distributed-training-job
spec:
  parallelism: 8
  completions: 8
  scheduling:
    schedulingPolicy:
      gang: {}   # minCount 생략 시 parallelism(8)이 기본값
    schedulingConstraints:
      topology:
        - key: topology.kubernetes.io/zone
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: worker
          image: registry.example/trainer:latest
```

`.spec.scheduling`을 생략하면 기존과 동일한 파드 단위 스케줄링이므로, 기존 잡에 영향 없이 점진적으로 도입할 수 있다. 생성 후 변경은 불가하지만 `schedulingPolicy.gang.minCount`만 예외적으로 수정 가능하다.

## CompositePodGroup: JobSet·LWS를 위한 계층 구조

v1.37의 가장 야심적인 추가는 **CompositePodGroup API**(알파, `scheduling.k8s.io/v1alpha3`)다. 실제 AI 워크로드는 평면 그룹 하나가 아니라 "드라이버 1개 + 워커 N개"의 LeaderWorkerSet 구성, JobSet처럼 여러 잡의 묶음처럼 계층적으로 생겼다. CompositePodGroup은 PodGroup들(그리고 다른 CompositePodGroup)을 트리로 엮어 그 전체를 하나의 스케줄링 단위로 다룬다.

```yaml
apiVersion: scheduling.k8s.io/v1beta1
kind: Workload
metadata:
  name: multi-level-tas-workload
spec:
  compositePodGroupTemplates:
    - name: root
      schedulingPolicy:
        gang:
          minGroupCount: 2
      schedulingConstraints:
        topology:
          - key: topology.kubernetes.io/zone   # 전체는 한 가용 영역 안에
      podGroupTemplates:
        - name: workers
          schedulingPolicy:
            gang: { minCount: 8 }
          schedulingConstraints:
            topology:
              - key: topology.example.com/rack  # 워커는 랙 단위로 뭉치기
        - name: driver
          schedulingPolicy:
            gang: { minCount: 1 }
```

이 예시에서 스케줄러는 하향식(top-down)으로 제약을 푼다. 루트 그룹이 가용 영역을 정하면, 자식 그룹의 랙 후보는 그 영역 내부로 한정된다. "학습 전체는 AZ 하나 안에, 워커들은 랙 단위로 공배치" 같은 요구를 외부 스케줄러 없이 표현할 수 있게 된 것이다.

컨트롤러 생태계를 위한 배려도 있다. `WorkloadPodGroup*`, `WorkloadCompositePodGroup*` 프리픽스의 공통 빌딩 블록 타입과 `workloadbuilder` Go 라이브러리(kubernetes/component-helpers 저장소)가 제공되어, 임의의 컨트롤러가 자기 API 필드 이름을 유지한 채 같은 스케줄링 원시형을 재사용할 수 있다. 검증은 기본 거부(deny-by-default) 방식이라 컨트롤러가 명시적으로 허용한 정책만 통과한다.

GPU 등 기기 할당과의 연결고리도 강화됐다. v1.34에서 GA가 된 DRA(Dynamic Resource Allocation) 위에 `DRAWorkloadResourceClaims` 기능이 v1.37에서 베타가 되어, ResourceClaimTemplate으로부터 **PodGroup 전체가 하나의 ResourceClaim을 공유**할 수 있다. 그룹원 간 GPU 공유가 필요한 추론 워크로드에 유효하다.

## 도입 전 확인할 세 가지

**기본 비활성화.** 베타든 알파든 WAS 기능은 전부 기능 게이트를 수동으로 켜야 한다. `WorkloadWithJob`(Job 통합), `TopologyAwareWorkloadScheduling`(TAS), `CompositePodGroup`, `PodGroupPreemptionPolicy` 등 각 기능이 붙는 컴포넌트(kube-apiserver / kube-controller-manager / kube-scheduler)가 다르니 공식 문서의 게이트 표를 그대로 따르자. 특히 `CompositePodGroup`은 kube-controller-manager에서 켤 때 `TopologyAwareWorkloadScheduling`도 함께 켜야 한다.

**알파 버전 전환 주의.** `v1alpha2`에서 `v1alpha3`으로 넘어갈 때 disruptionMode 구조가 바뀌는 파괴적 변경이 있었다. 알파 단계에서 앞서 도입한 클러스터는 마이그레이션 계획을 세워야 한다. 이런 변동성은 GA 전까지 반복될 수 있다.

**아직 Volcano·Kueue를 대체하는 아니다.** 베타 승격이 진행 중이고 큐 관리·쿼터·선점 정책의 성숙도는 여전히 Kueue 쪽이 앞선다. 현실적인 판단은 이렇다. 지금 신규로 갱 스케줄링이 필요한 소규모 클러스터라면 네이티브 WAS를 테스트 클러스터에서 검증할 가치가 충분하다. 반면 대규모 ML 플랫폼은 Kueue의 큐·쿼터 체계와 네이티브 갱 스케줄링이 만나는 지점(v1.38 로드맵)을 지켜보며 이중화 전략을 유지하는 편이 안전하다.

## 핵심 요약

- Kubernetes v1.37(2026-08-26, Garhwal)에서 Workload·PodGroup API와 갱 스케줄링이 **v1beta1 베타**로 승격됐다. 전부 아니면 전무 스케줄링이 코어에 자리 잡는 중이다.
- PodGroup 큐잉 1급 시민화, `minCount` 가변화, `disruptionMode` 정리(`all`/`single`)로 실무 요구가 많이 반영됐다.
- CompositePodGroup(알파)은 JobSet·LeaderWorkerSet 같은 계층형 워크로드의 다단계 갱·토폴로지 제약을 네이티브로 표현한다.
- Job의 `.spec.scheduling` 필드로 기존 잡과 무관하게 점진 도입이 가능하다. DRA와의 연동(`DRAWorkloadResourceClaims`)도 베타가 됐다.
- 모든 WAS 기능은 기본 비활성화다. 첫 단계는 테스트 클러스터에서 `WorkloadWithJob` 게이트를 켜고 Indexed Job 하나로 갱 스케줄링을 직접 확인하는 것부터다.

## 참고 자료

- Kubernetes 블로그, "Kubernetes v1.37: Advancing Workload-Aware Scheduling" (2026-09-08) — https://kubernetes.io/blog/2026/09/08/kubernetes-v1-37-advancing-workload-aware-scheduling/
- Kubernetes 블로그, "Kubernetes v1.36: Advancing Workload-Aware Scheduling" (2026-05-13) — https://kubernetes.io/blog/2026/05/13/kubernetes-v1-36-advancing-workload-aware-scheduling/
- Kubernetes 블로그, "Kubernetes v1.37: Garhwal" 릴리스 발표 (2026-08-26) — https://kubernetes.io/blog/2026/08/26/kubernetes-v1-37-release/
- Kubernetes 블로그, "Kubernetes v1.34: DRA has graduated to GA" — https://kubernetes.io/blog/2025/09/01/kubernetes-v1-34-dra-updates/
