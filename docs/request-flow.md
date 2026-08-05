# 요청 처리 플로우

호스트네임 종류별로 요청이 어떻게 흘러가는지 시퀀스 다이어그램으로 정리.

## 1. API 요청 (`server.storkspear.cloud`)

```mermaid
sequenceDiagram
  autonumber
  participant U as 사용자
  participant CF as Cloudflare 엣지
  participant CT as cloudflared (Mac mini)
  participant KP as kamal-proxy
  participant API as server-factory-web

  U->>CF: GET https://server.storkspear.cloud/api/...
  Note over CF: TLS 종단, WAF, 라우팅
  CF->>CT: 터널로 forward (Host 헤더 보존)
  CT->>CT: ingress 매칭<br/>→ http://localhost:80
  CT->>KP: HTTP forward
  KP->>KP: server_name 매칭<br/>→ server-factory-web
  KP->>API: localhost:8080
  API-->>KP: JSON 응답
  KP-->>CT: 응답 전달
  CT-->>CF: 터널 통해 응답
  CF-->>U: HTTPS 응답
```

핵심: Cloudflare에서 TLS 종단 후 평문 HTTP로 터널을 통과 → 내부에서도 평문 HTTP로 컨테이너 간 통신. 외부에는 HTTPS로 보임.

---

## 2. 정적 사이트 요청 (`portfolio.seoseji.site`)

```mermaid
sequenceDiagram
  autonumber
  participant U as 사용자
  participant CF as Cloudflare
  participant CT as cloudflared
  participant N as homepage-nginx
  participant FS as ~/sites/portfolio/

  U->>CF: GET https://portfolio.seoseji.site/
  CF->>CT: 터널 forward (Host: portfolio.seoseji.site)
  CT->>N: localhost:8088
  N->>N: server_name 매칭<br/>→ root /sites/portfolio
  N->>FS: read index.html
  FS-->>N: HTML 반환
  N-->>CT: 200 OK + HTML
  CT-->>CF: 응답
  CF-->>U: HTML (Cloudflare 캐시 가능)
```

호스트별로 다른 `root` 디렉토리를 가지므로, 같은 nginx 컨테이너가 사이트마다 다른 콘텐츠를 서빙.

---

## 3. www → apex 리다이렉트 (`www.storkspear.co.kr`)

```mermaid
sequenceDiagram
  autonumber
  participant U as 사용자
  participant CF as Cloudflare
  participant CT as cloudflared
  participant N as homepage-nginx

  U->>CF: GET https://www.storkspear.co.kr/
  CF->>CT: 터널 forward
  CT->>N: localhost:8088
  N->>N: server_name www.* 매칭<br/>→ return 301
  N-->>CT: 301 Location: https://storkspear.co.kr/
  CT-->>CF: 301
  CF-->>U: 301
  U->>CF: GET https://storkspear.co.kr/ (자동)
  Note over CF,N: 위 "정적 사이트" 플로우와 동일
  CF-->>U: 200 OK + HTML
```

브라우저 두 번 요청. 사용자 경험상 자연스럽고 SEO 일관성도 확보됨.

---

## 4. 배포 플로우 — 정적 사이트 (`portfolio.seoseji.site`)

```mermaid
sequenceDiagram
  autonumber
  participant Dev as 개발자 노트북
  participant GH as GitHub (storkspear/design-portfolio)
  participant TS as Tailscale
  participant MM as Mac mini
  participant N as nginx 컨테이너

  Dev->>Dev: 코드 수정
  Dev->>GH: git push origin main
  Dev->>TS: ssh <mac-mini-user>@<tailscale-ip>
  TS->>MM: SSH 세션
  MM->>GH: cd ~/sites/portfolio && git pull
  GH-->>MM: 새 파일들
  Note over N: nginx는 mount된 파일을<br/>요청 시마다 읽음 → 즉시 반영
  Note over MM: GitHub Actions 빌드 불필요<br/>Pages 배포 불필요
```

이 플로우 덕분에 GitHub Actions 사용량과 무관하게 무한 배포 가능.

---

## 5. 배포 플로우 — 백엔드 (`server.storkspear.cloud`)

```mermaid
sequenceDiagram
  autonumber
  participant Dev as 개발자
  participant GH as GitHub Actions
  participant GHCR as GHCR (이미지 레지스트리)
  participant MM as Mac mini
  participant KP as kamal-proxy

  Dev->>GH: git push (백엔드 레포)
  GH->>GH: Docker 이미지 빌드
  GH->>GHCR: ghcr.io/storkspear/server-factory:<sha> push
  Dev->>MM: kamal deploy (또는 자동화)
  MM->>GHCR: 새 이미지 pull
  MM->>MM: 새 컨테이너 spin up
  MM->>KP: 새 타겟 등록
  KP->>KP: health check
  KP->>KP: 트래픽을 새 컨테이너로 swap
  KP->>KP: 옛 컨테이너 graceful shutdown
  Note over KP: 사용자 입장에서 무중단
```

Kamal의 핵심: 옛 컨테이너에 요청 처리 끝나기를 기다린 뒤 종료 → 1초도 안 끊김.

---

## 6. www DNS 추가 시 SSL 발급 흐름 (운영 팁)

```mermaid
sequenceDiagram
  autonumber
  participant Op as 운영자
  participant CFAPI as CF API
  participant CFNS as CF Auth NS
  participant SSL as Universal SSL
  participant U as 사용자

  Op->>CFAPI: POST /dns_records (CNAME, proxied=true)
  CFAPI-->>Op: 201 Created (즉시)
  CFAPI->>SSL: 인증서 요청 트리거
  Note over SSL: 5~15분 소요
  Note over CFNS: SSL 발급 완료 전까지<br/>외부 dig는 NODATA
  SSL->>CFNS: 발급 완료 → DNS 활성화
  U->>CFNS: dig 호스트
  CFNS-->>U: A 레코드 (CF proxy IP)
  U->>U: HTTPS 요청 가능
```

새로 추가한 proxied 호스트가 외부에 안 보이면 90% 이 발급 대기 때문.
