# Kubernetes ConfigMap

## 개요

- **ConfigMap**은 기밀이 아닌 구성 데이터를 키-값 쌍으로 저장하는 API 오브젝트이다.
- 파드에서 환경 변수, 명령 인수, 구성 파일 등으로 사용할 수 있다.
- 컨테이너 이미지와 환경 구성을 분리해 **애플리케이션 이식성**을 높일 수 있다.
- 기밀 데이터는 Secret 사용을 권장한다.

---

## 사용 이유

- 코드와 환경 설정을 분리하여 운영 환경 및 개발 환경에서 재사용성을 확보
- 1MiB를 초과하지 않는 작은 설정 데이터를 저장

---

## 구성 형식

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-demo
data:
  player_initial_lives: "3"
  ui_properties_file_name: "user-interface.properties"
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5    
  user-interface.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true    
```

- `data`는 일반 텍스트
- `binaryData`는 base64 인코딩된 이진 데이터

---

## 파드에서 사용하는 방법

1. **환경 변수로 주입**
2. **명령어 및 인수에 사용**
3. **볼륨으로 마운트**
4. **API에서 직접 조회**

```yaml
env:
- name: PLAYER_INITIAL_LIVES
  valueFrom:
    configMapKeyRef:
      name: game-demo
      key: player_initial_lives
```

```yaml
volumes:
- name: config
  configMap:
    name: game-demo
    items:
    - key: "game.properties"
      path: "game.properties"
```

---

## 자동 업데이트

- 컨피그맵은 볼륨 마운트일 경우 kubelet 주기 동기화에 따라 자동 업데이트됨
- 환경 변수로 사용한 경우에는 자동 업데이트되지 않음 → 파드 재시작 필요
- `subPath` 마운트 시 자동 업데이트 없음

---

## 변경할 수 없는 ConfigMap (Immutable)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
immutable: true
data:
  key1: value1
```

- **v1.21 이상**
- 수정 불가능, 삭제 후 재생성 필요
- **클러스터 성능 향상** 및 **우발적 변경 방지**