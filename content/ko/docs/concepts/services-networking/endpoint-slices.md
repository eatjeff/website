---
title: 엔드포인트슬라이스
feature:
  title: 엔드포인트슬라이스
  description: >
    쿠버네티스 클러스터에서 확장 가능한 네트워크 엔드포인트 추적.

content_template: templates/concept
weight: 10
---


{{% capture overview %}}

{{< feature-state for_k8s_version="v1.17" state="beta" >}}

_엔드포인트슬라이스_ 는 쿠버네티스 클러스터 내의 네트워크 엔드포인트를
추적하는 간단한 방법을 제공한다. 이것은 엔드포인트를 더 확장하고, 확장 가능한
대안을 제안한다.

{{% /capture %}}

{{% capture body %}}

## 엔드포인트슬라이스 리소스 {#endpointslice-resource}

쿠버네티스에서 EndpointSlice는 일련의 네트워크 엔드 포인트에 대한
참조를 포함한다. 쿠버네티스 서비스에 {{< glossary_tooltip text="셀렉터"
term_id="selector" >}} 가 지정되면 EndpointSlice
컨트롤러는 자동으로 엔드포인트슬라이스를 생성한다. 이 엔드포인트슬라이스는
서비스 셀렉터와 매치되는 모든 파드들을 포함하고 참조한다. 엔드포인트슬라이스는
고유한 서비스와 포트 조합을 통해 네트워크 엔드포인트를 그룹화 한다.
EndpointSlice 오브젝트의 이름은 유효한
[DNS 서브도메인 이름](/ko/docs/concepts/overview/working-with-objects/names/#dns-서브도메인-이름들)이어야 한다.

예를 들어, 여기에 `example` 쿠버네티스 서비스를 위한 EndpointSlice
리소스 샘플이 있다.

```yaml
apiVersion: discovery.k8s.io/v1beta1
kind: EndpointSlice
metadata:
  name: example-abc
  labels:
    kubernetes.io/service-name: example
addressType: IPv4
ports:
  - name: http
    protocol: TCP
    port: 80
endpoints:
  - addresses:
      - "10.1.2.3"
    conditions:
      ready: true
    hostname: pod-1
    topology:
      kubernetes.io/hostname: node-1
      topology.kubernetes.io/zone: us-west2-a
```

기본적으로, EndpointSlice 컨트롤러가 관리하는 엔드포인트슬라이스에는
각각 100개 이하의 엔드포인트를 가지고 있다. 이 스케일 아래에서 엔드포인트슬라이스는
엔드포인트 및 서비스와 1:1로 매핑해야하며, 유사한 성능을 가져야 한다.

엔드포인트슬라이스는 내부 트래픽을 라우트하는 방법에 대해 kube-proxy에
신뢰할 수 있는 소스로 역할을 할 수 있다. 이를 활성화 하면, 많은 수의 엔드포인트를 가지는
서비스에 대해 성능 향상을 제공해야 한다.

### 주소 유형

EndpointSlice는 다음 주소 유형을 지원한다.

* IPv4
* IPv6
* FQDN (Fully Qualified Domain Name)

### 토폴로지

엔드포인트슬라이스 내 각 엔드포인트는 연관된 토폴로지 정보를 포함할 수 있다.
이는 해당 노드, 영역 그리고 지역에 대한 정보가 포함된
엔드포인트가 있는 위치를 나타나는데 사용 한다. 값을 사용할 수 있으면
다음의 토폴로지 레이블이 엔드포인트슬라이스 컨트롤러에 의해 설정된다.

* `kubernetes.io/hostname` - 이 엔드포인트가 있는 노드의 이름.
* `topology.kubernetes.io/zone` - 이 엔드포인트가 있는 영역의 이름.
* `topology.kubernetes.io/region` - 이 엔드포인트가 있는 지역의 이름.

이런 레이블 값은 슬라이스의 각 엔드포인트와 연관된 리소스에서
파생된다. 호스트 이름 레이블은 해당 파드의
NodeName 필드 값을 나타낸다. 영역 및 지역 레이블은 해당
노드에서 이름이 같은 값을 나타낸다.

### 관리

기본적으로 엔드포인트슬라이스는 엔드포인트슬라이스 컨트롤러에의해
생성되고 관리된다. 서비스 메시 구현과 같은 다른 엔드포인트슬라이스
유스 케이스는 다른 엔터티나 컨트롤러가 추가 엔드포인트슬라이스
집합을 관리할 수 있게 할 수 있다. 여러 엔티티가 서로 간섭하지 않고
엔드포인트슬라이스를 관리할 수 있도록 엔드포인트슬라이스를 관리하는
엔티티를 나타내는데 `endpointslice.kubernetes.io/managed-by` 레이블이 사용된다.
엔드포인트슬라이스 컨트롤러는 관리하는 모든 엔드포인트
슬라이스에 레이블의 값으로 `endpointslice-controller.k8s.io` 를 설정한다.
엔드포인트슬라이스를 관리하는 다른 엔티티도 이 레이블에
고유한 값을 설정해야 한다.

### 소유권

대부분의 유스 케이스에서 엔드포인트를 추적하는 서비스가 엔드포인트슬라이스를
소유한다. 이는 각 엔드포인트슬라이스의 참조와 서비스에 속하는 모든
엔드포인트슬라이스를 간단하게 조회할 수 있는 `kubernetes.io/service-name`
레이블로 표시된다.

## 엔드포인트슬라이스 컨트롤러

엔드포인트슬라이스 컨트롤러는 해당 엔드포인트슬라이스가 최신 상태인지
확인하기 위해 서비스와 파드를 감시한다. 컨트롤러가 셀렉터로 지정한 모든
서비스에 대해 엔드포인트슬라이스를 관리한다. 이는 서비스 셀렉터와
일치하는 파드의 IP를 나타내게 된다.

### 엔드포인트슬라이스의 크기

기본적으로 엔드포인트슬라이스는 각각 100개의 엔드포인트 크기로 제한된다.
최대 1000개까지 `--max-endpoints-per-slice` {{< glossary_tooltip
text="kube-controller-manager" term_id="kube-controller-manager" >}} 플래그를
사용해서 구성할 수 있다.

### 엔드포인트슬라이스의 배포

각 엔드포인트슬라이스에는 리소스 내에 모든 엔드포인트가 적용되는
포트 집합이 있다. 서비스에 알려진 포트를 사용하는 경우 파드는
동일하게 알려진 포트에 대해 다른 대상 포트 번호로 끝날 수 있으며 다른
엔드포인트슬라이스가 필요하다. 이는 하위 집합이 엔드포인트와 그룹화하는
방식의 논리와 유사하다.

컨트롤러는 엔드포인트슬라이스를 최대한 채우려고 노력하지만,
적극적으로 재조정하지는 않는다. 컨트롤러의 동작은 매우 직관적이다.

1. 기존 엔드포인트슬라이스에 대해 반복적으로, 더 이상 필요하지 않는 엔드포인트를
   제거하고 변경에 의해 일치하는 엔드포인트를 업데이트 한다.
2. 첫 번째 단계에서 수정된 엔드포인트슬라이스를 반복해서
   필요한 새 엔드포인트로 채운다.
3. 추가할 새 엔드포인트가 여전히 남아있으면, 이전에 변경되지 않은
   슬라이스에 엔드포인트를 맞추거나 새로운 것을 생성한다.
   
중요한 것은, 세 번째 단계는 엔드포인트슬라이스를 완벽하게 전부 배포하는 것보다
엔드포인트슬라이스 업데이트 제한을 우선시한다. 예를 들어, 추가할 새 엔드포인트가
10개이고 각각 5개의 공간을 사용할 수 있는 엔드포인트 공간이 있는 2개의
엔드포인트슬라이스가 있는 경우, 이 방법은 기존 엔드포인트슬라이스
2개를 채우는 대신에 새 엔드포인트슬라이스를 생성한다. 다른 말로, 단일
엔드포인트슬라이스를 생성하는 것이 여러 엔드포인트슬라이스를 업데이트하는 것 보다 더 선호된다.

각 노드에서 kube-proxy를 실행하고 엔드포인트슬라이스를 관찰하면,
엔드포인트슬라이스에 대한 모든 변경 사항이 클러스터의 모든 노드로 전송되기 
때문에 상대적으로 비용이 많이 소요된다. 이 방법은 여러 엔드포인트슬라이스가
가득 차지 않은 결과가 발생할지라도, 모든 노드에 전송해야 하는
변경 횟수를 의도적으로 제한하기 위한 것이다.

실제로는, 이러한 이상적이지 않은 분배는 드물 것이다. 엔드포인트슬라이스
컨트롤러에서 처리하는 대부분의 변경 내용은 기존 엔드포인트슬라이스에
적합할 정도로 적고, 그렇지 않은 경우 새 엔드포인트슬라이스가
필요할 수 있다. 디플로이먼트의 롤링 업데이트도 모든 파드와 해당
교체되는 엔드포인트에 대해서 엔드포인트슬라이스를
자연스럽게 재포장한다.

## 사용동기

엔드포인트 API는 쿠버네티스에서 네트워크 엔드포인트를 추적하는
간단하고 직접적인 방법을 제공한다. 불행하게도 쿠버네티스 클러스터와
서비스가 점점 더 커짐에 따라, 이 API의 한계가 더욱 눈에 띄게 되었다.
특히나, 많은 수의 네트워크 엔드포인트로 확장하는 것에
어려움이 있었다.

이후로 서비스에 대한 모든 네트워크 엔드포인트가 단일 엔드포인트
리소스에 저장되기 때문에 엔드포인트 리소스가 상당히 커질 수 있다. 이것은 쿠버네티스
구성요소 (특히 마스터 컨트롤 플레인)의 성능에 영향을 미쳤고
엔드포인트가 변경될 때 상당한 양의 네트워크 트래픽과 처리를 초래했다.
엔드포인트슬라이스는 이러한 문제를 완화하고 토폴로지 라우팅과
같은 추가 기능을 위한 확장 가능한 플랫폼을 제공한다.

{{% /capture %}}

{{% capture whatsnext %}}

* [엔드포인트슬라이스 활성화하기](/docs/tasks/administer-cluster/enabling-endpointslices)
* [애플리케이션을 서비스와 함께 연결하기](/ko/docs/concepts/services-networking/connect-applications-service/) 를 읽는다.

{{% /capture %}}