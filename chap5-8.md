# 쿠버네티스 (Kubernetes)

# Chap 5. 쿠버네티스 입문

## 발전 계보

```
도커 (dockerd + CLI)
  → 컴포즈 (한 호스트, 멀티 컨테이너 / compose.yaml)
    → 스웜 (도커 자체 오케스트레이션, 현재는 사용 X)
      → 쿠버네티스
```

---

## 대시보드 설치

**대시보드란?** 쿠버네티스 클러스터 안에 배포된 웹 기반 관리 도구. `kubectl` 명령어에 익숙하지 않을 때 리소스들을 눈으로 볼 수 있는 GUI.

> 📸 **1. 대시보드 캡처 확인**

### 설치 순서

1. 대시보드를 클러스터 안에 배포
```bash
   kubectl apply -f https://.../recommended.yaml
```
2. `dashboard-user.yaml` → 로그인용 계정(ServiceAccount) + 권한(ClusterRoleBinding) 생성
3. `kubectl create token` → 로그인용 토큰 발급
4. `kubectl proxy` → 외부(브라우저)에서 접속 가능하게 통로 열기
5. 브라우저 접속 → 토큰 입력 → 로그인

**핵심:** 대시보드는 클러스터 내부 전용이라 바로 접근이 불가능하다.

> 📸 **2. 클러스터 ~ 컨테이너 계층 구조**

---

## 클러스터와 노드

클러스터는 크게 **컨트롤 플레인**과 **워커 노드**로 구분된다.

### 컨트롤 플레인

`kube-apiserver` / `etcd` / `kube-scheduler` / `kube-controller-manager`

> ⚠️ `kube-scheduler`는 Spring의 `@Scheduled` 같은 **시간 기반** 작업이 아니다.
> 파드를 **어느 노드에 배치할지** 결정하는 배치 알고리즘 개념.

### 워커 노드

컨트롤 플레인의 결정에 따라 실제로 파드가 실행되는 서버.

### 실무 환경

실무에서는 이 둘이 분리되어 있어, 컨트롤 플레인은 클라우드 제공자가 관리형으로 운영하고 실무자는 **워커 노드 그룹만** 신경 쓰면 된다.

### 로컬 환경 확인

```bash
kubectl get nodes
```

```
NAME                           STATUS   ROLES           AGE
docker-desktop-control-plane   Ready    control-plane   2d
```

- `docker-desktop` 노드는 직접 만든 것이 아니라, Docker Desktop에서 Kubernetes를 활성화하는 순간 자동으로 구성된 기본 노드다. (Docker Desktop 앱과는 다른 것)
- 내부적으로 **kind**라는 도구로 만들어낸 가상 노드이며, 실제로는 **컨테이너 하나**로 구현되어 있다.
- 즉, 쿠버네티스 환경을 켜는 순간 **최소 하나의 노드는 자동으로 생성**된다.

---

## 레플리카셋 (ReplicaSet)

실무에서는 "API 파드 3개는 항상 떠 있어야 한다" 같은 조건이 필요하다. 하나가 죽으면 자동으로 다시 채워주는 역할을 하는 것이 레플리카셋.

```yaml
spec:
  replicas: 3              # 몇 개 유지할지
  selector:
    matchLabels:
      app: echo            # 어떤 파드가 내 담당인지
  template:                # 여기 아래가 파드 매니페스트 그대로
    metadata:
      labels:
        app: echo
```

### 동작 방식

레플리카셋은 **자기가 만든 파드 이름을 기억하지 않는다.**

매 순간 `app=echo` 라벨이 붙은 파드를 세고, 3개보다 많으면 삭제 / 적으면 생성만 담당한다.

> ⚠️ 그렇기 때문에 `selector`의 라벨과 `template`의 라벨이 **반드시 동일해야 한다.**

> 📸 **3. 레플리카셋 캡처 확인**
>
> 3개를 유지하도록 설정했기 때문에, 임의로 한 파드를 삭제해도 바로 다시 생성해 3개를 유지하는 모습.

---

## 디플로이먼트 (Deployment)

### 왜 필요한가

레플리카셋으로 개수 유지는 되지만 **버전을 바꿀 때** 문제가 생긴다.

이미지를 `v0.1.0` → `v0.2.0`으로 바꾸려고 YAML을 수정해 apply해도 **기존 파드는 안 바뀐다.** 레플리카셋은 "3개가 있는가?"만 보는데 이미 3개이기 때문에 아무것도 하지 않는다.

그 **세대 관리**를 해주는 것이 디플로이먼트.

### 구조

```
디플로이먼트
 ├─ 레플리카셋 (v0.1.0)  → 파드
 └─ 레플리카셋 (v0.2.0)  → 파드
```

디플로이먼트는 레플리카셋을 여러 개 거느린다. **버전 하나당 레플리카셋 하나.**

### 롤링 업데이트

배포하면 새 레플리카셋을 만들고, **새 것의 개수를 늘리면서 옛 것의 개수를 줄인다.** 서서히 갈아타기 때문에 서비스가 끊기지 않는다.

옛 레플리카셋은 지우지 않고 `replicas: 0`으로 남겨둔다. → 되돌릴 때 0을 다시 3으로 올리기만 하면 되므로 **롤백이 가능**하다.

> 📸 **4. 디플로이먼트 캡처 확인**
>
> 새 레플리카셋 `echo-5c787f9b77`이 올라가고, 기존 `echo-674c56b8fd`의 파드들이 순서대로 종료(`Error`)되는 것 확인.

> 📸 **5. 디플로이먼트 2 — 리비전 이력**
>
> - 지금까지의 이력이 모두 남아 있다. 실패한 레플리카셋도 지워지지 않고 `0`으로 남아 있는 것을 확인 → `rollout undo`로 롤백 가능.
> - 리비전 번호가 2, 4, 6... 처럼 띄엄띄엄한 이유: `rollout undo`로 롤백했기 때문. 특정 리비전으로 되돌리면 **그 리비전이 사라지고 내용이 최신 번호로 다시 붙는다.**

---

## 서비스 (Service)

### 왜 필요한가

파드가 죽고 새로 뜰 때마다 **IP가 바뀌는 동적 특성** 때문에 파드를 직접 호출하기 곤란하다. 변하지 않는 고정 주소를 제공하는 것이 서비스.

- 트래픽이 서비스로 들어오면 뒤의 파드 중 하나로 넘겨준다 (**로드밸런싱**)
- **서비스 디스커버리**: IP 대신 이름으로 부를 수 있게 해준다

서비스를 `echo`라는 이름으로 만들면 클러스터 안 어디서든 호출 가능하다.

```
http://echo
http://echo.default.svc.cluster.local
```

> 📸 **6. 서비스 캡처 확인**

### 실습 흐름

**1) 레플리카셋 두 개로 파드를 띄움**

`spring` 1개 / `summer` 2개. 컨테이너 내용은 동일하고 **라벨만 다르다.**

**2) 서비스 생성**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: echo
  labels:
    app: echo
spec:
  selector:
    app: echo
    release: summer
  ports:
  - name: http
    port: 80
```

셀렉터를 `app=echo` + `release=summer`로 지정 → **summer 라벨을 가진 파드만 대상으로 삼겠다는 뜻.**

**3) `describe`로 확인**

- `Endpoints`에 IP가 **두 개만** 잡혀 있다 → summer 파드 2개만 선택된 것
- `CLUSTER-IP`는 파드 IP 대역과 다르다. 이건 **실재하는 IP가 아니라 가상 IP**라서 어느 노드나 파드도 이 주소를 갖고 있지 않다. `kube-proxy`가 이 주소로 가는 트래픽을 가로채 실제 파드로 전달한다.
- 단, **존재하는 동안은 절대 바뀌지 않는다.** 파드 IP는 배포할 때마다 바뀌지만 이 IP는 고정.

**4) 파드 안에서 호출**

확인을 위해 **spring 파드** 안에 들어가서 호출했다. spring은 이 서비스의 **대상이 아닌** 파드 — 일부러 대상이 아닌 곳에서 호출한 것.

```bash
curl http://echo
```

IP가 아닌 **서비스 이름**으로 호출. 클러스터 안의 DNS(CoreDNS)가 `echo`를 서비스 IP로 바꿔주고, 거기서 실제 파드로 넘어가는 구조다.

→ 부르는 쪽은 상대 IP를 하나도 몰라도 된다. 이것이 **서비스 디스커버리**.

**5) 로드밸런싱 확인**

오른쪽 터미널은 summer 파드 중 하나의 로그를 실시간으로 본 것. 왼쪽에서 `curl`을 여러 번 호출했으나 **호출한 만큼 찍히지 않는다.** summer 파드 2개에 나눠서 가기 때문 → 서비스의 로드밸런서 역할 확인.

**6) 정리**

> 파드는 죽고 새로 뜰 때마다 IP가 바뀌므로 직접 부를 수 없다.
> 서비스는 그 앞에 **고정된 이름과 IP**를 세워두고, 뒤에 붙을 파드는 **라벨로** 찾는다.
> 파드가 몇 개든 언제 바뀌든, 부르는 쪽은 **이름만 알면 된다.**

---

### Q. 실무에서 셀렉터로 대상을 좁히는 걸 왜 쓰는가?

신버전 파드를 다른 라벨로 띄우고 서비스 셀렉터만 바꿔서 트래픽을 옮기는 **카나리·블루그린 배포**에 쓴다.

**카나리 배포**

기존 v1이 파드 10개로 돌고 있을 때 전체 교체가 부담스러운 경우:
- v1 레플리카셋 파드 9개 / v2 레플리카셋 파드 1개
- 서비스 셀렉터는 `app: todo` 하나만 지정 → 10개 전부가 대상이 되지만 **10%만 v2를 받는다**
- 에러율을 보며 서서히 v2 비율을 늘린다

**블루그린 배포**

- v1, v2를 각각 10개씩 띄워 준비시켜 놓고, 서비스 셀렉터만 v1 → v2로 변경
- 트래픽이 **순간적으로 통째로** 넘어간다
- 문제가 생기면 셀렉터를 되돌려 즉시 롤백

> 다만 일반적인 배포는 **디플로이먼트의 롤링 업데이트로 충분하다.**

---

### Q. 서비스가 레플리카셋을 가리키는 것인가?

**아니다.** 서비스는 레플리카셋을 모른다. 그저 **라벨 조건에 맞는 파드를 직접 찾을 뿐**이다.

`Endpoints`에도 파드 IP만 나오고 레플리카셋 이름은 없는 것이 그 증거.

> **레플리카셋 ↔ 서비스는 서로 아무 연관이 없다.**

| 리소스 | 역할 |
|---|---|
| **디플로이먼트** | 어떤 버전을, 어떻게 교체할지 (리비전 관리) |
| **레플리카셋** | 그 버전의 파드를 몇 개 유지할지 |
| **서비스** | 어떤 파드로 트래픽을 보낼지 |

각각의 역할은 이것뿐이다. 다만 **파드의 라벨을 바라본다**는 공통점만 갖고 있다.

그래서 파드의 라벨링은 디플로이먼트 · 레플리카셋 · 서비스 **모두에 영향을 준다.**

> 💡 **라벨은 쿠버네티스에서 리소스를 잇는 유일한 접착제다.**

---

## 서비스 유형

| 구분 | 유형 | 설명 |
|---|---|---|
| **클러스터 안에서 부를 때** | `ClusterIP` | 기본값. 가상 IP + 로드밸런싱 (실습에서 사용) |
| | `Headless` | 가상 IP 없이 파드 IP를 그대로 반환. DB처럼 특정 파드를 지목해야 해 **로드밸런싱이 방해가 될 때** 사용 |
| **클러스터 밖에서 들어올 때** | `NodePort` | 모든 노드의 특정 포트를 개방 (30000~32767). 포트 대역 제한과 노드 IP 의존 때문에 단독 사용은 부적합 |
| | `LoadBalancer` | 클라우드 로드밸런서를 자동 생성. 내부적으로 NodePort를 사용 |
| **클러스터 밖으로 나갈 때** | `ExternalName` | 셀렉터도 파드도 없음. 외부 주소에 **DNS 별명(CNAME)**을 붙여, 앱이 안팎을 구분하지 않고 같은 이름으로 호출하게 함 |

## 인그레스 (Ingress)

### 왜 필요한가

클러스터 외부에 서비스를 공개하려면 NodePort로 열어야 하는데, 이 방법은 **L4 계층까지로 한정**된다.

| 계층 | 볼 수 있는 정보 | 비유 |
|---|---|---|
| **L4** (전송 계층) | IP 주소, 포트만 | 봉투의 주소만 읽고 배달 |
| **L7** (응용 계층) | HTTP 경로, 도메인, 헤더 | 봉투를 뜯어 내용까지 확인 |

NodePort는 포트 번호로만 구분하므로 아래처럼 **경로 기반으로 나누는 것이 불가능**하다.

```
todo.com/       → web 서비스
todo.com/api    → api 서비스
```

같은 도메인, 같은 포트인데 경로로 갈라야 하기 때문. 이 문제를 해결하는 리소스가 **인그레스**다.

---

### 하는 일

- **경로 기반 라우팅** — `/api`는 api 서비스로, `/`는 web 서비스로
- **virtual host** — 도메인별로 다른 서비스로 (nginx의 `server_name`과 같은 개념)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: todo
spec:
  rules:
  - host: todo.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web
            port:
              number: 80
```

---

### 전체 요청 흐름

```
사용자: todo.com 입력
  ↓ DNS 조회 ("todo.com이 어디야?" → "1.2.3.4")
브라우저가 1.2.3.4:443 으로 접속
  ↓
로드밸런서 (LoadBalancer 타입 서비스)      ← 외부로 열린 문
  ↓
인그레스 컨트롤러                          ← 규칙을 읽고 판단
  ↓ 경로/도메인 확인
서비스                                     ← 라벨로 파드 선택
  ↓
파드
```


각 층이 하나씩만 책임진다.

| 리소스 | 판단 기준 |
|---|---|
| **인그레스** | 경로 / 도메인 → 어느 **서비스**로? |
| **서비스** | 라벨 → 어느 **파드**로? |

인그레스는 서비스 **이름**으로 넘길 뿐, 파드를 직접 지목하지 않는다.

---

### 인그레스는 LoadBalancer를 대체하지 않는다

인그레스 컨트롤러도 결국 클러스터 안에서 도는 파드이므로, 외부 트래픽이 닿으려면 **앞에 LoadBalancer가 필요하다.** 인그레스는 그 뒤에 붙는다.

**인그레스가 해결하는 건 포트 문제가 아니라 개수 문제다.**

```
LoadBalancer만 사용:        인그레스 사용:
  web   → ELB 1개             ELB 1개 → 인그레스 → web
  api   → ELB 1개                                → api
  admin → ELB 1개                                → admin
  (서비스마다 하나씩, 비쌈)    (하나로 전부 커버)
```

---

### 주의: 인그레스 컨트롤러를 따로 설치해야 한다

| | 역할 |
|---|---|
| **인그레스** (매니페스트) | 규칙이 적힌 **종이** |
| **인그레스 컨트롤러** | 그 종이를 읽고 실행하는 **프로그램** (파드로 동작) |

디플로이먼트·서비스는 쿠버네티스가 기본으로 처리해주지만, 인그레스는 **규칙만 정의하고 실행은 별도 컨트롤러에 맡기는 구조**다. 컨트롤러를 설치하지 않으면 인그레스를 만들어도 아무 일도 일어나지 않는다. (`ADDRESS`가 계속 비어 있음)

대표적인 컨트롤러:
- **nginx-ingress** — 가장 많이 사용
- **AWS Load Balancer Controller** — EKS
- **Traefik**

---

---

## 인그레스 실습

### 1. 인그레스 컨트롤러 설치

인그레스는 매니페스트만 만들어서는 아무 일도 일어나지 않는다. 규칙을 실행할 **인그레스 컨트롤러**를 별도로 설치해야 하며, `deploy.yaml`을 apply하면 `ingress-nginx` 네임스페이스에 컨트롤러 파드가 생성된다.

- 컨트롤러의 서비스 타입은 **LoadBalancer**. 포트가 `80:31925`처럼 두 개 붙어 나오는데, 왼쪽은 서비스 포트 오른쪽은 내부 NodePort — LoadBalancer가 NodePort를 포함하는 구조임을 실물로 확인
- 설치 직후 `curl http://localhost` → `404 Not Found`. 컨트롤러는 살아있지만 아직 라우팅 규칙이 없어서 nginx 기본 응답이 나오는 것이 정상

---

### 2. 인그레스 리소스 생성 (host 기반 라우팅)

```yaml
spec:
  ingressClassName: nginx
  rules:
  - host: ch05.jpub.local
    http:
      paths:
      - path: /
        backend:
          service:
            name: echo
            port:
              number: 80
```

- `ingressClassName` — 컨트롤러가 여러 개 설치될 수 있어, 어느 컨트롤러가 이 규칙을 처리할지 지정
- `host`는 실제 도메인이 아니라서 hosts 파일 등록이 필요하지만, **`curl -H 'Host: ch05.jpub.local'`로 헤더만 지정하면 등록 없이도 테스트 가능**

> 💡 인그레스 컨트롤러는 요청이 어디서 왔는지가 아니라 **`Host` 헤더**를 보고 매칭한다. `Host` 헤더가 다른 요청(그냥 `localhost`)은 여전히 404 — virtual host가 실제로 동작하는 증거.

---

### 3. 조건부 라우팅 (annotation으로 nginx 설정 주입)

`spec`만으로는 host/path 라우팅까지만 가능하다. 그 이상(헤더 검사, 리다이렉트 등)은 **annotation**으로 nginx 설정을 직접 주입해서 구현한다.

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/server-snippet: |
      set $agentflag 0;
      if ($http_user_agent ~* "(Mobile)") { set $agentflag 1; }
      if ($agentflag = 1) { return 301 http://jpub.tistory.com/; }
```

**목적:** User-Agent에 `Mobile`이 포함된 요청(모바일)만 다른 사이트로 리다이렉트.

> `nginx.ingress.kubernetes.io/` 로 시작하는 annotation은 **nginx 컨트롤러 전용 확장 기능**이다. 쿠버네티스 표준 스펙엔 없지만 컨트롤러가 알아듣고 그대로 nginx 설정 파일에 끼워 넣는다. 사용하려면 컨트롤러 configmap에서 `allow-snippet-annotations: "true"`가 켜져 있어야 한다.

**결과:**
- 일반 요청 → `200 OK`
- 모바일 UA 요청 → `301 Moved Permanently` + `Location: http://jpub.tistory.com/`

> ⚠️ 짧은 UA 문자열(`iPhone; CPU iPhone OS 14_0`)로는 리다이렉트가 안 됐다. 정규식이 찾는 `Mobile`이라는 단어 자체가 UA에 없었기 때문 — **설정이 정확해도 테스트 입력값이 조건과 안 맞으면 실패한다**는 걸 확인.

**디버깅 순서:** YAML 문법 → configmap 설정 → `kubectl exec ... cat /etc/nginx/nginx.conf`로 실제 반영된 설정 확인 → 마지막으로 테스트 입력값. annotation이 반영 안 된 것 같을 때는 nginx.conf를 직접 까보는 게 제일 확실하다.

---

### 결론

| 구분 | 판단 기준 |
|---|---|
| **NodePort (L4)** | IP, 포트만 — 헤더 같은 건 볼 수 없어 조건부 라우팅 불가능 |
| **인그레스 (L7)** | 경로·도메인은 물론 **헤더 등 HTTP 요청 전체**로 라우팅 가능 |

인그레스는 host/path 라우팅뿐 아니라 HTTP 요청 안의 어떤 정보로도 조건부 라우팅이 가능하다. 표준 스펙을 벗어나는 세밀한 제어는 컨트롤러별 annotation으로 확장한다.

---

### 참고: `-f` 옵션

`kubectl apply -f <파일>`의 `-f`는 file. 파일이든 URL이든 폴더든 받을 수 있고, 매니페스트를 파일로 관리해야 git으로 이력이 남고 재현이 가능하다.

---

---

---

---

---

# Chap 6. 쿠버네티스 배포와 클러스터 구축

## 작업 관리 앱 구성

---
~ 6.2.2 까지는 단순 생성 관련 명령어들이라 제외

---

## 6.2.3 MySQL 배포 — 왜 볼륨이 문제인가

### 먼저, Compose가 뭐였는지 (4장 복습)

4장에서 taskapp을 만들 때 `docker compose up` 한 번으로 컨테이너 6개(nginx-web, web, nginx-api, api, mysql, migrator)를 동시에 띄웠다. 그게 **도커 컴포즈**다.

```
호스트 1대
 ├─ nginx-web
 ├─ web
 ├─ nginx-api
 ├─ api
 ├─ mysql
 └─ migrator
```

**핵심 특징: compose는 호스트 한 대 안에서만 동작한다.** 여러 컨테이너를 관리해주지만, 그 컨테이너들이 서로 다른 컴퓨터에 나뉘어 뜨는 일은 없다.

지금 6장은 이 taskapp을 컴포즈가 아니라 **쿠버네티스로** 옮기는 작업이라서, "컴포즈 때는 이랬는데 쿠버네티스에서는 왜 달라지는가"를 비교하며 이해해야 한다.

---

### Compose에서는 볼륨 문제가 없었던 이유

```
호스트 A (하나뿐)
 └─ mysql 컨테이너
     └─ 데이터 볼륨 (호스트 A의 디스크)
```

컨테이너가 죽어도 볼륨은 호스트 A 디스크에 그대로 남는다. 컨테이너가 다시 뜨면 **무조건 같은 호스트 A**에서 뜨기 때문에, 그 디스크에 다시 연결되는 게 당연했다.

즉, 컴포즈는 **컨테이너와 호스트의 관계가 사실상 고정**돼 있었다 — 애초에 호스트가 하나뿐이니 "어느 호스트에 뜰까"를 고민할 필요조차 없었던 것.

---

### 쿠버네티스에서는 그대로 못 쓰는 이유

쿠버네티스는 노드가 여러 대고, **파드가 어느 노드에 뜰지는 그때그때 스케줄러가 정한다.** (5.4에서 배운 내용) 죽었다가 다시 뜬 파드가 이전과 다른 노드에 배치될 수 있다는 뜻이다.

```
노드 A                       노드 B
 └─ mysql 파드 (죽음)          └─ mysql 파드 (다시 뜸)
     └─ 데이터가 여기 있었는데       └─ 여긴 그 데이터가 없음!
```

**노드 A의 디스크에 데이터가 있는데 파드는 노드 B에서 떠버리는** 상황이 발생한다. Compose 때는 호스트가 하나라 신경 쓸 필요 없던 문제가, 노드가 여러 대인 쿠버네티스 환경에서는 그대로 터진다.

> 컨테이너-호스트 관계가 컴포즈에서는 **고정적**이었다면, 쿠버네티스에서는 **유동적**이다. "노드는 그냥 자원 풀이고 파드가 어디든 배치될 수 있다"는 쿠버네티스의 장점이, DB처럼 상태를 유지해야 하는 컨테이너 입장에서는 오히려 문제가 되는 지점이다.

---

### 쿠버네티스의 해법: 외부 스토리지

볼륨을 **특정 노드에 묶지 않고, 노드 밖의 외부 스토리지**에 둔다.

```
노드 A            노드 B            외부 스토리지 (예: EBS)
 └─ mysql 파드      └─ mysql 파드         └─ 실제 데이터
    (죽음)             (다시 뜸)
                        ↓ 자동으로 재연결
```

파드가 다른 노드에서 다시 뜨면, **쿠버네티스가 그 외부 스토리지를 새 노드에 자동으로 할당**해준다. 노드가 어디든 상관없이 데이터가 파드를 따라갈 수 있다.

> 💡 **비유**
> - Compose 방식 = 데이터를 책상 서랍에 보관. 책상(호스트)이 바뀌면 서랍도 못 씀
> - 쿠버네티스 방식 = 데이터를 클라우드 드라이브에 보관. 어느 컴퓨터로 접속하든 같은 데이터에 접근 가능

---

### 핵심 정리

| | Compose | 쿠버네티스 |
|---|---|---|
| 동작 범위 | 호스트 **1대** 안에서만 | 노드 **여러 대**에 걸쳐서 |
| 컨테이너-호스트 관계 | 고정 (호스트가 하나뿐) | 유동 (파드가 다른 노드로 재배치될 수 있음) |
| 볼륨 위치 | 컨테이너가 도는 호스트 그 자체 | 노드 밖의 외부 스토리지 |
| 파드가 다른 곳에서 재기동될 때 | 해당 없음 | 외부 스토리지가 자동으로 재할당됨 |

> **"호스트와 데이터 볼륨의 밀접한 연결이 느슨해진다"** — 파드가 어느 노드에 뜨든 데이터 접근에 지장이 없어진다는 뜻. 이 덕분에 DB처럼 지속성 데이터를 다루는 애플리케이션도 쿠버네티스에서 자유롭게 운영할 수 있게 된다.

---

### PV / PVC 상세

쿠버네티스는 스토리지를 사용하기 위해 **PersistentVolume(PV)**과 **PersistentVolumeClaim(PVC)** 리소스를 제공한다. 클러스터가 구축되는 플랫폼(AWS, GCP, 로컬 등)에 대응하는 지속성 볼륨을 생성하기 위한 리소스다.

- **PV = 스토리지 그 자체.** 실제로 존재하는 저장 공간
- **PVC = 스토리지 추상화 리소스.** 파드는 이를 통해 필요한 용량의 PV를 **동적으로** 확보한다

> **"동적으로"** — PV를 미리 손으로 만들어두지 않아도, PVC가 "이만큼 필요하다"고 요청하면 클러스터가 알아서 그 스펙에 맞는 PV를 만들어준다. 이를 **동적 프로비저닝(dynamic provisioning)**이라 한다.

```
PV (실제 재고)  ←── 바인딩 ──  PVC (요청서)  ←── 파드가 마운트해서 사용
```

파드는 PV를 직접 알지 못하고, PVC를 통해 간접적으로 연결된다. (서비스가 파드를 직접 모르고 라벨로 찾았던 것과 비슷한 한 겹의 추상화)

---

### 마운트란?

**외부 저장 공간을 컨테이너 안의 특정 경로에서 접근 가능하게 연결하는 것.**

USB를 컴퓨터에 꽂으면 "D 드라이브"로 인식되어 그 드라이브 경로로 파일을 읽고 쓸 수 있게 되는 것과 같은 개념. 컨테이너는 원래 자기 안의 파일만 볼 수 있는데, 외부 볼륨을 특정 경로에 마운트하면 그 경로가 마치 자기 폴더인 것처럼 접근 가능해진다.

```yaml
containers:
- name: mysql
  volumeMounts:
  - name: data
    mountPath: /var/lib/mysql
```

mysql 컨테이너 입장에서는 `/var/lib/mysql`에 파일을 쓰는 것뿐이지만, 실제로는 그 데이터가 외부 스토리지(PV)에 저장된다. 4장 Compose에서 `-v` 옵션으로 호스트 폴더를 컨테이너에 연결했던 것(bind mount)과 같은 개념이며, 쿠버네티스에서는 이를 PV/PVC로 구현한다.

---

### PVC 매니페스트 예시

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

**`accessModes`** — 파드에서 스토리지로 마운트하는 정책.

| 모드 | 뜻 |
|---|---|
| `ReadWriteOnce` (RWO) | **하나의 노드**에서만 읽기/쓰기 가능 |
| `ReadOnlyMany` (ROX) | 여러 노드에서 동시에 읽기만 가능 |
| `ReadWriteMany` (RWX) | 여러 노드에서 동시에 읽기/쓰기 가능 |

- RWO가 제일 제한적이고, ROX·RWX로 갈수록 느슨해진다.
- ROX·RWX는 **일부 플랫폼에서 사용 불가**할 수 있으므로 주의. 일반 디스크(AWS EBS 등)는 물리적으로 한 서버에만 붙을 수 있는 구조라 여러 노드가 동시에 쓰는 것이 원천적으로 불가능하며, RWX를 쓰려면 NFS·EFS처럼 애초에 다중 접근용으로 설계된 스토리지가 필요하다.
- MySQL은 단일 파드(`mysql-0`)에서만 돌기 때문에 **RWO로 충분**하다.

**`resources.requests.storage`** — 필요한 볼륨 용량 지정.

파드에서 PVC로 요청한 볼륨을 직접 마운트할 수 있으며, 데이터를 볼륨에 저장해두면 **파드가 정지하거나 재생성되어도 애플리케이션 상태가 유지**된다.

---

---

## StatefulSet — 상태 유지 레플리카셋

**스테이트풀셋 = 상태 유지 레플리카셋.** 파드 복제, 컨테이너/환경변수 정의는 레플리카셋과 동일. 차이는 `volumeClaimTemplates`로 파드마다 전담 PVC를 자동 생성한다는 점.

| | 레플리카셋 | 스테이트풀셋 |
|---|---|---|
| 파드 이름 | 랜덤, 재생성마다 바뀜 | `mysql-0` 등 **고정 식별자**, 재생성돼도 유지 |
| 적합 대상 | 무상태 앱 (api, web) | 상태 유지 앱 (mysql) |

**Q. api/web과 mysql이 둘 다 "회원 조회/생성" 같은 행위를 하는 건 같은데 왜 다르게 취급하나?**
→ 행위가 같냐가 기준이 아니라 **결과 데이터가 어디 저장되냐**가 기준. api는 처리 결과를 mysql에 저장하므로 api 자신은 상태가 없음(stateless) → 아무 파드가 처리해도 무관. mysql은 자기 볼륨에 직접 저장하므로 **그 데이터를 가진 그 파드**가 응답해야 함 → 지목 필요.

**mysql.yaml 핵심 구성:**
- 이미지: `ghcr.io/jpubdocker/taskapp-mysql` (compose 때와 달리 완성된 이미지 사용, 태그 없음)
- 비밀번호: 시크릿을 볼륨 마운트 → 그 파일 경로를 `MYSQL_ROOT_PASSWORD_FILE` 등 환경변수로 지정 (매니페스트에 평문 노출 안 함)
- `clusterIP: None` → **Headless 서비스**. DB는 로드밸런싱되면 안 되므로 필수
- `serviceName: "mysql"` → 파드에 `mysql-0.mysql`이라는 **불변 호스트명** 부여

**Q. IP 대신 호스트명이 필요한 이유, `mysql-0`에 직접 접속한다는 게 무슨 뜻?**
→ 파드 IP는 재생성마다 바뀌지만 호스트명은 안 바뀜. "mysql-0에 접속"은 API를 거치지 않고 **직접 SQL 실행/백업/디버깅**할 때를 말함 (api는 아무 파드나 접속해도 되지만 DB는 데이터를 가진 그 파드를 정확히 찍어야 함).

**결과 확인:** `mysql-0` 파드 생성 → PVC(`mysql-data-mysql-0`)가 **손으로 안 만들었는데 자동 생성**되어 `Bound` 확인. `volumeClaimTemplates`가 파드 이름과 조합해 PVC를 자동으로 찍어낸 것.

**Q. `mysql-0`이 PV냐?**
→ 아님. `mysql-0`은 파드. PVC는 `mysql-data-mysql-0`, 그게 Bound된 실제 PV는 `pvc-e16f6a66-...`. 셋 다 다른 리소스.

**Q. `ReadWriteOnce`가 "한 번만 읽고 쓴다"는 뜻이냐?**
→ 아니고 "**하나의 노드**에서만 마운트 가능"이라는 뜻. 횟수 제한이 아니라 동시 접근 가능한 노드 개수 제한.

---

## Job — 데이터베이스 마이그레이터

레플리카셋/스테이트풀셋은 "파드가 계속 살아있어야 한다"고 가정. migrator는 **한 번 실행하고 끝나야 하는 작업**이라 Job이 필요.

**Q. Job이 파드보다 상위 개념이냐?**
→ 아니다. 레플리카셋·스테이트풀셋과 **같은 급**의, 파드를 만드는 방식이 다른 리소스. 파드가 끝났을 때 반응이 다를 뿐:
- 레플리카셋 등: 파드 종료 → 다시 만듦 (계속 재시작 루프 위험)
- **Job**: 파드가 정상 종료(Exit 0)하면 → 그대로 둠

**migrator.yaml 핵심:**
- `apiVersion: batch/v1` (파드/서비스의 `v1`, 디플로이먼트의 `apps/v1`과 다른 그룹)
- `env`는 대부분 평문, **비밀번호만** 시크릿 파일 경로로 `args`에 전달
- `restartPolicy: Never` — Job에 필수, 재시작 안 함을 명시

**결과:** `migrator-up-xxx` 파드가 `STATUS: Completed`, `READY: 0/1`로 남음 — 정상. (5.10의 Job 파드들과 같은 패턴)

---

## API 서버 배포

API는 상태 없는 앱이라 **디플로이먼트**로 구축. `nginx-api` + `api` 두 컨테이너를 가진 파드 (사이드카 구조).

**핵심:**
- `BACKEND_HOST: "localhost:8180"` — 같은 파드 내 api 컨테이너를 localhost로 호출 (파드 IP 공유)
- `secret.items` — 시크릿 안 특정 키만 골라 원하는 파일명으로 마운트
- 서비스는 nginx-api의 포트(80)를 바라봄 — api 컨테이너 자체는 8180

apply 결과: `deployment.apps/api created`, `service/api created`. `READY 2/2` 확인.

---

## Web 서버 배포 (6.2.3 마무리)

**Init 컨테이너**가 새로 등장: 본 컨테이너보다 먼저 실행되는 전처리 전용 컨테이너.

**필요한 이유:** `nginx-web`이 정적 파일(assets)을 참조해야 하는데, compose 때는 web-nginx 간 공유 볼륨으로 해결했음. 쿠버네티스에서도 공유 볼륨(`emptyDir: {}`)을 만들고, **Init 컨테이너가 이미지 안의 assets를 그 볼륨에 복사**해두는 방식으로 재현.

**핵심:**
- `initContainers`는 `containers`보다 먼저 실행되고 끝나야 본 컨테이너 시작
- Init 컨테이너와 web 컨테이너가 같은 이미지 사용 (이미지 안에 assets 포함)
- `web` 컨테이너의 `--api-address=http://api:80` → **다른 서비스**(api)를 이름으로 호출 (localhost 아님)
- Ingress `host: localhost`로 로컬 접속 경로 완성

**apply 결과:** Deployment, Service, Ingress 세 개 생성. `ingress`의 `ADDRESS`가 `localhost`가 되면 요청 수신 가능 상태. 브라우저에서 `http://localhost` 접속 → Task Management Application 화면 표시. **taskapp 전체(mysql→migrator→api→web) 로컬 배포 완료.**

---

## 6.3 클라우드(AKS) 배포

로컬 인그레스는 온라인 공개 불가. **Azure Kubernetes Service(AKS)**로 클라우드에 배포해 실제 온라인 공개.

**컨텍스트(context)** — 여러 클러스터를 다룰 때 조작 대상을 전환하는 설정. `kubectl config get-contexts`로 확인, `az aks get-credentials`로 AKS 컨텍스트 병합.

**배포 흐름은 로컬과 완전히 동일** (시크릿 → mysql → migrator → api), **인그레스만 다름:**
- 로컬: `ingressClassName: nginx` (Ingress NGINX Controller)
- AKS: `ingressClassName: azure-application-gateway` (AGIC, Application Gateway Ingress Controller)

apply 후 `kubectl get ingress web`의 `ADDRESS`에 **글로벌 IP**가 할당됨 → 그 IP로 실제 인터넷에서 접속 가능.

**매니지드 쿠버네티스 대시보드:** AKS는 Azure portal에서 웹 기반으로 리소스 상태/파드 로그를 거의 실시간 확인 가능. GKE, EKS도 유사한 대시보드 제공.

**학습 종료 후 클러스터 삭제 필수** (비용 방지):
```bash
az group delete --name jpub --yes --no-wait
```
---

## Web 서버 배포 — 6.2.3 마무리

**Init 컨테이너** 개념 등장: 본 컨테이너보다 먼저 실행되는 전처리 전용 컨테이너.
```
둘 다 한 번 뜨고 끝난다 라는 공통점으로 인해 헷갈려?
Job         → 독립된 새 파드를 하나 만듦 (migrator-up-xxxxx)
Init 컨테이너 → 기존 파드 안의 컨테이너 목록에 하나 끼어들어감 (web 파드 안의 init)
** kubectl get pod 하면 web-xxxxx라는 파드 하나만 보임. init은 그 안에 숨어있어서 별도 파드로 안 보임
```

**필요한 이유:** `nginx-web`이 정적 파일(assets)을 참조해야 하는데, compose 때는 web-nginx 간 공유 볼륨으로 해결했음. 
쿠버네티스에서도 공유 볼륨(`emptyDir: {}`)을 만들고, **Init 컨테이너가 이미지 안의 assets를 그 볼륨에 복사**해두는 방식으로 재현.
```컴포즈때 기억안나서~
compose 때는 공유볼륨. 즉 web 컨테이너와 nginx-web 컨테이너가 하나의 볼륨을 서로 동시에 마운트하여 사용함.
직접 주고받는 게 아니라 중간에 공용 저장 공간(볼륨)을 뒀다는 뜻.
그걸 쿠버네티스에서는 파드 안에 컨테이너들이 emptyDir 볼륨 하나를 공유하는 구조로 그대로 옮긴 것.

근데 왜 Init 컨테이너가 필요해졌냐???
compose 때는 web 컨테이너가 뜰 때 자기가 알아서 자기 assets를 볼륨에 복사하는 로직이 있었을 수 있는데(또는 시작 스크립트로), 
쿠버네티스 매니페스트에서는 그런 커스텀 로직을 다시 정의하는 게 번거로워서 
— "assets 복사"라는 그 한 가지 작업만 전담하는 별도 컨테이너(Init)를 만들어서 처리한 것.
```

**핵심:**
- `initContainers`는 `containers`보다 먼저 실행되고, 끝나야 본 컨테이너가 시작됨
- Init 컨테이너와 web 컨테이너가 같은 이미지 사용 (이미지 안에 assets 포함되어 있어서)
- `web` 컨테이너의 `--api-address=http://api:80`[파드 내부에서 나가는 요청] → **다른 서비스**(api)를 이름으로 호출 (같은 파드가 아니므로 localhost 아님)
- Ingress `host: localhost`로 로컬 접속 경로 완성 [브라우저에서 들어오는 요청] → 실제 회사 서비스라면 host: myapp.com이었을 텐데, 로컬 실습이라 진짜 도메인이 없어서 그냥 localhost를 도메인처럼 쓴 것

**apply 결과:** Deployment, Service, Ingress 세 개 생성. `ingress`의 `ADDRESS`가 `localhost`가 되면 요청 수신 가능 상태.

**브라우저에서 `http://localhost` 접속 → Task Management Application 화면 표시.**

→ **taskapp 전체(mysql → migrator → api → web)를 로컬 쿠버네티스에 배포 완료.**

---

## 6.3 클라우드(AKS) 배포 

### 왜 클라우드 배포가 필요한가

로컬 인그레스는 **온라인에 공개할 수 없음.** 실제 서비스로 배포하려면 클라우드의 매니지드 쿠버네티스가 필요하며, 책은 **Azure Kubernetes Service(AKS)**를 예시로 사용.

### 컨텍스트(Context)

여러 클러스터를 다룰 때 조작 대상을 전환하는 설정.

```bash
kubectl config get-contexts
```

로컬(`docker-desktop`)과 AKS(`jpub-aks`) 등 여러 클러스터가 목록에 뜨고, `CURRENT`에 `*` 표시된 게 현재 조작 대상. `az aks get-credentials`로 AKS 클러스터 정보를 kubeconfig에 병합해 전환.

### 배포 흐름은 로컬과 동일, 인그레스만 다름

시크릿 → mysql(StatefulSet) → migrator(Job) → api(Deployment) 순서는 **로컬과 완전히 동일.**

차이는 인그레스 컨트롤러:
| 환경 | 컨트롤러 |
|---|---|
| 로컬 (Docker Desktop) | Ingress NGINX Controller (ingressClassName: nginx) |
| AKS | Application Gateway Ingress Controller, AGIC (ingressClassName: azure-application-gateway) |

apply 후 `kubectl get ingress`의 `ADDRESS`에 **글로벌 IP**가 할당되며, 이 IP로 실제 인터넷에서 접속 가능해짐.

### 매니지드 쿠버네티스의 대시보드

AKS는 **Azure portal**에서 웹 기반으로 리소스 상태와 파드 로그를 거의 실시간 확인 가능. GKE(구글), EKS(AWS)도 유사한 대시보드 제공. 매니지드 서비스는 쿠버네티스 외 컴포넌트(로드밸런서, DNS 등)도 함께 연동되는 경우가 많아 이런 대시보드가 유용함.

### 실습 시 주의사항 (참고용)

AKS는 **켜져 있는 시간만큼 과금**되는 서비스이므로, 실제로 실습했다면 학습 종료 후 클러스터를 반드시 삭제해야 함:

```bash
az group delete --name <리소스그룹명> --yes --no-wait
```

### 참고 — 자체 도메인/HTTPS로 공개하기

실제 운영에서는 IP가 아닌 **도메인 + HTTPS**로 서비스해야 함. 클라우드별 DNS·인증서 서비스:

| 클라우드 | DNS 서비스 | SSL/TLS 인증서 |
|---|---|---|
| 구글 클라우드 | Cloud DNS | Certificate Manager |
| AWS | Route 53 | AWS Certificate Manager |
| Azure | Azure DNS | Azure Key Vault |

또는 오픈소스 **cert-manager**를 사용하면 **Let's Encrypt**(무료 인증기관)로 인증서 발급·자동 갱신까지 가능.

### 참고 — kubectx / kubens

여러 클러스터·네임스페이스를 자주 전환할 때 쓰는 편의 도구.

```bash
kubectx docker-desktop   # 컨텍스트 전환
kubectx -                 # 이전 컨텍스트로 복귀
kubens taskapp             # 기본 네임스페이스 설정 → 이후 -n taskapp 생략 가능
```
---

## [실습 완료] Web 서버 배포 — 6.2.3 마무리

**Init 컨테이너** 개념 등장: 본 컨테이너보다 먼저 실행되는 전처리 전용 컨테이너.

**필요한 이유:** `nginx-web`이 정적 파일(assets)을 참조해야 하는데, compose 때는 web-nginx 간 공유 볼륨으로 해결했음. 쿠버네티스에서도 공유 볼륨(`emptyDir: {}`)을 만들고, **Init 컨테이너가 이미지 안의 assets를 그 볼륨에 복사**해두는 방식으로 재현.

**핵심:**
- `initContainers`는 `containers`보다 먼저 실행되고, 끝나야 본 컨테이너가 시작됨
- Init 컨테이너와 web 컨테이너가 같은 이미지 사용 (이미지 안에 assets 포함되어 있어서)
- `web` 컨테이너의 `--api-address=http://api:80` → **다른 서비스**(api)를 이름으로 호출 (같은 파드가 아니므로 localhost 아님)
- Ingress `host: localhost`로 로컬 접속 경로 완성

**apply 결과:** Deployment, Service, Ingress 세 개 생성. `ingress`의 `ADDRESS`가 `localhost`가 되면 요청 수신 가능 상태.

**브라우저에서 `http://localhost` 접속 → Task Management Application 화면 표시.**

→ **taskapp 전체(mysql → migrator → api → web)를 로컬 쿠버네티스에 배포 완료.**

---

## [책 내용 정리] 6.3 클라우드(AKS) 배포 — 실습은 진행하지 않음

> ⚠️ 아래 내용은 책의 진행 흐름 이해를 위한 정리이며, **실제로 AKS 클러스터를 만들거나 배포하는 실습은 하지 않았음.** 지금까지의 실습은 전부 로컬(Docker Desktop) 환경에서만 진행됨. 따라서 클러스터 삭제 등 비용 관련 조치도 해당 없음.

### 왜 클라우드 배포가 필요한가

로컬 인그레스는 **온라인에 공개할 수 없음.** 실제 서비스로 배포하려면 클라우드의 매니지드 쿠버네티스가 필요하며, 책은 **Azure Kubernetes Service(AKS)**를 예시로 사용.

### 컨텍스트(Context)

여러 클러스터를 다룰 때 조작 대상을 전환하는 설정.

```bash
kubectl config get-contexts
```

로컬(`docker-desktop`)과 AKS(`jpub-aks`) 등 여러 클러스터가 목록에 뜨고, `CURRENT`에 `*` 표시된 게 현재 조작 대상. `az aks get-credentials`로 AKS 클러스터 정보를 kubeconfig에 병합해 전환.

### 배포 흐름은 로컬과 동일, 인그레스만 다름

시크릿 → mysql(StatefulSet) → migrator(Job) → api(Deployment) 순서는 **로컬과 완전히 동일.**

차이는 인그레스 컨트롤러:
| | 컨트롤러 |
|---|---|
| 로컬 (Docker Desktop) | Ingress NGINX Controller (`ingressClassName: nginx`) |
| AKS | Application Gateway Ingress Controller, **AGIC** (`ingressClassName: azure-application-gateway`) |

apply 후 `kubectl get ingress`의 `ADDRESS`에 **글로벌 IP**가 할당되며, 이 IP로 실제 인터넷에서 접속 가능해짐.

### 매니지드 쿠버네티스의 대시보드

AKS는 **Azure portal**에서 웹 기반으로 리소스 상태와 파드 로그를 거의 실시간 확인 가능. GKE(구글), EKS(AWS)도 유사한 대시보드 제공. 매니지드 서비스는 쿠버네티스 외 컴포넌트(로드밸런서, DNS 등)도 함께 연동되는 경우가 많아 이런 대시보드가 유용함.

### 실습 시 주의사항 (참고용)

AKS는 **켜져 있는 시간만큼 과금**되는 서비스이므로, 실제로 실습했다면 학습 종료 후 클러스터를 반드시 삭제해야 함:

```bash
az group delete --name <리소스그룹명> --yes --no-wait
```

### 참고 — 자체 도메인/HTTPS로 공개하기

실제 운영에서는 IP가 아닌 **도메인 + HTTPS**로 서비스해야 함. 클라우드별 DNS·인증서 서비스:

| 클라우드 | DNS 서비스 | SSL/TLS 인증서 |
|---|---|---|
| 구글 클라우드 | Cloud DNS | Certificate Manager |
| AWS | Route 53 | AWS Certificate Manager |
| Azure | Azure DNS | Azure Key Vault |

또는 오픈소스 **cert-manager**를 사용하면 **Let's Encrypt**(무료 인증기관)로 인증서 발급·자동 갱신까지 가능.

### 참고 — kubectx / kubens

여러 클러스터·네임스페이스를 자주 전환할 때 쓰는 편의 도구.

```bash
kubectx docker-desktop   # 컨텍스트 전환
kubectx -                 # 이전 컨텍스트로 복귀
kubens taskapp             # 기본 네임스페이스 설정 → 이후 -n taskapp 생략 가능
```


---
# Chap 7. 쿠버네티스 활용

---

## 7.1 파드 배포 전략

컨테이너로 애플리케이션을 배포하는 시대가 되면서 배포 전략도 바뀜.

[📖] 컨테이너를 통한 배포는 각 노드가 컨테이너 이미지를 가져오는 PULL타입 배포 -> 배포와 스케일 아웃이 쉽다.

-> 배포할 때 누가 누구한테 파일을 옮기느냐의 차이를 말하는 것.

```예전방식
[예전 방식]

배포 담당자/스크립트 ──scp──▶ 서버1
                    ──scp──▶ 서버2
                    ──scp──▶ 서버3   (서버가 늘면 목록도 스크립트도 다 고쳐야 함)
```
- 서버 목록을 **배포하는 쪽**이 들고 있어야 함. 서버가 하나 늘면 그 목록에 직접추가.
- 파일을 보낸 다음에도 서버마다 접속해서 압축 풀고, 재시작하고, 순서 맞추는 스크립트가 필요.
- 책에서 말하는 "번거로운 방식"

```컨테이너방식
[컨테이너 방식]

           레지스트리 (ghcr.io, Docker Hub)
          ▲          ▲          ▲
        pull       pull       pull
          │          │          │
        노드1      노드2      노드3
```
- 배포 하는 쪽은 이미지를 레지스트리에 한 번 올리고, 매니페스트에 image: ...:v.0.0.2 이런식으로 적기만 하면 됨.
- 그럼 파드가 배치된 노드의 kubelet이 그 이미지를 직접 받아 실행
- 배포하는 쪽은 노드가 몇 대인지, 어떤 노드인지 몰라도 상관 X

#### 🔎그래서 왜 "배포와 스케일 아웃이 쉽다"는 거임?

| 상황 | PUSH (예전 방식) | PULL (컨테이너 방식) |
|---|---|---|
| 새 버전 배포 | 서버마다 파일을 보내는 스크립트 실행 | 매니페스트의 이미지 태그만 변경 |
| 같은 앱 3개 → 10개로 늘리기 | 새 서버에 실행 환경 세팅 + 파일 전송 | `replicas: 10` 수정. 파드가 배치된 노드가 알아서 이미지를 pull |
| 서버(노드) 추가 | 배포 스크립트의 서버 목록 수정 | 클러스터에 노드를 붙이기만 하면 끝 |

- 핵심: 이미지 안에 실행 환경까지 다 들어 있어서 노드는 "받아서 실행"만 하면 됨
- 그래서 노드 수가 늘어도 배포하는 쪽이 할 일은 늘지 않음

#### 🔎스케일 아웃?
처리할 사람(대수)을 늘려서 감당할 수 있는 양을 키우는 것.

- 스케일 업 => 요리사 한 명을 더 빠르 셰프로 교체 => 서버 한 대의 CPU·메모리를 업그레이드
- 스케일 아웃 => 요리사를 3명에서 10명으로 늘림 => 같은 앱을 띄운 서버(파드)를 3개에서 10개로 늘림

즉, `replicas: 10` 수정 .. 이 부분이 스케일 아웃이라는 거임. 

예전에는 서버를 한 대 늘릴때마다 실행 환경 깔고 랄지 부르스 해야해서 그 자체가 일이었는데, 컨테이너는 저 `replicas` 숫자만 바꾸면 새 파드가 뜬 노드가 이미지를 알아서 받아오니까 늘리는 비용이 거의 0이 된 셈이라는 뜻~!~!~!~!~!~!

#### 🔎kubelet은 뭐에요

- **모든 노드에 하나씩 떠 있는 관리 프로그램(에이전트)**
- 컨트롤 플레인이 "이 노드에서 이 파드 돌려"라고 정하면, 그걸 노드에서 **실제로 실행**하는 주체

| 구성 요소 | 위치 | 역할 |
|---|---|---|
| kube-scheduler | 컨트롤 플레인 | 파드를 **어느 노드에** 놓을지 결정 |
| kubelet | 각 노드 | 배정된 파드를 **실행하고 계속 감시** |
| kube-proxy | 각 노드 | 서비스 IP → 파드 IP 네트워크 규칙 관리 (5장) |

1. API 서버를 감시하다가 자기 노드에 배정된 파드를 발견
2. 컨테이너 런타임(containerd)에 이미지 pull + 컨테이너 실행 지시 → **PULL 타입 배포의 실제 수행자**
3. liveness/readiness probe 실행 (7.1.2), liveness 실패 시 컨테이너 재시작
4. 파드 상태(Running, READY 0/1 등)를 API 서버에 계속 보고

### [🤔] 혜영이 질문 5. 컨트롤 플레인이 '이 노드에서 이 파드 돌려'라고 명령하면 kubelet이 알아서 실행한다고 했는데 그 노드가 죽으면 어떻게 되나요?

kubelet이 보내던 heartbeat가 끊기면, 컨트롤 플레인은 약 40초 뒤 노드를 `NotReady`로 표시하고 그 노드 파드로 가는 트래픽을 차단한다.
이후 파드의 기본 1) toleration(300초)이 지나면 파드가 삭제 처리되고, Deployment는 부족한 파드를 다른 정상 노드에 새로 만든다.
즉시 복구되는 게 아니라 **감지 → 대기 → 재생성**의 시간이 걸리므로, replicas가 1이면 그 사이(대략 5분) 서비스가 중단된다.
StatefulSet은 이중 실행으로 데이터가 깨질 수 있어 파드를 자동으로 옮기지 않고, 노드가 확실히 죽은 걸 확인한 뒤 수동 조치한다.
(40초, 300초는 기본값이며 클러스터 설정에 따라 달라질 수 있다.)

** 1) 파드가 노드에 붙은 taint(오염 표시)를 감수하고 거기에 머물러도 된다고 선언하는 설정

① 지시 받기	| API 서버 → kubelet	"내 노드에 배정된 파드가 뭐지? 바뀐 거 있나?"	watch (구독)  
② 상태 보고	| kubelet → API 서버	"나 살아있어, 파드들 상태는 이래"	heartbeat (Lease, NodeStatus)

~파드 배포 전략 실습~

### 7.1.1 RollingUpdate

-> 우리 이거 5장에서 디플로이먼트 배울 때 들었던 개념!

다시 한 번 설명하자면 [📖]구 버전의 애플리케이션을 실행한 상태에서 새 버전의 애플리케이션을 가동하고 준비가 완료된 것 부터 순서대로 교체하는 구조.

```📸chap7-롤링업데이트
우리가 5장에서 롤링업데이트를 실습한 적이 있으나, 그와 다른 점은
5장에서는 레플리카셋 목록과 리비전(쿠버네티스 안쪽 시점)이었다면,
지금 확인하는 건 사용자 시점을 더 한 것!
가짜 사용자(update-checker)가 요청을 보내고 그 응답이 어떻게 왔는지를 자기 로그에 기록하여 우리가 확인.


앱이 응답하는 버전 문자열(VERSION=cb76260)은 이미지 태그가 아니라, 이미지를 빌드할 때 git에서 읽어온 값인데, 빌드 시점에 git 태그를 찾으면 v0.0.2처럼 태그 이름이 들어가고, 못 찾으면 대신 커밋 해시(cb76260)가 들어감. v0.0.1 이미지는 태그를 못 찾은 상태로 빌드돼서 해시가 찍힌 것..
```

### [🤔] 지민이 질문 1. RollingUpdate: 의 반대개념으로 recreate가 있던데 이 방식과 blue green 의 방식, 세 가지의 - 특징 및 차이점? 어떠한 상황(case)에서 쓰이는지?

| | RollingUpdate | Recreate | Blue/Green |
|---|---|---|---|
| **방식** | 옛 파드를 조금씩 새 파드로 교체 | 옛 파드를 전부 내린 뒤 새 파드 생성 | 새 버전 환경을 통째로 띄워두고 트래픽을 한 번에 전환 |
| **다운타임** | 없음 | **있음** | 없음 |
| **추가 리소스** | 조금 | 없음 | **2배** |
| **버전 공존** | 교체 중 v1·v2 동시 서비스 | 없음 | 전환 전엔 없음 |
| **롤백** | 느림 | 느림 | **빠름** (selector만 되돌림) |
| **쿠버네티스** | 기본 (default) | 기본 | 기본 아님 (Service 전환/Argo Rollouts 등) |
| **쓰는 상황** | 일반 웹/API 서비스 | 두 버전이 동시에 돌면 안 될 때 (스키마 비호환, 싱글턴, RWO 볼륨) | 사전 검증과 즉시 롤백이 중요한 서비스 |

#### update-checker 요청이 응답을 받기까지

```text
update-checker 파드
   │ curl http://print-version/
   ▼
CoreDNS: "print-version" → 서비스 IP          
   ▼
kube-proxy: 서비스 IP → 실제 파드 IP 중 하나  
   ▼
print-version 파드 → "VERSION=v0.0.2" 응답
   ▼
update-checker가 받아서 echo
```
- 로그 한 줄 = `[update-checker가 찍은 시각] + print-version이 보낸 응답`

#### 이미지 변경 방법 비교: edit / patch / set image

| 비교 | `kubectl edit` | `kubectl patch` | `kubectl set image` |
|---|---|---|---|
| 방식 | 에디터로 매니페스트를 열어 직접 수정 | 바꿀 부분만 JSON으로 전달 | 이미지 전용 단축 명령 |
| 에디터 | 열림 | 안 열림 | 안 열림 |
| 바꿀 수 있는 것 | 아무 필드나 | 아무 필드나 | 이미지만 |
| 적합한 상황 | 사람이 직접 확인하며 수정 | 스크립트·CI 자동화 | 이미지만 빠르게 교체 |
| 적용 후 동작 | RollingUpdate 시작 | 동일 | 동일 |

```bash
# edit: 에디터가 열리면 v0.0.1 → v0.0.2로 직접 고치고 저장
KUBE_EDITOR=nano kubectl edit deployment/print-version

# patch: 매니페스트 경로(spec.template.spec.containers)를 JSON으로 지정
kubectl patch deployment/print-version -p '{"spec":{"template":{"spec":{"containers":[{"name":"print-version","image":"ghcr.io/jpubdocker/print-version:v0.0.2"}]}}}}'

# set image: 컨테이너이름=새이미지
kubectl set image deployment/print-version print-version=ghcr.io/jpubdocker/print-version:v0.0.2
```

- 세 방법 모두 결과는 동일함 (디플로이먼트 image 필드 변경 → 새 레플리카셋 → RollingUpdate)
- 차이는 "누가, 어떻게 지시하느냐"뿐

#### RollingUpdate 동작 제어: maxUnavailable / maxSurge

파드가 여러 개일 때 **한 번에 몇 개씩 교체할지** 정하는 설정

```yaml
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 3
      maxSurge: 4
```

| 설정 | 의미 | 기본값 | 리스트 7.4 값 (replicas 4) | 결과 |
|---|---|---|---|---|
| `maxUnavailable` | 교체 중 **동시에 내려가 있어도 되는** 파드 최대 수 | replicas의 25% (소수점 내림) | `3` | 4개 중 3개까지 동시에 내려도 됨 → 최소 1개는 항상 살아 있어야 함 |
| `maxSurge` | replicas보다 **추가로 더 띄워도 되는** 파드 최대 수 | replicas의 25% (소수점 올림) | `4` | 4개를 추가로 띄워도 됨 → 순간 최대 8개까지 존재 가능 |

→ 시작하자마자 새 파드 4개 생성 + 옛 파드 3개 종료, 새 파드가 Ready 되면 남은 1개 종료

**트레이드오프**

| 크게 잡으면 | 장점 | 단점 |
|---|---|---|
| `maxUnavailable` ↑ | 교체가 빨라짐 | 남은 파드에 트래픽이 몰림 |
| `maxSurge` ↑ | 교체가 빨라짐 | 순간적으로 CPU·메모리가 더 필요함 |

- 책의 권장: 처음에는 `maxUnavailable: 1`로 하나씩 교체하는 게 안전함
- 참고: replicas 1이면 기본값이 maxUnavailable 0(내림), maxSurge 1(올림) → "하나도 내리면 안 되고 하나는 더 띄워도 됨" → 그래서 앞 실습에서 **새 파드를 먼저 띄우고 옛 파드를 내렸음**

#### 🔎트레이드 오프가 뭐에요
하나를 얻으면 다른 하나를 내줘야 하는 관계.

### 7.1.2 컨테이너 실행 시 상태 체크 설정하기

| 부분 | kubelet이 하는 검사 | 통과 조건 |
|---|---|---|
| `livenessProbe` → `exec: cat /var/tmp/live.txt` | 컨테이너 안에서 `cat` 명령을 실행해 봄 | 파일이 있으면 OK (이미지에 미리 들어 있음) |
| `initialDelaySeconds: 3`, `periodSeconds: 5` | 뜨고 3초 뒤부터 5초마다 검사 | - |
| `readinessProbe` → `httpGet: /hc :8080` | 컨테이너의 8080 포트로 HTTP 요청 | 200번대 응답이 오면 OK |
| `initialDelaySeconds: 15` | 뜨고 15초 동안은 검사를 안 함 → 그동안 계속 Not Ready | - |
| `timeoutSeconds: 3` | 3초 안에 응답이 없으면 실패로 처리 | - |

- liveness 실패 → 컨테이너 **재시작**
- readiness 실패 → 재시작 없이 **서비스 대상에서 제외** (요청을 안 보냄)


(`kubectl rollout restart` 후 관찰)

📸chap7-컨테이너 상태체크.png
| 줄 | 시점 | 의미 |
|---|---|---|
| `0/1 Running 1s` | 컨테이너 시작 | 프로세스는 떴지만 아직 **Not Ready** → 서비스가 요청을 보내지 않음 |
| (약 21초 대기) | | readiness 검사 전 대기(`initialDelaySeconds: 15`) + 첫 검사·보고 시간 |
| `1/1 Running 22s` | readiness 통과 | 이제야 서비스 대상에 포함 |
| `Terminating 105s` | 바로 다음 | 새 파드가 Ready가 된 **뒤에야** 옛 파드 종료 |

- 핵심: **Running인데 0/1인 구간** = 파드는 떴지만 앱은 아직 응답할 준비가 안 된 상태
- 15초가 아니라 21초인 이유: 15초는 검사를 "시작하는" 시점이고, 검사·보고에 몇 초가 더 걸림

#### probe 없을 때 vs readinessProbe 있을 때

| 비교 | probe 없음 | readinessProbe 있음 |
|---|---|---|
| Running → 1/1 | 거의 동시 (약 4초) | 약 21초 |
| 옛 파드 종료 시점 | 새 파드가 Running 되자마자 | 새 파드가 **readiness를 통과한 뒤** |
| Ready 판단 기준 | "프로세스가 떴다" | "앱이 실제로 응답했다" |

- probe가 없으면 프로세스만 떠도 Ready로 보고 옛 파드를 바로 내림
- Spring Boot처럼 기동이 오래 걸리는 앱은 그 사이 요청이 준비 안 된 새 파드로 가서 에러 발생 → readinessProbe로 방지

---
### [🤔] 혜영이 질문 3. readiness와 liveness는 왜 실패했을 때 다르게 처리하나요? 둘의 목적이 뭔가요?

#### 1. 한 줄 답
두 probe는 **서로 다른 질문**에 답한다. 질문이 다르면 "실패"가 의미하는 상황도 다르고, 그 상황에 맞는 조치도 달라진다.

| | liveness | readiness |
|---|---|---|
| 질문 | 이 컨테이너가 **회복 불가능한 상태**는 아닌가? | 지금 **트래픽을 받아도 되는 상태**인가? |
| 실패의 의미 | 앱이 망가졌다 (스스로 못 고친다) | 지금은 못 받는다 (일시적일 수 있다) |
| 조치 | 컨테이너 **재시작** | Service 대상에서 **제외** (재시작 없음) |
| 목적 | 자동 복구 (self-healing) | 무중단 서비스 (트래픽 보호) |

---

#### 2. 목적

**liveness: "죽은 것을 살려낸다"**
- 프로세스는 떠 있지만 앱이 먹통인 상태(데드락, 무한루프, 스레드 풀 고갈, 메모리 문제 등)는 쿠버네티스가 기본으로는 알 수 없다. ("Running"으로 보인다.)
- liveness가 이를 감지하면 kubelet이 컨테이너를 재시작해서 사람 개입 없이 복구한다.

**readiness: "준비 안 된 곳으로 요청이 가지 않게 한다"**
- 기동 중 일시적 과부하, 의존 시스템 순간 장애처럼 **앱은 멀쩡하지만 지금은 요청을 처리할 수 없는** 구간이 있다.
- readiness가 실패하면 해당 파드를 Service 엔드포인트에서 빼서 사용자 요청이 에러로 가는 걸 막는다. 준비되면 자동으로 다시 들어온다.
- 롤링 업데이트에서도 사용된다. 새 파드가 readiness를 통과해야 옛 파드를 내린다. (무중단 배포의 근거)

---

#### 3. 실패했을 때 내부 동작

**liveness 실패**
1. kubelet이 `failureThreshold` 횟수만큼 연속 실패를 확인한다.
2. 해당 **컨테이너**를 종료한다 (SIGTERM → 유예 시간 → SIGKILL).
3. `restartPolicy`에 따라 같은 파드 안에서 컨테이너를 다시 띄운다. (파드가 교체되는 게 아니라 컨테이너 재시작이다. `RESTARTS` 카운트가 증가한다.)
4. 계속 실패하면 재시작 간격이 점점 늘어난다 (`CrashLoopBackOff`).

**readiness 실패**
1. kubelet이 파드의 `Ready` 조건을 `False`로 보고한다.
2. 엔드포인트 컨트롤러가 이 파드를 Service의 엔드포인트 목록에서 제외한다.
3. 컨테이너는 **재시작되지 않고 그대로 실행**된다.
4. probe는 계속 수행되며, 성공하면 `Ready=True`로 돌아와 다시 트래픽을 받는다. (**들어왔다 나갔다를 반복할 수 있다.**)

---

#### 4. 왜 다르게 처리해야 하나 (같은 처리를 하면 생기는 문제)

**(A) readiness 실패에 재시작을 적용하면**
- 기동이 느린 앱(Spring Boot 등)이 준비되기 전에 재시작되고, 다시 기동하는 동안 또 실패해서 **재시작 루프**에 빠진다. 영원히 준비 상태에 도달하지 못한다.
- 일시적 과부하나 의존 시스템 순간 장애에서도 앱이 재시작되며 상황이 악화된다. (재시작 시 캐시와 커넥션이 초기화되어 부하가 더 커진다.)
- 재시작은 "일시적으로 바쁜 상태"를 해결하지 못한다. 기다리면 되는 문제다.

**(B) liveness 실패에 "트래픽 제외"만 적용하면**
- 데드락처럼 **스스로 회복할 수 없는 상태**의 파드가 영원히 남는다. 트래픽을 못 받는 파드가 자원만 차지하고 복구 수단이 없다.
- 결국 사람이 직접 파드를 지워야 한다. 자동 복구라는 목적이 사라진다.

**정리: 일시적 문제 → 기다리며 트래픽만 차단 / 영구적 문제 → 재시작**
"재시작이 효과가 있는 문제인가?"가 두 probe를 나누는 기준이다.

---

#### 5. 비유 (식당)
- **liveness** = 주방장이 쓰러졌는지 확인. 쓰러졌으면 교대 투입(재시작).
- **readiness** = 주방장이 재료 손질 중인지 확인. 손질 중이면 쫓아내지 않고 **손님만 안 받는다.** 준비되면 다시 받는다.

---

#### 6. 같이 쓰는 이유 (역할 분담)

| 상황 | liveness | readiness | 결과 |
|---|---|---|---|
| 정상 | OK | OK | 트래픽 수신 |
| 기동 중 | OK | 실패 | 재시작 없이 대기, 트래픽 없음 |
| 일시 과부하 | OK | 실패 | 트래픽만 빠짐, 부하 해소 후 복귀 |
| 데드락 | 실패 | (보통 같이 실패) | 컨테이너 재시작 |

---

#### 7. 설정 예시 (실습 기준)
```yaml
livenessProbe:
  exec:
    command: ["cat", "/var/tmp/live.txt"]
  periodSeconds: 10
  failureThreshold: 3
readinessProbe:
  httpGet:
    path: /hc
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 10
```
- `initialDelaySeconds`: 첫 검사 전 대기. 실습에서 `1/1 Running`까지 약 21초 걸린 이유.
- `periodSeconds`: 검사 주기. `failureThreshold`: 연속 몇 번 실패해야 "실패"로 볼지.

---

#### 8. 실무 주의사항

| 주의 | 이유 |
|---|---|
| liveness에 외부 의존성(DB, 타 서비스) 체크를 넣지 않는다 | DB 장애 시 **모든 파드가 동시에 재시작**되어 연쇄 장애가 된다. 의존성 상태는 readiness에서 본다. |
| liveness를 너무 공격적으로 설정하지 않는다 | 부하로 응답이 느려진 정상 파드가 죽고 재시작되며 **재시작 폭풍**이 생긴다. |
| 느린 기동 앱은 `startupProbe`를 쓴다 | 기동이 끝나기 전에 liveness가 컨테이너를 죽이는 걸 막는다. (책에서는 다루지 않음) |
| 두 probe를 같은 엔드포인트로 만들 땐 의도를 확인한다 | 같은 검사를 하면 일시적 문제에도 재시작이 걸릴 수 있다. liveness는 "프로세스 응답 여부" 수준으로 가볍게 둔다. |
| readiness가 모든 파드에서 동시에 실패하면 서비스 전체가 빠진다 | 공통 의존성(DB 등)을 readiness에 넣을 때는 이 영향을 알고 선택해야 한다. |

---

#### 9. 한 줄 요약
**liveness = "고장 났으니 재시작", readiness = "아직 준비 중이니 잠깐 빼두기".**
재시작으로 해결되는 문제만 liveness에, 기다리면 해결되는 문제는 readiness에 맡긴다.


---

#### COLUMN: 안전하게 애플리케이션을 중지하고 파드 삭제하기 (Graceful Shutdown)

**파드 종료 순서**

```text
① 파드 삭제 명령
② preStop 훅 실행 (설정했을 때만)
③ SIGTERM 전송 → 앱이 정리 후 종료
④ terminationGracePeriodSeconds (기본 30초) 대기
⑤ 안 끝나면 SIGKILL → 강제 종료
```

| 상황 | 해결책 |
|---|---|
| 종료 정리가 오래 걸리는 앱 (DB 등) | `terminationGracePeriodSeconds: 60`처럼 대기 시간 늘리기 |
| SIGTERM에 즉시 죽는 앱 (nginx) | `lifecycle.preStop`에서 `nginx -s quit` 먼저 실행 |
| 종료 중 파드로 요청이 새는 문제 (실습에서 겪은 2분 멈춤) | `preStop`에 `sleep 5` 등으로 서비스 목록 갱신 시간 확보 (책 외 보충) |

- 원인: "파드 종료"와 "Endpoints에서 제거"가 동시에 따로 진행됨 → 그 틈에 요청이 죽어가는 파드로 감

>즉, 파드 삭제 명령이 떨어지면 terminationGracePeriodSeconds 카운트가 시작돼. preStop이 설정돼 있으면 먼저 실행되고, 그다음 SIGTERM 신호가 가서 앱이 정리를 해. 이 모든 게 정해진 시간 안에 끝나야 하고, 시간이 지나도 안 끝나면 SIGKILL 신호로 강제 종료돼.

** Graceful Shutdown: 누가 어디서 다루나

| 항목 | 어디에 쓰나 | 누가 담당 |
|---|---|---|
| `preStop` | **매니페스트** (`lifecycle.preStop`) | 우리가 설정 → kubelet이 실행 |
| `terminationGracePeriodSeconds` | **매니페스트** (파드 spec) | 우리가 설정 → kubelet이 그만큼 기다림 |
| SIGTERM 보내기 | 안 씀 (자동) | kubelet이 알아서 보냄 |
| SIGTERM 받아서 정리하기 | **앱 코드** (2장 main.go의 `signal.Notify`) | 개발자가 앱에 구현 |
| SIGKILL 보내기 | 안 씀 (자동) | 시간 초과 시 kubelet이 알아서 보냄 |

- 매니페스트: 종료 과정을 **조율** (먼저 할 일, 최대 대기 시간)
- 앱 코드: SIGTERM을 받으면 처리 중인 요청을 마무리하고 **스스로 종료**
- 쿠버네티스: SIGTERM → (시간 초과 시) SIGKILL을 **자동으로** 전송
- 매니페스트와 앱 코드가 둘 다 준비돼야 Graceful Shutdown이 제대로 동작함
---

### [🤔] 혜영이 질문 4. SIGTERM을 왜 kubelet이 보내나요? API 서버가 직접 컨테이너에 신호를 주지 않고 kubelet이 하는 이유가 뭔가요?

**API 서버는 컨테이너가 어디서 어떻게 돌고 있는지 모르고, 건드릴 수도 없다.**
컨테이너를 실제로 만지는 건 **그 노드에 있는 kubelet**뿐이라서, 신호도 kubelet이 보낸다.

| | 하는 일 | 컨테이너에 접근? |
|---|---|---|
| **API 서버** | 원하는 상태를 **기록**하고 알려주는 창구 (etcd에 저장) | X (컨테이너 런타임과 통신하지 않음) |
| **kubelet** | 자기 노드의 파드를 **실제로 실행·감시·종료** | O (containerd 같은 런타임에 직접 요청) |

- SIGTERM은 "프로세스에 보내는 OS 신호"다. 컨테이너 안 프로세스에 신호를 줄 수 있는 건 **같은 노드에서 컨테이너 런타임을 다루는 주체**뿐이다.
- API 서버는 노드 안에 들어와 있지도 않고, 컨테이너 런타임과 연결되어 있지도 않다.

**비유 (본사 vs 지점 점장)**
- API 서버 = 본사. 지침을 문서로 내려보낸다. ("A지점 파드 철수")
- kubelet = 각 지점 점장. 지침을 보고 직접 매장 직원(컨테이너)에게 "마무리하고 퇴근하세요"라고 말한다.
- 본사가 지점 직원에게 직접 지시하는 구조는 아니다.




### 7.1.3 Blue-Green Deployment
= 구버전과 신버전이 혼재하는 시간이 발생하는데, 이것이 의도치 않은 부작용이 발생할 수 있다.  
이 방법을 해결하는 방법 중 하나로 Blue-Green Deployment가 있다.  
[📖] B.G.D는 배포 시 기존에 배포된 서버 그룹과는 별도로 새로운 애플리케이션을 배포할 서버 그룹을 준비, 로드 밸런서와 서비스 디스커버리 레벨에서 참조 대상을 교체하여 배포하는 방법.

=> 핵심은 "서버를 고치지 않고 화살표만 바꾼다"  
```
RollingUpdate:  입구 → [v1][v1][v1] → [v2][v1][v1] → [v2][v2][v1] → [v2][v2][v2]
                       (서버를 하나씩 교체, 중간에 섞임)

Blue-Green:     입구 → [v1][v1][v1]  blue          입구    [v1][v1][v1]  blue
                       [v2][v2][v2]  green   →       ↘     [v2][v2][v2]  green
                       (입구 방향만 한 번에 전환, 안 섞임)
```
[📸]chap7-블루그린 캡쳐를 통해 딜레이 없이 버전이 바뀐 걸로 호출되는 걸 확인

---

## 7.2 쿠버네티스에서 정기적인 배치 작업 실행하기
### 7.2.1 크론잡

**쿠버네티스에서 앱을 돌리는 세 가지 방식

| 리소스 | 언제 쓰나 | 예시 | 배운 곳 |
|---|---|---|---|
| Deployment | **계속 떠 있어야** 하는 앱 | 웹, API 서버 | 5장 |
| Job | **한 번 실행하고 끝나는** 작업 | DB 마이그레이터 (taskapp migrator) | 6장 |
| CronJob | **정해진 시간마다 반복** 실행하는 작업 | 매일 새벽 통계 집계, 1시간마다 정리 작업 | 7장 |

```text
CronJob ──(스케줄마다)──→ Job ──→ Pod
Deployment ─────────────→ ReplicaSet ──→ Pod
```

- CronJob은 스케줄 시간마다 Job을 하나씩 찍어냄 (Deployment가 ReplicaSet을 거느리는 구조와 같음)

** Concurrency Policy

크론잡의 다음 실행 시간이 됐는데 **이전 Job이 아직 안 끝났으면 어떻게 할지** 정하는 설정 (`.spec.concurrencyPolicy`)

**언제 문제가 되나**: 작업 시간이 스케줄 간격보다 길면 겹침

```text
스케줄: 1분마다 / 작업 시간: 90초

07:00  Job A 시작 ──────────────────────── 07:01:30 끝
07:01          Job B 시작 시간!  ← A가 아직 도는 중인데 어떻게 하지?
```

| 값 | 동작 | 위 상황에서 07:01에 |
|---|---|---|
| `Allow` (기본값) | 겹쳐도 **같이 실행** | A와 B가 동시에 돌아감 |
| `Forbid` | 이전 Job이 안 끝났으면 **이번 회차는 건너뜀** | B를 안 만듦. 다음 07:02에 다시 판단 |
| `Replace` | 이전 Job을 **중단하고 새 Job으로 교체** | A를 중단하고 B를 시작 |

```yaml
spec:
  schedule: "*/1 * * * *"
  concurrencyPolicy: Forbid   # spec 바로 아래, schedule과 같은 레벨
  jobTemplate:
    ...
```

** Suspend

크론잡을 **지우지 않고 일시정지**하는 설정 (`.spec.suspend`, 기본값 `false`)

| 방법 | 설정 보존 | Job 기록 | 재개 |
|---|---|---|---|
| 크론잡 삭제 | 사라짐 | 사라짐 | 다시 apply |
| `suspend: true` | 그대로 | 그대로 | `false`로 바꾸면 끝 |

- 사용 예: DB 점검 중, 배치 버그 수정 중, 외부 API 장애 중
- 주의: **이미 실행 중인 Job은 멈추지 않음** → 앞으로 새 Job을 만들지 않을 뿐

```bash
kubectl patch cronjob pingpong -p '{"spec":{"suspend":true}}'    # 일시정지
kubectl patch cronjob pingpong -p '{"spec":{"suspend":false}}'   # 재개
```
---

### [🤔] 혜영이 질문 1. CronJob에서 Suspend를 사용하는 상황에는 어떤 것들이 있을까요? Suspend를 왜 사용할까요?

```
**왜**: CronJob을 지우면 스케줄·템플릿·이력 설정이 다 날아간다. Suspend는 **정의는 그대로 두고 새 Job 생성만 멈추는 스위치**다. `false`로 되돌리면 재개된다.

**그럼 언제 사용할까?** : 두가지 정도의 사례를 들어볼게.
1. DB 장애/과부하, 외부 결제 API 장애 중에 5분마다 도는 동기화 배치가 계속 실패하며 재시도 부하를 줄 때.
  -> 왜? : 코드 수정, 재배포 없이 즉시 끌 수 있고, 복구 후 원복이 단 한줄이면 되기때문.

2. 배포 후 매 분 실패하는 잡이 쌓이고, 알림/로그/파드 리소스가 폭증하는 상황일 때.
  -> 왜? : delete하면 실패 이력도 살아질 수 있어 원인 추적이 어려워지기때문.

공통점은 결과적으로 지우고싶진 않고, 잠깐만 코드 수정 없이 즉시 멈추고 싶을때 쓰는 스위치라고 보면 될 듯.
```


### 7.2.2 타임존을 고려한 크론잡 실행

- 크론잡 스케줄은 기본적으로 **UTC 기준** (kube-controller-manager의 시간대)
- 예: `schedule: "0 9 * * *"` → 한국 시간 오전 9시가 아니라 **오후 6시**에 실행

| 방법 | 문제점 |
|---|---|
| 예전: 컨트롤 플레인 서버의 시간대 변경 | 클러스터 전체가 한 시간대에 묶임, 매니지드 서비스(EKS 등)는 변경 불가 |
| 1.27부터: CronJob의 `timeZone` 필드 | 크론잡마다 개별 지정 가능 |

```yaml
spec:
  schedule: "10 12 * * *"
  timeZone: "Asia/Seoul"    
```
### 7.2.3 크론잡에서 잡을 일회성으로 실행하기

스케줄을 기다리지 않고 **크론잡의 jobTemplate으로 Job을 즉시 1개 생성**

```bash
kubectl create job [Job 이름] --from=cronjob/[CronJob 이름]
kubectl create job onetime-pingpong --from=cronjob/pingpong
kubectl logs job/onetime-pingpong      # Job 이름으로 바로 로그 확인
```

| 비교 | 스케줄로 실행 | `--from`으로 실행 |
|---|---|---|
| 실행 시점 | 스케줄 시간 | 명령 친 즉시 |
| Job 이름 | 자동 (`크론잡이름-숫자`) | 내가 지정 |
| 크론잡 설정 변경 | - | 없음 |

- 사용 예: 배포 직후 동작 확인, 실패한 회차 재실행, 급하게 한 번 더 실행
- 장점: 스케줄을 임시로 바꿨다가 되돌리는 걸 깜빡하는 사고를 막을 수 있음


---

### [🤔] 혜영이 질문 2. CronJob은 실무에서 많이 사용하지 않는데 그 이유는 뭘까요? 어떤 단점이 있고, 어떤 대체품이 있고, 그 대체품의 특징은 무엇인가요?
```
  자 일단 실무에서 많이 사용하지 않는다는 것도 몰랐구요
간단한 정기 배치에는 CronJob이 실제로 쓰이긴 한다네요.. 다만 복잡한 업무 배치에서는 한계가 있어 다른 도구를 같이 병행해서 쓴다고 합니다..
```
| 단점 | 설명 |
|---|---|
| 의존성 표현 불가 | "A 끝나면 B 실행" 같은 워크플로우를 만들 수 없음 |
| 중복/누락 가능 | 같은 스케줄이 두 번 돌 수 있음. 멱등성은 앱 책임 (`concurrencyPolicy`로 완화) |
| 재처리·백필 불편 | 과거 날짜 재실행, 실패 구간만 재처리 기능 없음 |
| 가시성 부족 | 실행 이력·성공률·소요시간을 볼 UI 없음 (Job이 일정 개수만 남고 정리됨) |
| 정밀도 | 최소 단위 1분, 정확한 시각 보장 아님 |
| 시간대 | 기본은 컨트롤 플레인 시간대. `timeZone` 필드는 비교적 최근에 추가됨 |
| 알람·재시도 빈약 | 실패 알림은 별도 구축 필요 |

**대체품**

| 대체품 | 한 줄 설명 | CronJob보다 나은 점 | 이럴 때 쓴다 |
|---|---|---|---|
| **Argo Workflows** | 쿠버네티스 안에서 "작업 순서도"를 실행해주는 도구 | 순서와 의존성을 정의할 수 있다 (A → B → C, 일부는 동시에) | 단계가 여러 개인 배치를 쿠버네티스에서 돌릴 때 |
| **Airflow** | 파이썬으로 작업 순서(DAG)를 짜고, 웹 화면에서 관리하는 도구 | 실행 이력 화면, 실패한 날짜만 재실행, 과거 데이터 재처리(백필) | 데이터 파이프라인, ETL |
| **Spring Batch** | 자바에서 "대량 데이터를 나눠 처리"하는 프레임워크 | 중간에 실패해도 **실패 지점부터 재시작**, 처리 이력을 DB에 저장 | 자바 백엔드의 정산·집계 같은 업무 배치 |
| **Jenkins** | CI/CD 도구에 있는 스케줄 기능 | 이미 쓰는 도구라 따로 구축할 게 없다 | 간단한 운영 스크립트 |
| **클라우드 스케줄러** (AWS EventBridge 등) | 클라우드가 대신 관리해주는 알람시계 | 직접 운영할 필요가 없다 | 클라우드를 이미 쓰고, 트리거만 필요할 때 |


**한 줄 정리**: CronJob은 "정해진 시각에 파드 하나 띄우는 것"까지만 책임진다. 의존성·재처리·모니터링이 필요하면 워크플로우 엔진이나 배치 프레임워크로 올려야 한다.


---

## 7.3 유저 관리와 Role-Based Access Control(RBAC)

**왜 필요한가**: 지금 쓰는 `docker-desktop` 계정은 클러스터 관리자 권한이라 뭐든 다 됨 → 실무에서는 위험
- 신입이 실수로 운영 DB 파드 삭제
- 조회만 하면 되는 사람이 배포 설정 변경
- 앱 하나가 해킹당하면 그 권한으로 클러스터 전체 조작

→ **필요한 만큼만 권한을 주는** 장치 = RBAC (Role-Based Access Control, 역할 기반 접근 제어)

**쿠버네티스의 유저 두 종류** 

| 구분 | 누가 쓰나 | 예시 | 관리 위치 |
|---|---|---|---|
| 서비스 어카운트 | **파드**(프로그램)가 쿠버네티스 API를 호출할 때 | 다른 파드 목록을 조회하는 모니터링 앱 | 쿠버네티스 리소스 (`kubectl get sa`) |
| 일반 유저 | **사람**이 kubectl로 조작할 때 | 개발자, 운영자 (회사 EKS kubeconfig) | 클러스터 밖에서 인증 (인증서, 토큰, OIDC 등) |

[🔎]서비스 어카운트 추가 설명
--->  
우리가 ** kubectl get pods ** 를 치면 실제로는 kubectl이 컨트롤 프레인의 api 서버에게 "파드 목록 줘" 라고 http 요청을 보내는 것.   
이게 쿠버네티스 API 호출!  
이 요청은 파드 안에서 도는 프로그램들도 똑같이 API 서버에 요청할 수 있음.

우리 5장에서 인그레스 만들면 ingress-nginx가 알아서 라우팅해줬던 거  
그것이 가능했던 이유는 ingress-nginx 파드가 API 서버한테 "인그레스 목록 줘" 라고 계속 물어보고 있었기 때문.  
사람은 kubeconfig에 든 인증서로 증명이 가능하지만, 파드는 뭘로 자기를 증명하냐? 그때 쓰는 신분증이 서비스 어카운트다.

서비스 어카운트는 파드나 서비스처럼 쿠버네티스 안에 리소스로 존재해서 kubectl로 조회하고 만들고 지울 수 있다.

### [🤔] 혜영이 질문 6. 인증 부분에서 서비스 어카운트와 일반 유저를 왜 나눠서 관리하나요? 둘 다 인증을 위한 것인데 하나로 퉁칠 수 있지 않나요?

쿠버네티스는 일반 유저(사람)를 클러스터 안의 오브젝트로 관리하지 않고 외부(인증서 등)에서 신원을 가져오는 반면,   
서비스 어카운트(파드)는 네임스페이스에 속한 리소스로 클러스터 안에서 만들고 지운다.  
둘을 합치면 수시로 생성·삭제되는 파드마다 외부 계정을 만들어야 하고,  
파드에 사람 인증서를 넣게 되어 유출 위험과 권한 과다 문제가 생기며,   
퇴사자 계정 삭제 같은 사람의 생애주기가 서비스 장애로 이어진다.  
그래서 파드는 자동 주입되는 토큰으로 인증하고 앱마다 서비스 어카운트를 따로 두어 
최소 권한과 감사 추적(사람/워크로드 구분)을 가능하게 한다.  
다만 인증 이후의 권한 부여는 둘 다 같은 RBAC(Role + Binding)으로 처리하므로, **인증은 분리하고 인가는 공통**으로 쓴다.

### 사람 계정과 서비스 어카운트를 합치면 안 되는 이유

| 문제 | 합쳤을 때 (사람 계정을 파드에 사용) | 분리했을 때 (서비스 어카운트) |
|---|---|---|
| **생성 속도** | 파드는 하루에도 수십 번 생기고 사라지는데, 그때마다 외부(HR/SSO)에서 계정을 발급받아야 한다 | 쿠버네티스 안에서 API로 즉시 생성·삭제 가능 |
| **사람이 없다** | 사람은 로그인(비밀번호, MFA)하지만 파드는 로그인할 사람이 없다 | 토큰이 파드에 **자동 주입**된다 |
| **퇴사 사고** | 홍길동 계정으로 앱이 API를 호출 중인데 퇴사로 계정이 삭제되면 **서비스 장애** | 사람의 생애주기와 무관하게 앱과 함께 관리된다 |
| **유출 위험** | 사람 인증서를 파드 이미지·설정에 넣으면 유출될 수 있고, 사람 계정은 권한이 넓어 피해가 크다 | 앱 전용 신분증이라 유출돼도 권한·범위가 제한적이다 |
| **최소 권한** | 사람 계정 권한을 그대로 써서 앱에 필요 이상의 권한이 부여된다 | 앱마다 SA를 따로 만들어 "파드 조회만" 식으로 권한을 좁게 준다 |
| **감사 로그** | "이 파드 누가 지웠어?"를 볼 때 사람이 한 건지 앱이 한 건지 구분이 안 된다 | 로그의 주체가 사람/서비스 어카운트로 구분된다 |

### 7.3.1 RBAC를 사용해 권한 제어 구현하기

**RBAC = 역할 정의 + 연결**

| 단계 | 회사 비유 | 쿠버네티스 |
|---|---|---|
| ① 역할 정의 | "열람실 출입 가능" 권한 카드 만들기 | Role / ClusterRole |
| ② 연결 | 그 카드를 직원에게 발급 | RoleBinding / ClusterRoleBinding |

| 비교 | 네임스페이스 하나에서만 유효 | 클러스터 전체에서 유효 |
|---|---|---|
| 무엇을 할 수 있나 (권한 목록) | Role | ClusterRole |

[📸] chap7-서비스어카운트 
```
네임스페이스마다 default 있는 거 확인.
네임스페이스를 만들면 쿠버네티스가 자동으로 하나씩 만들어주는 기본 신분증.
매니페스트에 serviceAccountName을 안적으면 파드는 이걸 달고 뜬다.

```

---

### 7.3.2 클러스터 롤 생성 [권한 카드 만들기]

| 필드 | 값 | 의미 |
|---|---|---|
| `apiGroups` | `[""]` | 코어 API 그룹 (빈 문자열 = 코어) |
| `resources` | `["pods"]` | 대상은 파드 |
| `verbs` | `["get", "watch", "list"]` | 하나 조회, 변화 감시, 목록 조회만 허락 |

→ **"모든 파드를 조회만 할 수 있음"** (create / update / delete 없음)

**apiGroups 값 찾기**: 매니페스트의 `apiVersion` 슬래시 앞부분 (`kubectl api-resources --sort-by name`)

[🔎] apiGroups로 랄지부르스 이유 뭐야? 뭔데 이게?
==> 그냥 규약이라고 생각하면 됨. 정해져있는 리소스고 갖다쓰면 되는 고정 값.

---

### 7.3.3 서비스 어카운트 생성 [직원 등록]

```
apiVersion: v1           # ServiceAccount는 코어 그룹 (슬래시 없음)
kind: ServiceAccount     # 직원(신분증)을 만든다
metadata:
  name: pod-reader       # 직원 이름은 pod-reader
```

** name이 같다고 자동연결 X 지금 여기까지는 권한 카드 만들었고, 직원 등록만 한 상태. 연결 XXX

---

### 7.3.4 클러스터 롤 바인딩 생성 [카드를 직원한테 주기]

```
kind: ClusterRoleBinding        # 카드를 직원한테 주는 "발급 기록"
metadata:
  name: pod-reader              # 발급 기록의 이름
subjects:                       # 누구에게? 
- kind: ServiceAccount          #   서비스 어카운트인
  name: pod-reader              #   pod-reader 직원에게
  namespace: default            #   (default 네임스페이스에 등록된)
roleRef:                        # 어떤 카드를? ↓
  kind: ClusterRole             #   ClusterRole인
  name: pod-reader              #   pod-reader 카드를
  apiGroup: rbac.authorization.k8s.io
```

여기까지 진행하면 이제 연결 된 것.
그럼 출입 테스트를 해보자.  

pod-reader 신분증으로 파드 띄우기
```
apiVersion: v1
kind: Pod
metadata:
  name: pod-reader
  labels:
    app: pod-reader
spec:
  serviceAccountName: pod-reader   # 이 파드는 pod-reader 신분증을 달고 실행된다
  containers:
   . . . . . . . . . 
```
=> [📖] 그림 7.6 pod-reader가 출력하는 파드 리스트 

``` serviceAccountName: pod-reader ``` 를 안적으면 default 신분증이 붙어버리고, default에는 아무 권한이 없기때문에 조회가 거절됨.

그럼, 같은 카드로 허락 안 된 걸 시도하면 거절당하는 지 확인해보자는 것이 다음.  

[📸]chap8-잘못된권한조회

pod-reader은 "파드" 대상 권한 okay인데, 상위 레벨인 디플로이먼트에 pod-reader 권한을 추가하고 가져오도록 명렁해보니
에러가 뜨는 걸 확인.


---

### 7.3.5 일반 유저 [직원카드 복사본으로 출입하는 임시출입증]

[📸]chap7-임시출입증발급로그
```
1. cluster -> 어느 건물 / authinfo -> 어떤 카드
2. 임시 출입증 발급
3. 내 지갑에 넣음.
4. 임시 출입증에 포스트잇 "docker-desktop 건물에는 local-pod-reader 출입증으로 들어간다" 를 붙여놓기.
5. 현재 내 지갑에는 카드 두장이 있는 것. 하나는 관리자 카드, 하나는 임시 출입증
[📸]chap7-임시출입증발급로그2
6. 카드 임시출입증으로 바꿔들고 들어가기~

==> 전부 이 전에 봤던 똑같은 결과가 터미널에 출력됨을 확인.
```

---
# Chapter 8. 쿠버네티스 애플리케이션 패키징

? 왜 필요한가 ?

- 지금까지는 로컬 환경 하나뿐이었음
- 실무에서는 **같은 앱을 여러 클러스터(환경)에 배포**함

| 환경 | 다른 점 |
|---|---|
| dev | replicas 1, 도메인 `dev-echo.회사.com` |
| prod | replicas 5, 도메인 `echo.회사.com` |

- 매니페스트를 환경마다 통째로 복사하면 → 이미지 버전 하나 올릴 때마다 모든 사본을 수정해야 함 → 하나라도 빠뜨리면 사고
- 해결: **공통 부분은 한 벌만 두고, 환경별 차이만 따로 관리**

| 도구 | 방식 | 저자의 구분 |
|---|---|---|
| Kustomize (8.1) | 원본 YAML + 바꿀 부분만 **덮어쓰기** | 환경별 **최소한의 차이**만 관리할 때 |
| Helm (8.2) | YAML **템플릿**에 값을 채워 넣기 | 세밀한 제어가 필요할 때, 남에게 배포하거나 오픈소스로 공개할 때 |

## 8.1 Kustomize 

환경별 차이를 적용하려면 먼저 바뀌지 않는 공통 부분이 있어야하고, 그걸 만들거야 (base)  

1️⃣ 웹앱 하나를 쿠버네티스에 올리는 데 필요한 최소 세트인   
Deployment / Service / Ingress 각각의 베이스를 만들자.  
[📸] 리스트 8.2 echo 애플리케이션의 디플로이먼트 매니페스트 파일 (~/k8s/kustomize/echo/base/deployment.yaml)  

2️⃣ 그리고 그 세개의 베이스를 하나로 합치자.
[📸]리스트 8.6 kustomize create --autodetect로 생성된 kustomization.yaml (~/k8s/kustomize/echo/base/
kustomization.yaml)

[🔎] 하나로 합치는 이유는 뭐야?
1. 폴더 하나를 배포단위 1개로 다룰 수 있다.
2. 공통 설정을 한 곳에서 한번에 입힐 수 있다.
3. overlay가 이 base를 가져다 쓸 수 있다.

3️⃣ 공통 레이블을 kustomization.yaml 한 곳으로 (p.306~308)
- 문제: `app.kubernetes.io/name: echo`가 deployment/service/ingress 3개 파일에 6번 중복
- 해결: kustomization.yaml에서 한 번만 설정
  - 리스트 8.13 (p.307) `kustomize edit set label "app.kubernetes.io/name:echo"`
  - 리스트 8.14 (p.307) `commonLabels`가 추가된 kustomization.yaml
  - `kustomize build .` 결과는 이전과 동일!
- `commonLabels`는 deprecated 예정 → `kustomize edit fix`로 변환하셔라~

4️⃣ 오버레이로 환경별 차이 적용 (p.309~312)
- 디렉터리 구조: `base/` (공통) + `overlays/dev/` (환경별)
- dev의 kustomization.yaml 생성 → `resources: ../../base` (오버레이 하기 전 베이스 가져오기~)
- 패치 방식 2가지

```
값을 덮어쓰려면 방법이 필요한데, 그 방법의 이름이 Strategic Merge Patch.
Deployment는 이 방식으로 됐는데 Ingress의 host는 이 방식으로 안됨.  
그 대안이 JSON Patch
```
** 패치 방식 선택 기준
- Strategic Merge Patch: 바꿀 부분만 원본 모양으로 작성. 리스트 항목은 name 같은 식별 값으로 짝을 맞춤
  - 예) replicas 변경, env의 BACKEND_MAX_FAILS value 변경 (식별 값은 그대로, 값만 변경)
- JSON Patch: op/path/value로 경로를 직접 지정
  - 예) Ingress host 변경 (host가 곧 식별 값이라 Strategic Merge로 위치를 못 찾음)
- 판단 기준: 식별 값(이름표)을 건드리지 않으면 Strategic Merge, 식별 값 자체를 바꾸거나 경로로 찍어야 하면 JSON Patch

[🔎] 이해가 너무 않되,,  
-->  
 #### Ingress `rules`와 host

- `rules`는 "이 host로 들어온 요청은 이 경로로 이 Service에 보내라"를 정의하는 **리스트**. 규칙을 여러 개 둘 수 있다.

```yaml
spec:
  ingressClassName: nginx          # 어떤 Ingress Controller가 처리할지 (ingress-nginx)
  rules:                           # 라우팅 규칙 목록
  - host: echo.jpub.local          # ① 이 도메인으로 들어오면
    http:
      paths:
      - pathType: Prefix
        path: /                    # ② 이 경로(/ 이하 전부)는
        backend:
          service:
            name: echo             # ③ echo Service의
            port:
              number: 80           #    80번 포트로 보내라
```

- 도메인별로 echo.jpub.local → echo, admin.jpub.local → admin Service로 나눌 수 있고,
  한 도메인 안에서도 `/api`와 `/`를 다른 Service로 보낼 수 있다.
- `host`는 규칙 하나를 구분하는 이름 역할을 한다.

#### 이래도 이해가 안돼서 비유로 알랴달라함 ㅋ

환자 차트 보관함에서 차트를 고치는 상황으로 비유한다.

- **"김철수 환자의 연락처를 바꿔줘"** → 가능
  - 이름으로 차트를 찾고 연락처만 고치면 되므로, 찾는 기준(이름)과 바꾸는 값(연락처)이 다르다.
  - (env의 `name: BACKEND_MAX_FAILS`로 항목을 찾고 `value`만 바꾸는 경우)
- **"김철수 환자의 이름을 이영희로 바꿔줘"를 이름으로 찾으라고 시키면** → 곤란
  - 보관함에 이영희 차트는 없으니 어느 차트를 고치라는 건지 알 수 없다.
  - (Ingress의 `host`를 `host` 값으로 찾으려는 경우)
- **해결: 이름 대신 위치로 지정** → "3번 서랍의 첫 번째 차트의 이름을 이영희로 바꿔"
  - (JSON Patch의 `/spec/rules/0/host`)

| | Strategic Merge | JSON Patch |
|---|---|---|
| 항목 찾는 법 | 이름(식별 값)으로 | 위치(경로)로 |
| 이름 자체를 바꿀 수 있나 | 불가 (찾는 기준이 사라지니까) | 가능 (위치로 찾으니까) |

- 리스트 8.21 (p.312) 패치 등록
  - Strategic Merge: `kustomize edit add patch --path patch-deployment.yaml`
  - JSON Patch: `kustomize edit add patch --kind Ingress --path patch-ingress.yaml --name echo` (패치 파일만으로 대상을 알 수 없어 `--kind`/`--name` 필요)
- 효과: 환경마다 YAML 복붙 없이 base 한 벌 + 바뀌는 부분만 관리


### 8.1.3 Kustomize에서 시크릿 다루기

문제: Secret 매니페스트는 값이 base64로 들어가는데, 이건 암호화가 아니라 인코딩일 뿐이라 이 파일을 Git에 올리면 비밀번호가 노출됨!

해결: 
MySQL 파드가 읽는 Secret 오브젝트가 클러스터에 있어야 한다. 원래는 그 Secret YAML을 직접 써야 했는데(비밀번호가 든 파일이라 Git 관리도 번거로움), secretGenerator는 비밀번호 메모 파일만 두면 build 때 Secret을 대신 만들어준다.

---
[🔎] 이해안돼서 또 물어봄;
- MySQL 파드는 비밀번호를 **클러스터 안의 Secret 오브젝트**에서 읽는다 (StatefulSet의 `secretName: mysql`). 내 PC의 파일은 파드가 읽을 수 없다.
- 그래서 Secret 오브젝트가 클러스터에 있어야 하는데, 직접 만들면 비밀번호가 base64로 든 Secret YAML이 생긴다.
  - base64는 암호화가 아니라 인코딩일 뿐이라 (`echo <값> | base64 -d`로 바로 복원됨) Git에 올리면 안 되고, 제외 관리가 번거롭다.
- **secretGenerator**: 비밀번호가 든 **텍스트 파일(재료)**만 두면 `kustomize build` 때 Secret YAML을 대신 만들어준다.
  - Git에는 Secret YAML이 아예 없고, 비밀번호 파일만 제외하면 된다.

#### 진행 순서
| 단계 | 내용 | 책 |
|---|---|---|
| ① | StatefulSet 작성, `kustomize create --autodetect` | 리스트 8.24~8.26 (p.314~316) |
| ② | 비밀번호 파일 생성 + `secretGenerator` 등록 | 리스트 8.27 (p.316) |
| ③ | Headless Service 추가, 공통 라벨 설정 | 리스트 8.28~8.29 (p.317) |
| ④ | `kustomize build`로 Secret 생성 확인 | 리스트 8.30 (p.318~319) |

#### 핵심 커맨드
```bash
# 재료: 비밀번호 파일 (-n: 줄바꿈이 비밀번호에 섞이지 않게)
echo -n "your-root-password" > mysql_root_password
echo -n "your-user-password" > mysql_user_password

# secretGenerator 등록
kustomize edit add secret mysql --from-file=root_password=./mysql_root_password
kustomize edit add secret mysql --from-file=user_password=./mysql_user_password

# Service를 관리 대상에 추가, 공통 라벨 설정
kustomize edit add resource service.yaml
kustomize edit set label "app.kubernetes.io/component:mysql"
```

#### 완성된 kustomization.yaml (리스트 8.29)
```yaml
resources:
- statefulset.yaml            # kustomize create --autodetect가 자동 등록 (①)
- service.yaml                # kustomize edit add resource service.yaml로 우리가 수동 등록 (③)
secretGenerator:              # kustomize edit add secret ... 으로 등록 (②)
- files:
  - root_password=./mysql_root_password   # 이 파일 내용을 root_password 항목으로
  - user_password=./mysql_user_password   # 이 파일 내용을 user_password 항목으로
  name: mysql                              # 생성될 Secret 이름 (build 때 해시가 붙음)
  type: Opaque
labels:                       # kustomize edit set label → edit fix 로 변환된 형태 (③)
- includeSelectors: true      # selector에도 라벨 주입
  pairs:
    app.kubernetes.io/component: mysql
```

#### build 결과에서 확인한 것 (리스트 8.30)
- **Secret이 생성됨**: `kind: Secret`, `data`의 값은 base64. 우리가 Secret YAML을 쓴 적은 없다.
- **이름에 해시 접미사**: `mysql-867k2ghhm5`
  - 내용에서 계산된 값이라 비밀번호가 바뀌면 해시도 바뀜 → StatefulSet이 가리키는 이름이 바뀜 → 파드 템플릿이 달라져 **파드가 자동 재시작**됨
- **참조 이름 자동 연결**: StatefulSet의 `secretName: mysql` → `secretName: mysql-867k2ghhm5`
- **라벨이 7곳에 주입됨**: Secret, Service(metadata + `spec.selector`), StatefulSet(metadata, `selector.matchLabels`, `template.metadata.labels`), volumeClaimTemplates
  - `selector`는 우리가 쓰지 않았는데 `includeSelectors: true`로 자동 생성됨

#### 참고: Headless Service (`clusterIP: None`)
- StatefulSet 파드는 `mysql-0`, `mysql-1`처럼 **순번이 고정된 이름**을 가진다 (Deployment 파드는 랜덤 이름).
- 일반 Service는 가상 IP 하나로 아무 파드에나 분배하지만, DB는 특정 파드를 지목해야 한다.
- Headless Service는 가상 IP 없이 DNS가 파드별 주소를 준다 (`mysql-0.mysql.default.svc.cluster.local`).
- StatefulSet의 `serviceName: "mysql"`이 이 Service를 가리키므로 같은 이름의 Service가 필요하다.

#### 정리
- Secret 오브젝트는 클러스터에 있어야 한다. secretGenerator는 그 YAML을 대신 만들어주는 것.
- Git 관리 대상: `statefulset.yaml`, `service.yaml`, `kustomization.yaml` / 제외 대상: `mysql_*_password` 파일


---

### 8.1.4 네트워크를 경유해서 매니페스트 생성하기

: 3줄요약

우와! github 주소를 resouces에 넣어서 남의 메니페스를 가져올 수 있구나!  
내 로컬에 메니페스트 파일이 하나도 없는데! 리소수가 출력되잖아!  
신기하군~  

---


## 8.2 Helm (p.322~)

### Helm이란
- **Helm**은 CNCF가 관리하는 쿠버네티스용 **패키지 관리 도구**이고, **Chart**는 쿠버네티스 리소스와 설정을 정리한 **패키지**다.
- Chart는 여러 매니페스트의 **템플릿**을 정리한 것이다. 대규모 앱이면 하위 Chart로 나누거나, 다른 Chart와 의존 관계를 가질 수 있다.
- 서드 파티 Chart 리포지터리에서 Chart를 받아 쓸 수 있다.
- 실무에서는 환경(로컬, 운영)에 상관없이 배포할 앱이라면 Chart로 패키징하고, `kubectl`이 아니라 Helm으로 배포와 업데이트를 하며, `kubectl`은 배포된 리소스를 일시적으로 수정할 때 쓴다.

### 개념 비유 (Java 개발자 기준)
| 개념 | 비유 |
|---|---|
| Helm | Maven/Gradle 같은 패키지 관리 도구 |
| Chart | 라이브러리 패키지 (jar) |
| Chart 리포지터리 | Maven Central 같은 저장소 |
| `helm install` | 의존성을 받아서 설치 |

### Kustomize와의 차이
| | Kustomize | Helm |
|---|---|---|
| 방식 | 완성된 YAML 위에 **패치**를 덮어씀 (템플릿 없음) | YAML **템플릿**에 values를 넣어서 생성 |
| 변경 지점 | 패치로 일부만 바꿈 | `values.yaml`의 변수만 바꿈 |
| 남이 만든 걸 쓸 때 | GitHub URL을 `resources`에 넣음 (8.1.4) | 리포지터리에서 Chart를 받아 `helm install` |
| 설치 이력 관리 | 없음 (`kubectl apply`가 담당) | **릴리스**로 관리 (업그레이드/롤백 가능) |

### 8.2 진행 순서
| 단계 | 내용 | 책 |
|---|---|---|
| ① | Helm 설치 (`asdf`) | 리스트 8.35 (p.323~324) |
| ② | Chart 리포지터리 추가, 검색 | 리스트 8.36~8.38 (p.324~325) |
| ③ | Chart 구조 파악 | 리스트 8.39 (p.326) |
| ④ | `bitnami/mysql` 설치, values 덮어쓰기 | 리스트 8.40~ (p.327~) |


---

#### Kustomize vs Helm: 같은 목표(MySQL + 비밀번호)를 푸는 방식

**목표**: StatefulSet, Service, Secret이 필요하고, 비밀번호는 내가 정한 값을 넣고 싶다.

#### Kustomize 방식 (8.1.3)
매니페스트를 **내가 직접 쓰고**, 비밀번호만 별도 파일로 뺀다.

```
내가 직접 씀:  statefulset.yaml, service.yaml
재료 파일:     mysql_root_password (비밀번호 글자)
설정:          kustomization.yaml (secretGenerator 등록)
         ↓ kustomize build
출력:          Secret(자동 생성) + StatefulSet + Service
```

#### Helm Chart 방식 (8.2)
매니페스트를 **Chart 제작자가 이미 만들어 두었고**, 나는 바꿀 값만 적는다.

```
남이 만들어 둠:  templates/ (statefulset, service, secret ... 템플릿들)
                 values.yaml (기본값)
내가 쓰는 것:    mysql.yaml (덮어쓸 값만: rootPassword 등)
         ↓ helm template / helm install
출력:           Secret + StatefulSet + Service + ... (전부 생성)
```

#### 파일 대응표
| 역할 | Kustomize | Helm Chart |
|---|---|---|
| 매니페스트 | 내가 쓴 `statefulset.yaml` (완성된 YAML) | `templates/*.yaml` (`{{ }}`가 든 **템플릿**) |
| 비밀번호 같은 값 | `mysql_root_password` 파일 | `values.yaml`의 `auth.rootPassword` |
| 내 값으로 바꾸기 | 파일 내용을 바꿈 | `mysql.yaml`로 덮어씀 |
| Secret 생성 | `secretGenerator` 설정 | `templates/secrets.yaml`이 처리 |
| 만드는 명령 | `kustomize build` | `helm template` / `helm install` |

#### Chart 구성 (p.326, 리스트 8.39)
```
chart-name/
├─ Chart.yaml        # Chart 정보
├─ charts/           # 의존 Chart
├─ templates/        # 매니페스트 템플릿 (+ NOTES.txt, _helpers.tpl)
└─ values.yaml       # 템플릿 변수의 기본값
```
- 기본값은 `values.yaml`, 바꾸려면 **커스텀 values 파일**로 덮어쓴다.

#### 커스텀 values로 덮어쓰기 (리스트 8.41)
```yaml
# mysql.yaml : 덮어쓸 항목만 적는다
auth:
  rootPassword: root_password
  database: ch08
  username: jpub
  password: jpub_password
```
- Kustomize의 `mysql_root_password` 파일에 대응 (비밀번호를 쓰는 곳)

#### helm install / upgrade / delete (p.328~331)

- 릴리스명은 네임스페이스 안에서 고유하며 업데이트, 삭제 때 필요

### 8.2.4 자체 Chart 생성하기

#### 템플릿 문법 3가지

| 문법 | 의미 | Kustomize에서 비슷한 것 |
|---|---|---|
| `{{ .Values.nginx.port }}` | `values.yaml`의 `nginx:` → `port:` 값을 여기에 넣는다 | 패치로 덮어쓸 그 값 자리 |
| `{{ include "echo.fullname" . }}` | `_helpers.tpl`에 만들어 둔 조각(이름 규칙)을 가져와 쓴다 | `namePrefix` 같은 공통 규칙 (자바로 따지면 유틸 메서드를 모아둔 클래스같은 거)|
| `{{- if ... }}` / `{{- with ... }}` | 조건에 맞을 때만 이 블록을 출력한다 | 없음 (패치는 조건 없이 적용) |

### 완성본 → 템플릿 변환 (값이 박혀 있던 자리를 변수로)
```yaml
# 완성본 (리스트 8.48)                          # 템플릿 (리스트 8.49)
name: echo                                      name: {{ include "echo.fullname" . }}
replicas: 1                                     replicas: {{ .Values.replicaCount }}
image: ghcr.io/.../simple-nginx-proxy:v0.1.0    image: "{{ .Values.nginx.image.repository }}:{{ .Values.nginx.image.tag }}"
value: "3"   # BACKEND_MAX_FAILS                value: "{{ .Values.nginx.maxFails }}"
```
- 값이 박혀 있던 자리가 `.Values.xxx`로 바뀌고, 그 값은 `values.yaml`에 정리한다 (리스트 8.50)
- 구조가 실제로 달라지는 부분은 `containers`뿐이다 

### 과도한 변수화는 피한다 (p.336, 리스트 8.51)
- 모형은 가능한 모든 항목을 변수화하지만, 과하면 조정 시간이 늘어난다.
- 외부 공개용 Chart가 아니면 `80`처럼 정해진 값은 직접 적어도 된다.
- 기본값은 템플릿에서 지정할 수도 있다.
```yaml
value: {{ .Values.nginx.port | default "80" }}
```

#### echo Chart 만들기 (p.332~346) — Kustomize(8.1)와 비교

같은 echo 앱을 Kustomize는 **base + overlay**로, Helm은 **Chart + values**로 만든다.

| 단계 | Kustomize (8.1.1~8.1.2) | Helm (8.2.4) | 책 |
|---|---|---|---|
| ① 뼈대 | 매니페스트 3개를 직접 작성 | `helm create echo`로 뼈대 생성 후 불필요한 템플릿 삭제 | 8.46~8.47 |
| ② 매니페스트 | 완성된 YAML | 뼈대의 템플릿을 echo에 맞게 수정 (Deployment만) | 8.48~8.49 |
| ③ 값 | YAML 안에 박혀 있음 | `values.yaml`에 모음 | 8.50 |
| ④ 환경별 값 | `overlays/dev` + 패치 파일 | 커스텀 values 파일 (`-f`) | 8.63 |
| ⑤ 미리보기 | `kustomize build .` | `helm template` (= **렌더링**) | 8.60 |
| ⑥ 배포 | `build \| kubectl apply -f -` | `helm package` → `helm install` | 8.62, 8.64 |

#### ① 뼈대 만들기 (리스트 8.46~8.47)
```bash
helm create echo
rm -rf hpa.yaml serviceaccount.yaml tests      # 최신 Helm은 httproute.yaml도 삭제
```
- Kustomize의 `kustomize create --autodetect`가 `kustomization.yaml`을 만들어 주듯, Helm은 **폴더 구조 전체**를 만들어 준다.
- 지우는 것: `hpa.yaml`(파드 수 자동 조절, 9장), `serviceaccount.yaml`, `tests/`(Chart 품질 테스트)

#### ②~③ templates와 values.yaml의 관계
- `templates/deployment.yaml` = **빈칸이 있는 양식**, `values.yaml` = **빈칸에 들어갈 값을 적어둔 목록** (서로 다른 파일)
- `{{ .Values.nginx.maxFails }}`는 `values.yaml`의 `nginx:` → `maxFails:` 값을 찾아 채운다. 들여쓰기 계층이 점(`.`)으로 이어지는 경로가 된다.
- 리스트 8.50의 "디플로이먼트에 대응하는 부분" = Deployment 템플릿의 빈칸들에 들어갈 값 (`nginx:`, `echo:` 블록)
- [🔎] `include "echo.fullname"`을 쓰는 이유 ???   
 이름이 **설치마다 달라져야** 하기 때문이다. 같은 Chart를 `dev`, `prod` 릴리스로 두 번 설치하면 `dev-echo`, `prod-echo`로 갈라져 이름 충돌이 없다. (한 번만 설치한다면 직접 적어도 된다)

#### Service, Ingress: 템플릿은 그대로, values만 수정 (리스트 8.52~8.58)
- 책: Service 템플릿(8.53)과 Ingress 템플릿(8.56)은 **수정이 필요 없다**. 뼈대 그대로 쓴다.
- Kustomize에서는 둘 다 YAML을 직접 썼지만, Helm은 `values.yaml`의 스위치와 값만 바꾼다.

| `values.yaml` 항목 | 뼈대 기본값 | echo용 | 이유 |
|---|---|---|---|
| `ingress.enabled` | `false` | `true` | 기본은 Ingress를 안 만듦 |
| `ingress.className` | `""` | `"nginx"` | 사용하는 Ingress Controller |
| `pathType` | `ImplementationSpecific` | `Prefix` | 책과 같이 변경 |
| `serviceAccount.create` | `true` | `false` | echo는 별도 계정 불필요 (이 설정이면 헬퍼가 `default` 계정을 반환) |

- Ingress의 `host: chart-example.local`은 뼈대의 **자리표시용 예시 값**이다. 기본값은 두고 설치할 때 덮어쓴다.

#### ⑤ 렌더링 확인 (리스트 8.60)
- `helm template echo ./echo`로 Service, Deployment, Ingress가 에러 없이 나오는지 설치 전에 확인한다 (Kustomize의 `build`에 해당).
- 값만 바꿔서 결과가 달라지는지는 `--set`으로 확인한다 (dev 오버레이가 하던 일).
```bash
helm template echo ./echo -s templates/deployment.yaml --set replicaCount=3 --set nginx.maxFails=10
```

#### ⑥ 패키징 (리스트 8.61~8.62)
```yaml
# Chart.yaml
apiVersion: v2
name: echo
description: Jpub echo application chart for Kubernetes
type: application
version: 0.0.0        # Chart 자체 버전 (패키징할 때마다 올림)
appVersion: "0.0.0"   # 앱 고유 버전
```
```bash
helm package echo     # echo-0.0.0.tgz 생성 (패키지 = Chart 폴더를 묶은 tgz)
tar tzf echo-0.0.0.tgz
```
- 파일명은 `name`과 `version`에서 나온다. 버전을 올려 다시 패키징하면 `echo-0.0.1.tgz`가 된다.
- Kustomize에는 이 패키징 단계가 없다 (폴더 그대로 Git으로 관리).

#### ④⑥ 커스텀 values로 덮어쓰고 설치 (리스트 8.63~8.64)
```yaml
# local-echo.yaml : 바꿀 항목만 적는다
ingress:
  hosts:
    - host: ch08-echo.jpub.local
      paths:
        - path: /
          pathType: Prefix
```
```bash
helm install echo ./echo-0.0.0.tgz -f local-echo.yaml      # 첫 echo = 릴리스명
helm ls
curl http://localhost -H 'Host: ch08-echo.jpub.local'      # Hello Container!!
```
- Chart 안의 `values.yaml`은 고치지 않고, **설치할 때** 값만 덮어쓴다. 그래야 같은 Chart를 환경별 values로 재사용한다.
- 환경을 늘리려면 `dev-echo.yaml`을 만들고 **릴리스명을 다르게** 설치한다.
```bash
helm install dev-echo ./echo-0.0.0.tgz -f dev-echo.yaml
```

#### 설치 후 확인
- `kubectl get deployment` → `READY 1/1`
- `curl`이 `Hello Container!!`을 반환하면 요청이 **Ingress → Service → 파드(nginx → echo)** 까지 통과한 것이다. 이로써 Deployment/Service/Ingress 템플릿, host 덮어쓰기, `className: nginx` 연결이 모두 확인된다.

#### 릴리스 관리 (Kustomize에 없는 부분, p.331)
```bash
helm ls                                      # 설치된 릴리스 목록
helm upgrade echo ./echo-0.0.1.tgz -f local-echo.yaml     # Chart/values 변경 반영
helm delete echo                             # 삭제 (릴리스명 지정)
```

#### 같은 echo, 두 방식 한눈에
| | Kustomize | Helm |
|---|---|---|
| 매니페스트 | 완성된 YAML을 직접 작성 | 템플릿(빈칸) + values(값) |
| 환경별 변경 | `overlays/`에 패치 파일 (Strategic Merge, JSON Patch) | 환경별 values 파일 (`-f`, `--set`) |
| 미리보기 | `kustomize build` | `helm template` (렌더링) |
| 배포 | `kubectl apply` | `helm install` (릴리스로 기록) |
| 패키지 | 없음 (폴더를 Git으로) | `helm package` → tgz |
| 강점 | 단순, 템플릿 없이 패치 | 조건 분기 등 세밀한 제어, 하나의 Chart로 여러 환경 대응 |

### [🤔] 지민이 질문 2. 그럼 환경별 세팅방법이 kustomize/helm인데, 버전관리 가능한 helm이 더 많이 사용되지 않을까? 만약 kustomize가 많이 사용된다면 어느 장점에서 사용하는지?

#### 1. 전제 보정
- "Kustomize는 버전관리가 안 된다"는 건 아니다. Kustomize도 **Git으로 버전 관리**한다
- Helm의 "버전관리"는 두 가지다: **Chart 버전**(패키지) + **릴리스 리비전**(`helm upgrade` 이력, `helm rollback`).
- 어느 쪽이 더 많이 쓰이는지는 확실한 통계가 없다. 실무에서는 **둘 다 널리 쓰이고, 용도가 다르며, 함께 쓰는 경우도 많다.**

#### 2. 각각 강한 영역

| | Helm | Kustomize |
|---|---|---|
| **핵심 장점** | 패키징·배포·릴리스 관리 | 있는 YAML을 환경별로 패치 |
| **방식** | 템플릿 + values | 순수 YAML + 패치 (템플릿 없음) |
| **강한 용도** | **남이 만든 소프트웨어 설치** (mysql, ingress-nginx 등 공개 Chart), 여러 곳에 배포하는 패키지 | **자사 앱의 환경별(dev/stg/prod) 차이 관리** |
| **롤백** | `helm rollback` 지원 | 기능 없음 (Git 되돌리기 + 재적용) |

#### 3. Kustomize를 쓰는 이유 (장점)

| 장점 | 설명 |
|---|---|
| **읽기 쉽다** | 템플릿 문법(`{{ }}`)이 없고 완성된 YAML 그대로라 리뷰·디버깅이 쉽다 |
| **kubectl 내장** | 별도 설치 없이 `kubectl apply -k`로 사용 가능 |
| **원본을 안 건드린다** | base는 그대로 두고 overlay로 차이만 표현한다 |
| **클러스터에 상태를 안 남긴다** | Helm 3는 릴리스 정보를 클러스터에 Secret으로 저장하지만, Kustomize는 빌드 결과를 적용할 뿐이다 |
| **남의 매니페스트를 수정하기 좋다** | 공개 YAML을 가져와 우리 환경에 맞게 패치할 수 있다 (Chart가 값으로 열어두지 않은 부분도 수정 가능) |
| **GitOps에 잘 맞는다** | Argo CD, Flux 모두 기본 지원 |

#### 4. Helm을 쓰는 이유 (장점)
- 공개 Chart 생태계가 커서 설치가 쉽다 (`helm install bitnami/mysql`).
- 값(values) 하나로 여러 리소스를 한 번에 조절할 수 있다.
- 릴리스 이력과 롤백, 의존성(subchart), 훅(설치 전후 작업)을 지원한다.

#### 5. 단점 (서로의 약점)
- Helm: 템플릿이 복잡해지면 읽기 어렵고, values.yaml이 비대해진다. 공백(indent) 실수가 잦다.
- Kustomize: 조건 분기·반복 같은 로직을 쓸 수 없다. 환경 간 차이가 크면 패치가 많아져 오히려 복잡해진다. 패키징·릴리스 관리 기능이 없다.

#### 6. 실무에서의 선택 기준

| 상황 | 선택 |
|---|---|
| 외부 오픈소스를 설치 (DB, ingress, 모니터링 등) | **Helm** |
| 여러 팀·클러스터에 배포할 패키지를 만들어 배포 | **Helm** |
| 둘의 장점을 다 쓰고 싶다 | **Helm Chart를 `helm template`으로 렌더링한 뒤 Kustomize로 환경별 패치** (Argo CD도 지원) |

#### 7. 한 줄 요약
**Helm = "패키지를 만들고 설치·관리"에 강하고, Kustomize = "이미 있는 YAML을 환경별로 단순하게 패치"에 강하다.**
용도가 달라서 경쟁이라기보다 **같이 쓰는 경우가 많다.**

# 마지막 8.2.5 Chart를 레지스트리에 등록하기는 제외하였습니다.
