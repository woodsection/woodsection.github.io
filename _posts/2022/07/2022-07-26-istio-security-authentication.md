---
title: Istio 에서의 authentication
author: woodsection
date: 2022-07-26 22:32:00 +0900
categories: [Infra, Istio, Security]
tags: [ServiceMesh, Istio, Kubernetes]
published: true
---

# Authentication (인증)

# 기본 컨셉

Istio에서는 두 가지 authentication(인증) 방식을 제공합니다.

1. Peer authenticaion
- 클라이언트의 연결을 확인하기 위해 서비스간 인증에 사용되는 인증방식입니다.

Istio에서는 서비스 코드의 변경없이 활성화가능한 상호간 TLS를 제공합니다.

- 서비스간 통신 보호
- 클러스터 - 클라우드 사이에 운용가능한 ID를 각 서비스에 제공
- 키, 인증서를 생성 및 배포, 순환을 자동화하는 시스템 제공
1. Request authentication

end 유저를 위한 사용자 인증에 사용됩니다.

JWT를 이용한 유효성 검사, 권한확인 혹은 OpenID Connect 공급자를 사용해 개발자 환경을 제공합니다.

예시로

- ORY Hydra, Keycloak, Auth0, Firebase Auth, Google Auth 등 ..

Istio는 인증정책은 `Istio config store`에 k8s CRD를 이용해 저장하며, istiod는 키를 통해 각각의 프록시를 최신 정보로 업데이트합니다.

# **Mutual TLS authentication (상호간 TLS 인증)**

Istio에서는 Envoy 프록시로 구현하는 클라이언트와 서버측의 PEP를 이용해 서비스간 통신을 터널링합니다.

Mutual TLS 인증으로 다른 워크로드에 request 할때의 동작방식은 다음과 같습니다.

1. 클라이언트의 로컬 사이드카 Envoy로 아웃바운드를 다시 라우팅합니다.
2. 클라이언트 Envoy와 서버 Envoy가 TLS handshake를 수행합니다. 핸드쉐이크가 일어나는동안, 클라이언트 Envoy 에서는 secure naming 검사를 통해 서버 인증서의 service account가 대상의 서비스를 실행할 권한이 있는지 확인합니다.
3. 각 Envoy는 TLS 연결을 설정하고, 클라이언트의 Envoy는 서버의 Envoy로 트래픽을 전달합니다.
4. 서버 Envoy가 요청을 승인하고, 로컬 TCP 연결로 트래픽을 백엔드 서비스로 전달합니다.

## Permissive mode (허용모드)

서비스가 일반 트래픽과 Mutual TLS 트래픽을 모두 수락할 수 있는 모드입니다.

## Secure naming

server identity는 인증서로 인코딩되지만, 서비스의 이름은 서비스나 DNS를 통해 검색됩니다.

secure naming은 server indentity를 서비스 이름에 매핑하며, 이는 server identity가 서비스를 실행할 권한이 있음을 의미합니다.

마스터 노드에서 apiserver를 감시하고 sercure naming을 생성하여 PEP에 배포합니다.

서비스를 호출할때, 인증서에서 ID를 추출하고, secure naming과 비교하여 봄으로서 서비스 실행가능 여부를 검증하므로, secure naming을 통해 DNS spoofing, BGP/route hijacking, ARP spoofing 등의 공격으로부터 보호할 수 있는 수단이됩니다. 

# **Authentication architecture (인증 아키텍처)**

peer와 authorization policy를 사용해서 워크로드에 대한 인증 설정을 지정할 수 있습니다.

yaml 파일을 통해 policy를 지정하고 정책이 배포되면 istio 구성 스토리지에 저장되며, istio controller에서 구성 스토리지를 감시합니다.

policy의 변경시 istio는 변경된 policy를 엔드포인트에 비동기식으로 보내고, 프록시가 수신하면 해당 policy가 포드에 즉시 적용됩니다.

이러한 요청은 다음과 같은 아키텍처를 통해 이뤄집니다.

![https://istio.io/latest/docs/concepts/security/authn.svg](https://istio.io/latest/docs/concepts/security/authn.svg)

# **Authentication policies (인증정책)**

인증정책은 서비스가 수신하는 request에 적용되며, mutural TLS에서 클라이언트 측의 인증 규칙을 지정하기 위해선 `DestinationRule` 을 설정해야합니다.

앞서 이야기했듯이 yaml 파일 형식으로 k8s의 CRD를 통해 지정할 수 있습니다.

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: "example-peer-policy"
  namespace: "foo"
spec:
  selector:
    matchLabels:
      app: reviews
  mtls:
    mode: STRICT
```

## **Policy storage (정책 저장)**

istio root 네임스페이스에 mesh-scope policy를 저장합니다.

이 정책은 전역적으로 설정되며, 네임스페이스를 지정하여 policy를 저장하면, 해당 네임스페이스 내의 워크로드내에서만 적용이 됩니다.

 `selector` 필드를 이용하여 네임스페이스를 지정할 수 있습니다.

## Selector field

다음과 같이 selector 필드를 이용해 정책이 적용되는 워크로드 레이블을 지정할 수 있습니다.

```yaml
selector:
  matchLabels:
    app: product-page
```

- selector 필드에 값이 없으면, 루트 네임스페이스에 대해 지정된 정책이 적용됩니다.

다음과 같은 순서로 각 워크로드에 정책을 적용합니다.

1. 워크로드별
2. 네임스페이스 전체
3. 메쉬 전체

정책이 여러가지가 적용되면, 모든 정책을 결합해서 적용되므로 여러 네임스페이스의 정책을 가질수도 있지만 권장하지는 않습니다.

## **Peer authentication (피어 인증)**

피어인증에서는 mutural TSL 모드를 지정하며, 지원되는 모드는 다음과 같습니다.

- PERMISSIVE(허용)
    - Mutural TSL, 일반 트래픽 모두 허용하며, 사이드카가 없는 워크로드에서 사용하기에 적합합니다.
    - 사이드카가 적용이 된 후에는 STRICT 모드로 변경해야합니다.
- STRICT(엄격)
    - Mutural TSL만을 허용합니다.
- DISABLE(비활성화)
    - Mutural TSL이 비활성화되므로 자체적인 보안 솔루션이 없다면 비활성화모드를 사용해선 안됩니다.

모드가 적용되지않으면 상위의 모드가 상속되며, 상속될 모드도 없다면 PERMISSIVE 모드로 적용됩니다.

다음의 설정에선 `foo` 네임스페이스 상의 워크로드에게 STRICT 모드가 적용됩니다.

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: "example-policy"
  namespace: "foo"
spec:
  mtls:
    mode: STRICT
```

특정 포트만을 지정하여 모드를 지정하는 것도 가능합니다.

다음의 설정에선 `foo` 네임스페이스에 속하는 `80` 포트의 워크로드에서 Mutural TSL을 비활성화했습니다.

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: "example-workload-policy"
  namespace: "foo"
spec:
  selector:
     matchLabels:
       app: example-app
  portLevelMtls:
    80:
      mode: DISABLE
```

다음의 구성에선 Mutural TSL이 비활성화된 80 포트로 바인딩되므로 요청이 성공적으로 작동합니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: example-service
  namespace: foo
spec:
  ports:
  - name: http
    port: 8000
    protocol: TCP
    targetPort: 80
  selector:
    app: example-app
```

## **Request authentication**

JWT의 유효성을 검사하기 위해 필요한 값을 지정하며, 그 값은 다음과 같습니다.

- request에서의 token의 위치
- issuer와 request
- 공개 JSON 웹 키 set(JWKS)

istio에서 규칙에 따라 토큰을 확인하고 유효성 여부를 판단합니다.

request에 token이 없으면, 기본적으로 허용되며, 토큰이 없는 요청을 거부하려면 path, action 등을 지정하는 규칙을 지정해야합니다.

둘 이상의 정책이 워크로드에 적용되어있다면, 각 정책에 따라 둘 이상의 JWT를 지정할 수 있으며, 단일 정책처럼 모든 규칙을 결합합니다.

하지만, 둘 이상의 JWT가 있는 요청은 지원하지 않습니다.

즉, 정책이 2개가 적용된 워크로드에 대하여 각 정책에 적용되는 JWT 1개 씩 총 2개는 가능하지만,

정책이 2개가 적용된 워크로드에 대하여 각 정책에 적용되는 JWT 2개씩 총 4개는 불가능합니다.

## Principals

peer authentication policy, mutural TLS 등을 사용할 때, identity를 가져오기 위해 
`source.principal` , `request.auth.principal` 등과 같은 보안 주체를 사용합니다. 

# **Updating authentication policies (인증정책 업데이트)**

authentication policy는 언제든 변경될 수 있으며, 거의 실시간으로 적용됩니다.

하지만, 모든 워크로드가 새로운 정책을 수신했는지 보장할 순 없습니다.

때문에 istio에서는 다음과 같이 권장합니다.

- 모드를 DISABLE → STRICT 또는 그 반대의 경우로 변경하고자한다면, 중간에 PERMISSIVE 모드로 지정한 뒤 바꿔야합니다. Istio telemetry를 통해 모드가 성공적으로 바뀌었는지 확인할 수 있습니다.
- request authentication을 다른 jwt로 마이그레이션하고자할때, 이전 규칙을 냅둔 상태로 새 jwt 정책을 추가한 뒤, 모든 트래픽이 새로운 jwt로 적용되고 난 이후에 이전 규칙을 제거해야합니다.
단, 새로운 jwt는 이전의 jwt와 다른 위치를 사용해야합니다.