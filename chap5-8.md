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

---

---

---

---
# Chap 7. 쿠버네티스 활용

---

---

---

---

---
# Chap 8. 쿠버네티스 애플리케이션 패키징