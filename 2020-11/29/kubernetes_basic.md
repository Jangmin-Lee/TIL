# Kubernetes 정리

* [출처] 조대협님 블로그 <https://bcho.tistory.com/1255?category=731548>
* TAG: Kubernetes

## 1. 소개

* Kubernetes가 왜 필요한가

VM이나 하드웨어의 수가 많아지고 컨테이너의 수가 많아지면 컨테이너 배포 등에 대한 관리가 필요하다. \
어플리케이션의 특성에 따라서 하드웨어 스팩과 네트워크 환경 등에 차이가 존재하기 때문이다. \
이런 작업을 Scheduling이라고 하는데, 이것 말고도 Health Check, Monitoring, Deletion 등을 관장하는게 필요. \
이를 관리하기 위해 컨테이너 운영환경이라는 것이 대두되었고, 그 중에 가장 성공적인 매니징 툴이 k8s인것.

## 2. 개념 정리

### Master & Node

가상머신 전체를 관리하는 마스터(컨트롤러)와 관리되는 컨테이너가 배포되는 노드(가상머신, 물리머신)이 존재한다. \

![구조 설명](./k8s_struct.png)

이때 구조가 간단하게 보이는데, 세부적인 구조는 4개의 기초적인 오브잭트와 컨트롤러로 설명 가능하다.

### 오브잭트

* 구성의 가장 밑바닥이 되는 단위, 컨트롤러의 관리를 받음.
* 오브잭트는 Json이나 Yaml 파일을 통해서 Config가 관리됨.

#### Pod

쿠버네티스에서 가장 기본적인 배포 단위로, 컨테이너를 포함한다. \
Pod에는

1. ApiVersion, 스크립트 실행에 필요한 쿠버네티스 버전
2. Kind, 오브잭트의 종류, Pod를 넣는다.
3. Metadata, 각종 메타 데이터들, 라벨이나 리소스의 이름 등
4. Spec, 리소스에 대한 상세한 스펙, 컨테이너의 스펙 및 config 정리

Pod 내부의 컨테이너는 IP와 Port를 공유한다. \
따라서 두개의 컨테이너가 하나의 Pod를 통해서 배포되면 localhost를 통해 통신이 가능하다. \
또 디스크 저장공간을 공유한다. 이 방법으로 로깅 서버를 다른 컨테이너에서 띄워서 어플리케이션 컨테이너의 파일을 읽어올 수 있다.

#### Volume

Pod가 가동될때 기본값으로 컨테이너마다 로컬 디스크를 생성한다. \
이 디스크는 컨테이너에 종속된 휘발성으로, 파일을 저장해서 남길 필요가 있을때는 \
볼륨의 형태로 저장공간을 만들어야한다. \

쿠버네티스의 볼륨은 Pod 내의 컨테이너에서 모두 접근이 가능함으로 \
웹 서버를 배포하는 Pod에서 container간 필요에 따라 데이터간 분리를 이뤄낼 수 있다.

[!k8s_volume](./k8s_volume.png)

#### Service

Pod와 Volume을 사용해서 컨테이너를 정의한 후에 여러개의 Pod를 서비스 하면서 \
로드밸런서를 사용해서 하나의 IP와 Port로 묶어서 서비스를 배포하게 된다. \
Pod는 오토스케일링, 장애등의 사유로 동적으로 자동생성되면서 IP가 바뀌게 된다. \
이때 이 Pod를 로드밸런서가 유연하게 선택할 수 있게 하기 위해서 Label과 Label Selector라는 개념을 사용한다. \

서비스를 정의할 때, 어떤 Pod들를 하나의 서비스로 묶을 것인지를 정해야 한다. 이를 라벨 셀렉터라고 한다. \
그리고 Pod를 정의할 때 Metadata에 라벨을 정의하면, 서비스는 특정 라벨을 가진 Pod를 선택해서 서비스에 binding한다.

```yaml
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  selector:
    app: myapp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9376
```

리소스 종류는 Service, 서비스의 이름은 my-service, spec에 서비스 상세를 정의한다. \
selector에서 라벨이 myapp인 Pod를 선택해서 서비스의 80번 포트를 컨테이너의 9376번 포트로 연결해준다. \

라벨은 쿠버네티스의 리소스를 선택할 때 사용되는데, 각 리소스는 라벨을 가질 수 있다. \
라벨의 검색 조건에 따라서 특정 라벨을 가지고 있는 리소스만을 선택하여 적용할 수 있다. \

```yaml
"metadata": {
  "labels": {
    "key1" : "value1",
    "key2" : "value2"
  }
}
```

이런 식으로 하나의 리소스에 여러개의 라벨을 적용할 수 있는데, 이 라벨을 선택할 때는 \
오브젝트의 spec에서 selector를 정의하고 라벨의 조건을 기술하면 된다. \
이때 label-selector는 2가지가 존재하는데,

1. Equality based selector, environment (!)= dev
2. Set based selector, environment (not)in (production, test)

t/f를 기반으로 한 셀렉터와 포함관계를 기반으로 한 셀렉터 중에 편한걸 선택해서 사용할 것.

#### Name space

Name Space는 쿠버네티스 클러스터 내부의 논리적인 분리단위라고 보면 된다. \
Pod나 Service는 Name Space별로 생성과 관리가 이뤄지고, 사용자의 권한 역시 각각으로 부여할 수 있다. \
하나의 클러스터 내부에 개발/운영/테스트의 환경이 있을때 클러스터를 각각의 Name Space로 나눠서 운영할 수 있다. \
사용자는 각각의 Name Space마다 다른 권한을 가지며, 각 Name Space는 개별적인 리소스를 가진다. \

* 주의점은 네임 스페이스는 논리적인 분리 단위고 물리적인 환경의 분리는 발생하지 않았다.
* 따라서 다른 Name Space Pod 사이에 통신을 할 수 있다.

[!k8s_namespace](./k8s_namespace.png)

### Controller

기본 오브젝트들로 어플리케이션을 setting & deploy하는 것이 가능한데, \
이를 편하게 관리하기 위해서 컨트롤러라는 개념을 만들어서 사용한다. \
컨트롤러는 기본 오브젝트들의 생성 및 관리를 담당한다. \

#### Replication Controller

RC는 Pod를 관리해주는 역할을 하는데, 지정된 숫자의 Pod를 가공시키고 관리한다. \
이때 Replica Number, Pod selector, Pod Template의 3가지로 구성된다. \

1. Selector, 라벨을 기반으로 하여 Pod를 가져오는 것에 사용한다.
2. Replica Number, RC에 의해 관리되는 복제 Pod의 수이다. 이 숫자만큼 유지한다.
3. Pod Template, Pod를 추가로 가동할 때 어떻게 만들지가 저장된 config template

이미 배포된 Pod가 있는 상태에서 RC를 생성할 때, 그 Pod의 라벨이 RC의 selector와 일치하면 \
새로 생성된 RC의 컨트롤을 받는다. 만약 Pod들의 개수가 Replica Number를 초과할 경우에는 Pod가 \
수에 맞게 삭제되고, 미만이라면 template에 정의된 것에 따라 Pod를 생성한다. \
이 상황에서 기존에 생성된 Pod가 template와 다르더라도 신경쓰지 않는다.

#### Replica Set

Replication Controller의 새로운 버전인데, set based selector를 사용하는 버전이다. \

#### Deployment

Deployment는 RC와 RS의 상위 추상화 개념이다. 실제 운영에서는 Deployment가 자주 사용된다. \
Deployment를 사용하지 않는 경우에는 Blue/Green deploy 혹은 Rolling deploy를 수동으로 진행하게 된다. \
B/G Deploy는 새로 RC를 만들어서 템플릿으로 Pod 생성 후에 Service를 옮기는 방식으로 진행한다. \
Rolling Deploy는 하나씩 Pod를 생성해서 기존의 RC에서 하나를 뺴고, 새로운 RC에서 하나씩 올리면서 \
라벨을 같은 이름으로 만들면 Service가 알아서 포함시킨다. 이를 반복해서 바꾼다.

위와 같은 배포와 롤백의 과정을 자동화하고 추상화한 개념을 Deployment라고 한다.
Deployment는 Pod 배포를 위해 RC를 생성하고 관리하며, 롤백을 위한 RC 또한 관리한다.
