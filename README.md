# Naver Search MCP (K8s + SSE Proxy)
**Author:** Jung Wookjin

네이버 검색/DataLab API를 **MCP 서버**로 제공하고, Kubernetes에서 **SSE(HTTP) 엔드포인트**로 노출하는 가이드입니다.  
백엔드는 `transport="sse"`로 `/sse`에 연결합니다.

## 0) Clone

```bash
git clone https://github.com/wookja-0/naver-search-mcp.git
cd naver-search-mcp
```

## 1) 환경 변수

```bash
# 컨테이너 레지스트리 (예: ghcr.io/org, harbor.example.com/team 등)
export REGISTRY="__YOUR_REGISTRY__/__YOUR_PROJECT__"

# 이미지 태그 (원하면 변경)
export IMAGE_BASE="$REGISTRY/mcp-server:v1.0"
export IMAGE_SSE="$REGISTRY/mcp-server:sse-proxy"
```

## 2) Docker 이미지 빌드 & 푸시

### 2-1. MCP 서버 베이스 이미지 (v1.0)

```bash
docker build -t "$IMAGE_BASE" . --network host
docker push "$IMAGE_BASE"
```

### 2-2. SSE 프록시 이미지 (stdio ↔ SSE 변환 포함)

```bash
docker build -f Dockerfile.sse -t "$IMAGE_SSE" . --network host
docker push "$IMAGE_SSE"
```

> `Dockerfile.sse`는 컨테이너에 `mcp-proxy`를 설치하고 `docker-entrypoint.sh`로 **/sse**, **/status**를 노출합니다.

## 3) Kubernetes 배포

> 아래 예시는 네임스페이스를 `dmp-poc`로 가정합니다. 환경에 맞게 변경하세요.

### 3-1. (선택) 로그 PVC

```bash
kubectl -n dmp-poc apply -f manifest/pvc.yaml
```

### 3-2. 서버용 Secret (Naver Developers 키)

```bash
kubectl -n dmp-poc create secret generic naver-api-keys   --from-literal=NAVER_CLIENT_ID='<your-client-id>'   --from-literal=NAVER_CLIENT_SECRET='<your-client-secret>'
```

### 3-3. Deployment / Service

```bash
kubectl -n dmp-poc apply -f manifest/deployment.yaml
kubectl -n dmp-poc apply -f manifest/service.yaml

# 배포 후 실제 이미지로 교체(컨테이너 이름: sse-proxy)
kubectl -n dmp-poc set image deploy/mcp-server sse-proxy="$IMAGE_SSE"

kubectl -n dmp-poc rollout status deploy/mcp-server
```

> 매니페스트의 리소스 이름이 `mcp-server`인지 확인하세요. (Deployment/Service 메타데이터 name)

## 4) 동작 확인

```bash
# 상태 확인
kubectl -n dmp-poc port-forward svc/mcp-server 8080:8080
curl http://127.0.0.1:8080/status     # JSON 응답이면 OK
```

로그:
```bash
# 프록시 표준로그
kubectl -n dmp-poc logs deploy/mcp-server -f

# 앱 로그(PVC 사용 시)
# 예) NFS 경로에 app.log 생성/증가
```

## 5) 백엔드(클라이언트) 연결 예시

```python
from langchain_mcp_adapters.client import MultiServerMCPClient

mcp_client = MultiServerMCPClient(
    {
        "naver-search-mcp": {
            "transport": "sse",
            "url": "http://mcp-server.dmp-poc.svc.cluster.local:8080/sse"
        }
    }
)
```

> Smithery 전용 경로(`/@org/server/mcp?...`)는 사용하지 않습니다.  
> 서버 컨테이너에는 `NAVER_CLIENT_ID/SECRET`(K8s Secret)이 주입되어 네이버 API를 호출합니다.

## 6) 트러블슈팅(짧게)

- **/status 200 아님** → `kubectl logs`로 프록시/앱 로그 확인  
- **연결됐는데 결과 없음** → `transport="sse"` & URL `/sse` 확인  
- **로그 파일 안 쌓임** → PVC 권한(fsGroup)·마운트 경로 점검

---

**License:** MIT  

