---
title: Maven Local
author: woodsection
date: 2022-09-18 17:16:00 +0900
categories: [Web, Java, Build]
tags: [Web]
published: true
---

# 개요

---

spring 으로 만들어진 레거시 프로젝트를 살펴보다가, 패키지를 가져올때 maven repo가 아닌 로컬에서 빌드해둔 jar 파일을 import 해오는 구문을 발견했습니다.

살펴보던 프로젝트는 msa에서 하나의 서비스 부분이였는데, 구글 protobuf로 만들어둔 dto 클래스 모음 프로젝트를 로컬에 jar로 빌드하고 해당 jar을 각 프로젝트마다 로컬에서 import 해와 사용하는 형태로 구성되어있었습니다.

패키지를 import 해올때 repo 같은 설정은 크게 신경쓰지 않았던 부분인데, 이번 기회에 간단하게나마 정리해두려고합니다.

---

# gradle 빌드

gradle에서 로컬의 maven 저장소로 빌드하는 방법은 간단합니다.

`gradle publishToMavenLocal`

---

# repo 설정확인

프로젝트 빌드는 gradle로 진행한다고 가정하겠습니다.

`build.gradle`

```gradle
...
repositories {
  mavenLocal()
  mavenCentral()
  ...
}
...
```

`build.gradle` 파일을 살펴보면 repositories를 설정하는 필드가 있는데, 해당 부분에 mavenCentral, mavenLocal이 있습니다.

mavenLocal 이 설정되어있으면 로컬에 있는 패키지를 import 할 수 있는 상태가 됩니다.

---

# dependencies 임포트

mac기준 default로 `~/.m2/repository ` path에 package들이 저장되며, 각 디렉토리마다 . 으로 구분하여 gradle의 `dependencies` 필드에 추가해줍니다.

```gradle
dependencies {
    ...
    implementation "com.woodsection.test.application:0.0.1"
    ...
}
```
