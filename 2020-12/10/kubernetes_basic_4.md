# Kubernetes 정리

* [출처] 조대협님 블로그 <https://bcho.tistory.com/1255?category=731548>
* TAG: Kubernetes

* 실습 <https://bcho.tistory.com/1261?category=731548> RC 사용해서 서비스 배포하기

## 서비스

Pod의 경우 지정되는 IP가 동적으로 재시작 시마다 변하기 때문에 고정된 엔드포인트로 호출하기 어렵다. \
여러 Pod에 같은 어플리케이션을 운용할때의 로드밸런싱을 위해서 서비스를 사용한다. \
서비스는 지정된 IP로 생성할 수 있고, 여러개의 Pod를 묶을 수 있고 (label), DNS domain을 가질 수 있다. \

서비스는 동시에 여러개의 포트를 지원할 수 있다. https와 http 포트가 대표적인 예시이다. \
또 서비스는 Pod 간에 랜덤으로 부하를 분산시키는데, 분산의 정책도 spec의 sessionAffinity로 정할 수 있다. \

아래에는 대표적인 4가지의 서비스가 있다.

### ClusterIP

기본 설정으로 서비스에 내부 IP를 할당한다. 쿠버네티스 클러스터 내에서는 접근이 가능하지만, 외부에서는 불가능하다.

### Load Balancer

클라우드 벤더사에서 제공하는 방식으로 외부 IP를 가진 로드밸런서를 할당한다.

### NodePort

클러스터 IP 뿐만 아니라 모든 노드의 IP와 포트를 통해서도 접근이 가능해진다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-node-svc
spec:
  selector:
    app: hello-node
  type: NodePort
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: 8080
      nodePort: 30036
```

위와 같이 정의된 서비스는 클러스터 IP의 80번으로도 접근이 가능하지만 \
다른 노드의 30036 포트로도 서비스에 접근할 수 있게 된다.

![k8s_nodeport](./k8s_nodeport)

위와 같이 Service에 직접 접근해서 node에 접근도 가능하지만, Node에 열린 30036 포트로도 접근 가능하다.

### ExternalName

외부 서비스를 쿠버네티스 내부에서 호출하고자 할 때 사용할 수 있다. \
Pod들은 ClusterIP를 가지기 때문에 클러스터 IP 대역에서 벗어난 외부의 서비스 호출시에는 NAT 설정이 필요하다. \
그럴때 서비스를 ExterName 타입으로 설정하고 domain을 설정해주면 이 서비스는 \
들어오는 모든 요청을 도메인으로 포워딩 해준다. (DNS 같은 느낌)

### Headless Service

서비스는 Cluster IP 혹은 External IP를 가지고 서비스 내부에서 제공되는 기능에 대한 엔드포인트를 \
쿠버네티스 서비스를 통해서 통제한다. 이때 Headless service는 DNS 이름만 가지고 cluster ip를 가지지 않는다. \
이 서비스의 IP를 loopup 하면 서비스에 연결된 Pod 들의 IP 주소를 리턴한다.

### Service Discovery

그러면 생성된 서비스의 IP는 알 수 있는 방법이 있을까, 서비스가 생성된 이후에 kubectl get svc 를 사용하면 \
생성된 서비스와 IP를 확인할 수 있지만, 이는 변경되는 IP이다.

DNS를 이용하면 가장 쉽게 해결할 수 있는데, 서비스는 서비스명.네임스페이스명.svc.cluster.local 이라는 \
Domain으로 내부 DNS에 등록되게 된다. 내부에서는 DNS 이름으로 접근이 가능하고 Cluster IP를 사용한다.

또 다른 방법으로는 외부 IP를 명시적으로 지정하는 방식이 있는데, 이 IP는 반드시 외부에서 명시적인 지정과 관리가 필요하다.

퍼블릭 클라우드와 같은 경우에는 클라우드 내부의 로드밸런서를 부착하는 방식을 더 잘 사용할 수 있는데, \
정적인 로드밸런서 IP를 생성해서 타입이 LoadBalancer인 서비스에 부착해주면 External IP를 해당 IP로 하는 \
서비스가 생성된 것을 확인할 수 있을 것이다.
