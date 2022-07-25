---
title: Istio의 Security
author: woodsection
date: 2022-07-24 22:33:00 +0800
categories: [Infra, Istio, Security]
tags: [ServiceMesh, Istio, Kubernetes]
published: true
---

# 보안(Security)

![Untitled](/assets/img/posts/2022/07/25/istio-security/Untitled.png)

**istio를 사용했을 때의 보안적인 특징**

- 애플리케이션과 인프라 코드의 수정없이 기본적으로 보안이 적용됨
- 기존 보안 시스템과 통합하여 보안 적용
- 신뢰할 수 없는 네트워크에도 보안을 적용할 수 있음


# 아키텍쳐

- CA(Certificate Authority) key와 인증 관리
- 설정 API 서버에서 프록시로 정책을 전달함
    - authentication policies(**[authentication policies](https://istio.io/latest/docs/concepts/security/#authentication-policies))**
    - authorization policies(**[authorization policies](https://istio.io/latest/docs/concepts/security/#authorization-policies))**
    - secure naming information(**[secure naming information](https://istio.io/latest/docs/concepts/security/#secure-naming))**
- 사이드카와 perimeter 프록시는 [PEPs](https://www.google.com/search?q=Policy+Enforcement+Points&oq=Policy+Enforcement+Points&aqs=chrome..69i57.2104j0j7&sourceid=chrome&ie=UTF-8)에 따라 클라이언트와 서버간에 안전한 커뮤니케이션
- Envoy proxy 확장세트는 원격측정과 감시를 관리함

![Untitled](/assets/img/posts/2022/07/25/istio-security/Untitled1.png)

- control plane에서 API서버와 PEPs에 따른 data plane을 관리
- PEPs는 Envoy를 통해 구현됨

# Istio identity

워크로드간 통신을 위해서는 각자가 신용할 수 있는지 확인하기 위해 서로의 credential을 교환해야함

**클라이언트**

서버의 ID가 ****[secure naming](https://istio.io/latest/docs/concepts/security/#secure-naming)을 통해 워크로드에 인증되어있는지 확인함

**서버**

서버에서는 클라이언트가 특정 정보에 액세스할 수 있는지에 대해 authorization policies를 통해 확인함

누가 언제 액세스했는지, 워크로드를 사용한 것에 대한 비용 부과, 비용을 지불하지 못한 클라이언트의 거부 등을 수행함

istio identity는 첫번째로 `service identity` 를 request origin의 identity를 결정함

매우 유연하고 세분화된 service identities를 개인, 그룹 유저에게 제공해줌

istio는 `service identity` 가 없는 플랫폼에선 여러 다른 인스턴스를 통해 identity를 결정하는데, 그 목록은 다음과 같음

- K8s → kubernetes service account
- GCE → GCP service account
- On-promises(non-k8s) → user account, custom service account, service name, istio service account, or GCP service account

# Identity and certificate management
(신원, 인증관리)

모든 워크로드에서 X.509 certificate를 사용

![Untitled](/assets/img/posts/2022/07/25/istio-security/Untitled2.png)

1. `istiod` 에서 CSR을 받는 gRPC 서비스를 제공
2. [istio-agent → istiod] private key, CSR 전송
3. [istiod] istio의 Certificate Authority를 통해 인증서 발급
4. 워크로드가 시작되면 Envoy는 SDS API를 통해 `istio-agent`에 인증서와 키를 요청함
5. `istio-agent`는 `istiod` 로부터 받은 인증서와 키를 SDS api를 통해 envoy로 전송
6. `istio-agent` 에서 워크로드 인증서의 만료여부를 모니터링함
7. 위의 과정들이 반복되며 인증서와 키가 순환됨

<!-- [Authentication (인증)](https://www.notion.so/Authentication-ca8d26fc4a804a39a9e70b47e82d0b15) -->


<!-- [[Authorization](https://istio.io/latest/docs/concepts/security/#authorization) (허가)](https://www.notion.so/Authorization-034325f7d4204a138b9ed662587bc3fa) -->