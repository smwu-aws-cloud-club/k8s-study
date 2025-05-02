
# NGINX 인그레스(Ingress) 컨트롤러로 Minikube에서 인그레스 설정하기

---

## 시작하기 전에
- Kubernetes 1.19 이상 필요
- `kubectl` 명령줄 도구 설치 필요
- 노드가 2개 이상인 클러스터 추천

클러스터가 없다면 아래 플랫폼 중 하나를 사용할 수 있음:
- Killercoda
- KodeKloud
- Play with Kubernetes

---

## Minikube 클러스터 생성
Minikube가 설치되어 있다면 다음 명령으로 클러스터를 생성한다:

```bash
minikube start
```

---

## 인그레스 컨트롤러 활성화
```bash
minikube addons enable ingress
```

### 컨트롤러 확인
```bash
kubectl get pods -n ingress-nginx
```

정상적인 결과 예시:
```
ingress-nginx-controller-xxxxx   1/1   Running   0   XXm
```

---

## Hello World 앱 배포
```bash
kubectl create deployment web --image=gcr.io/google-samples/hello-app:1.0
kubectl expose deployment web --type=NodePort --port=8080
```

### 서비스 확인
```bash
kubectl get service web
```

### 노드포트로 앱 접속
```bash
minikube service web --url
```

출력 예시:
```
http://172.17.0.15:31637
```

---

## 인그레스 리소스 생성
example-ingress.yaml:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: hello-world.info
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web
            port:
              number: 8080
```

적용:
```bash
kubectl apply -f https://k8s.io/examples/service/networking/example-ingress.yaml
kubectl get ingress
```

### /etc/hosts 설정
```
172.17.0.15 hello-world.info
```

### 테스트
```bash
curl hello-world.info
```

---

## 두 번째 디플로이먼트 추가
```bash
kubectl create deployment web2 --image=gcr.io/google-samples/hello-app:2.0
kubectl expose deployment web2 --port=8080 --type=NodePort
```

### 인그레스 수정
example-ingress.yaml에 아래 추가:
```yaml
      - path: /v2
        pathType: Prefix
        backend:
          service:
            name: web2
            port:
              number: 8080
```

적용:
```bash
kubectl apply -f example-ingress.yaml
```

---

## 최종 테스트
```bash
curl hello-world.info
# Version: 1.0.0

curl hello-world.info/v2
# Version: 2.0.0
```

브라우저에서도 `hello-world.info` 및 `/v2` 경로 접속 가능

---

**참고:** Minikube 환경에서는 `minikube ip`를 통해 실제 IP 주소 확인 필요
