---
title: HAProxy로 HTTPS와 LiveKit TURN/TLS를 443 포트에서 함께 사용하기
date: 2026-07-31 00:00:00 +0900
categories: [Project]
tags: [Infra, haproxy, livekit, turn, tls, webrtc, nginx, 말해봄]
---

## 문제 상황

LiveKit을 EC2에 배포한 뒤 signaling 연결까지는 정상적으로 이루어졌다.

브라우저 콘솔에서도 다음과 같은 로그를 확인할 수 있었다.

```text
signal connected
connected to LiveKit Server
```

하지만 실제 카메라와 마이크는 연결되지 않았다.

처음에는 WebSocket 프록시 문제라고 생각했지만, `/rtc` 요청은 Nginx를 거쳐 LiveKit까지 정상적으로 전달되고 있었다. 문제는 signaling이 아니라 WebRTC 미디어 연결이었다.

LiveKit 연결은 크게 두 경로로 나뉜다.

```text
signaling: 브라우저 → WSS → Nginx → LiveKit 7880
media:     브라우저 ↔ LiveKit TCP/UDP 포트
```

이번 환경에서는 LiveKit signaling은 연결됐지만, 외부 네트워크에서 미디어용 `7981/TCP`, `7982/UDP` 포트로 접근할 수 없었다.

```powershell
Test-NetConnection <서버 IP> -Port 7981
```

결과는 다음과 같았다.

```text
TcpTestSucceeded : False
```

서버 내부에서는 포트가 열려 있었기 때문에 애플리케이션 문제가 아니라 외부 방화벽 또는 네트워크 정책 문제로 판단했다.

## 왜 TURN/TLS가 필요한가?

WebRTC는 가능한 경우 UDP를 사용해 미디어를 직접 전달한다. 하지만 학교, 회사, 공공기관처럼 제한적인 네트워크에서는 UDP나 임의의 TCP 포트가 차단될 수 있다.

이때 사용할 수 있는 우회 경로가 TURN이다.

특히 TURN/TLS를 `443` 포트로 제공하면 일반 HTTPS와 유사한 TLS 트래픽으로 보이기 때문에 제한적인 네트워크에서도 연결될 가능성이 높아진다.

LiveKit에는 내장 TURN 서버가 있으므로 다음과 같이 활성화할 수 있다.

```yaml
turn:
  enabled: true
  domain: i15a605.p.ssafy.io
  tls_port: 443
  external_tls: true
```

여기서 `external_tls: true`는 LiveKit이 직접 TLS를 종료하지 않는다는 뜻이다.

외부의 로드 밸런서가 TLS를 종료하고, LiveKit에는 복호화된 TURN 트래픽을 전달한다. 클라이언트에게는 여전히 `turns:i15a605.p.ssafy.io:443` 주소가 제공된다.

## 443 포트 충돌

문제는 운영 웹 서버가 이미 `443` 포트를 사용하고 있다는 점이었다.

```text
https://i15a605.p.ssafy.io:443 → Nginx
```

TURN 전용 서브도메인을 만들 수 있다면 가장 간단하다.

```text
https://i15a605.p.ssafy.io:443      → Nginx
turns:turn.i15a605.p.ssafy.io:443   → LiveKit TURN
```

하지만 이번 환경에서는 DNS 레코드를 추가하기 어려웠다. 운영 웹의 `443`도 유지해야 했기 때문에 같은 도메인과 같은 포트에서 HTTPS와 TURN/TLS를 구분해야 했다.

두 요청은 같은 도메인을 사용하므로 SNI만으로 구분할 수 없다. 또한 Nginx의 HTTP reverse proxy는 TURN 바이너리 프로토콜을 처리할 수 없다.

그래서 공용 `443` 앞에 HAProxy를 두기로 했다.

## 전체 구조

최종 구조는 다음과 같다.

```text
외부 TCP 443
    ↓
HAProxy: TLS 종료 및 프로토콜 판별
    ├─ HTTPS ────────> Nginx 127.0.0.1:8443
    └─ TURN/STUN ────> LiveKit 127.0.0.1:7990
```

기존 dev 환경의 signaling 경로는 그대로 유지했다.

```text
https://i15a605.p.ssafy.io:8090/rtc
    → Nginx
    → LiveKit 127.0.0.1:7980
```

HAProxy는 `443`에서 TLS를 종료한 뒤 복호화된 데이터를 검사한다.

- ALPN이 `stun.turn`이면 TURN으로 전달
- ALPN이 없으면 STUN 헤더의 magic cookie를 검사
- 둘 다 아니면 일반 HTTPS로 전달

TURN over TLS의 표준 ALPN 이름은 `stun.turn`이다.

STUN 메시지 헤더의 4바이트부터는 고정된 magic cookie가 들어간다.

```text
0x2112A442
```

따라서 HAProxy에서 다음 조건으로 TURN 메시지를 식별할 수 있다.

```haproxy
acl turn_by_alpn ssl_fc_alpn -i stun.turn
acl turn_by_cookie req.payload(4,4) -m bin 2112A442
```

## LiveKit 설정

`docker-compose.dev.yml`에 LiveKit 서비스를 추가했다.

```yaml
services:
  livekit:
    image: livekit/livekit-server:v1.13.3
    container_name: dev-livekit
    restart: unless-stopped
    ports:
      - "127.0.0.1:7980:7880"
      - "127.0.0.1:7990:443"
      - "7981:7981"
      - "7982:7982/udp"
    environment:
      LIVEKIT_CONFIG: |
        port: 7880
        bind_addresses:
          - 0.0.0.0
        rtc:
          tcp_port: 7981
          udp_port: 7982
          node_ip: ${LIVEKIT_NODE_IP:?LIVEKIT_NODE_IP is required}
        room:
          enable_remote_unmute: true
        turn:
          enabled: true
          domain: i15a605.p.ssafy.io
          tls_port: 443
          external_tls: true
        keys:
          ${LIVEKIT_API_KEY:?LIVEKIT_API_KEY is required}: ${LIVEKIT_API_SECRET:?LIVEKIT_API_SECRET is required}
```

다음 포트 매핑이 핵심이다.

```yaml
- "127.0.0.1:7990:443"
```

외부에 `7990`을 공개하지 않고, 같은 서버의 HAProxy만 접근할 수 있도록 loopback 주소에 바인딩했다.

LiveKit 설정에 필요한 값은 배포 환경변수로 전달한다.

```dotenv
LIVEKIT_NODE_IP=<서버 공인 IP>
LIVEKIT_API_KEY=<API Key>
LIVEKIT_API_SECRET=<API Secret>
```

설정 문법은 컨테이너를 재생성하기 전에 먼저 검사한다.

```bash
sudo docker compose -f docker-compose.dev.yml -p dev config --quiet
```

그다음 LiveKit을 다시 생성한다.

```bash
sudo docker compose \
  -f docker-compose.dev.yml \
  -p dev \
  up -d --force-recreate livekit
```

로그와 포트를 확인한다.

```bash
sudo docker logs --since 2m dev-livekit 2>&1 \
  | grep -E 'Starting TURN|portTLS|externalTLS'

sudo ss -lntp | grep ':7990'
```

## Nginx의 443을 내부 포트로 이동

HAProxy가 외부 `443`을 사용해야 하므로 Nginx는 내부 포트로 이동시켰다.

기존 설정은 다음과 같았다.

```nginx
listen 443 ssl;
```

이를 loopback의 `8443`으로 변경했다.

```nginx
listen 127.0.0.1:8443 ssl;
```

인증서와 기존 location 설정은 그대로 유지했다.

```nginx
ssl_certificate /etc/letsencrypt/live/i15a605.p.ssafy.io/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/i15a605.p.ssafy.io/privkey.pem;
```

dev 환경이 사용하는 `8090` 포트도 변경하지 않았다.

설정을 적용하기 전에 반드시 문법을 검사한다.

```bash
sudo nginx -t
```

## HAProxy용 인증서 준비

HAProxy는 인증서 체인과 개인키를 하나의 PEM 파일로 사용한다.

```bash
sudo install -d -m 700 /etc/haproxy/certs
```

```bash
sudo sh -c '
umask 077
cat \
  /etc/letsencrypt/live/i15a605.p.ssafy.io/fullchain.pem \
  /etc/letsencrypt/live/i15a605.p.ssafy.io/privkey.pem \
  > /etc/haproxy/certs/i15a605.p.ssafy.io.pem
'
```

```bash
sudo chmod 600 /etc/haproxy/certs/i15a605.p.ssafy.io.pem
sudo chown root:root /etc/haproxy/certs/i15a605.p.ssafy.io.pem
```

인증서 정보를 확인한다.

```bash
sudo openssl x509 \
  -in /etc/haproxy/certs/i15a605.p.ssafy.io.pem \
  -noout -subject -issuer -dates
```

## HAProxy 설정

`/etc/haproxy/haproxy.cfg`는 다음과 같이 구성했다.

```haproxy
global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
    stats timeout 30s
    user haproxy
    group haproxy
    daemon
    maxconn 4096

    ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

defaults
    log global
    mode tcp
    option tcplog

    timeout connect 5s
    timeout client 1h
    timeout server 1h
    timeout tunnel 1h

frontend shared_tls_443
    bind :443 ssl \
        crt /etc/haproxy/certs/i15a605.p.ssafy.io.pem \
        alpn http/1.1,stun.turn

    tcp-request inspect-delay 5s

    acl turn_by_alpn ssl_fc_alpn -i stun.turn
    acl turn_by_cookie req.payload(4,4) -m bin 2112A442
    acl http_payload req.proto_http

    tcp-request content accept if turn_by_alpn
    tcp-request content accept if turn_by_cookie
    tcp-request content accept if http_payload

    use_backend livekit_turn if turn_by_alpn
    use_backend livekit_turn if turn_by_cookie

    default_backend nginx_https

backend nginx_https
    mode tcp
    server nginx 127.0.0.1:8443 ssl verify none check

backend livekit_turn
    mode tcp
    timeout server 1h
    server livekit 127.0.0.1:7990 check
```

공용 `443`에서는 `HTTP/2` 대신 `HTTP/1.1`만 협상하도록 했다.

```haproxy
alpn http/1.1,stun.turn
```

HAProxy가 TLS를 종료한 뒤 HTTP 데이터를 그대로 Nginx에 전달하기 때문에 프로토콜 불일치를 피하기 위한 선택이다. REST, STOMP, WebSocket은 HTTP/1.1로 동작할 수 있다.

Nginx 백엔드의 `verify none`은 loopback 구간에서만 사용하는 PoC 설정이다. 운영 환경에서는 내부 인증서 검증 정책을 별도로 정하는 것이 안전하다.

설정을 적용하기 전에 문법을 검사한다.

```bash
sudo haproxy -c -V -f /etc/haproxy/haproxy.cfg
sudo nginx -t
```

## 443 포트 소유권 전환

작업 순서가 중요하다.

먼저 Nginx가 기존 `443`을 놓고 `8443`을 사용하도록 재시작한다.

```bash
sudo systemctl restart nginx
```

포트를 확인한다.

```bash
sudo ss -lntp | grep -E ':(443|8443|8090)\b'
```

이 시점에는 다음 상태여야 한다.

```text
127.0.0.1:8443 → Nginx
0.0.0.0:8090   → Nginx
0.0.0.0:443    → 비어 있음
```

이후 HAProxy를 시작한다.

```bash
sudo systemctl restart haproxy
sudo systemctl enable haproxy
```

최종 포트 상태는 다음과 같다.

```text
0.0.0.0:443    → HAProxy
127.0.0.1:8443 → Nginx
127.0.0.1:7990 → LiveKit TURN
0.0.0.0:8090   → Nginx dev HTTPS
```

## HTTPS 경로 검증

먼저 기존 웹 서비스가 깨지지 않았는지 확인한다.

```bash
curl -skI --http1.1 \
  https://i15a605.p.ssafy.io/
```

```bash
curl -sk --http1.1 \
  -o /dev/null \
  -w '%{http_code}\n' \
  https://i15a605.p.ssafy.io/api-docs
```

ALPN도 확인할 수 있다.

```bash
openssl s_client \
  -connect i15a605.p.ssafy.io:443 \
  -servername i15a605.p.ssafy.io \
  -alpn http/1.1 \
  </dev/null 2>/dev/null \
  | grep 'ALPN protocol'
```

기대 결과는 다음과 같다.

```text
ALPN protocol: http/1.1
```

## TURN 경로 검증

TURN용 ALPN이 선택되는지 확인한다.

```bash
timeout 5 openssl s_client \
  -connect i15a605.p.ssafy.io:443 \
  -servername i15a605.p.ssafy.io \
  -alpn stun.turn \
  </dev/null 2>/dev/null \
  | grep 'ALPN protocol'
```

기대 결과는 다음과 같다.

```text
ALPN protocol: stun.turn
```

하지만 TLS 연결만 성공했다고 TURN 릴레이까지 성공한 것은 아니다.

브라우저에서 실제 LiveKit 방에 접속한 뒤 `chrome://webrtc-internals`의 selected candidate pair를 확인해야 한다.

```text
candidateType: relay
protocol: tcp
relayProtocol: tls
TURN URL: turns:i15a605.p.ssafy.io:443?transport=tcp
```

필요하면 서버에서 HAProxy와 LiveKit 사이의 패킷도 확인한다.

```bash
sudo tcpdump -ni any \
  'tcp port 443 or tcp port 7990'
```

관련 로그는 다음 명령으로 확인한다.

```bash
sudo journalctl -u haproxy --since '10 minutes ago' --no-pager
```

```bash
sudo docker logs --since 10m --timestamps dev-livekit 2>&1 \
  | tail -n 300
```

## 작업 중 겪은 Compose 오류

LiveKit 서비스를 처음 추가했을 때 Jenkins에서 다음 오류가 발생했다.

```text
(root) Additional property livekit is not allowed
```

원인은 `livekit`이 `services` 아래가 아니라 YAML 루트에 들어가 있었기 때문이다.

잘못된 구조는 다음과 같다.

```yaml
services:
  postgres:
    # ...

livekit:
  # 잘못된 위치
```

올바른 구조는 다음과 같다.

```yaml
services:
  postgres:
    # ...

  livekit:
    # services 아래에 위치
```

또한 다음과 같이 필수 환경변수를 지정하면 값이 없을 때 Compose interpolation 단계에서 바로 실패한다.

```yaml
node_ip: ${LIVEKIT_NODE_IP:?LIVEKIT_NODE_IP is required}
```

따라서 로컬 `.env`뿐 아니라 Jenkins가 실제로 읽는 배포 환경 파일에도 값을 넣어야 한다.

```dotenv
LIVEKIT_NODE_IP=<서버 공인 IP>
LIVEKIT_API_KEY=<API Key>
LIVEKIT_API_SECRET=<API Secret>
```

## 인증서 갱신 시 주의할 점

Certbot이 원본 인증서를 갱신해도 HAProxy용으로 합친 PEM 파일은 자동으로 갱신되지 않는다.

따라서 deploy hook에서 다음 작업을 수행해야 한다.

1. `fullchain.pem`과 `privkey.pem`을 다시 합친다.
2. HAProxy 설정 문법을 검사한다.
3. HAProxy와 Nginx를 reload한다.

이 과정이 빠지면 인증서 갱신 이후 HAProxy가 만료된 인증서를 계속 사용할 수 있다.

## 롤백 방법

HAProxy는 운영 HTTPS와 TURN이 모두 통과하는 지점이므로 문제가 생기면 두 기능이 동시에 영향을 받는다.

그래서 적용 전에 Nginx와 HAProxy 설정을 백업해 두는 것이 중요하다.

문제가 발생하면 HAProxy를 중지한다.

```bash
sudo systemctl stop haproxy
```

Nginx 설정을 기존 `443` 구성으로 복원한 뒤 다시 시작한다.

```bash
sudo nginx -t
sudo systemctl restart nginx
```

마지막으로 Nginx가 다시 `443`을 사용하고 있는지 확인한다.

```bash
sudo ss -lntp | grep ':443'
curl -skI https://i15a605.p.ssafy.io/
```

이 롤백은 프록시 경로만 되돌리므로 DB나 Redis 데이터에는 영향을 주지 않는다.

## 정리

이번 문제에서 가장 중요했던 점은 signaling 연결과 WebRTC 미디어 연결을 분리해서 보는 것이었다.

```text
WSS 연결 성공 ≠ 미디어 연결 성공
```

`/rtc` WebSocket이 정상이어도 미디어 포트가 차단되면 카메라와 마이크는 연결되지 않는다. 직접 TCP와 UDP 경로를 열 수 없다면 TURN/TLS가 대안이 될 수 있다.

이번 구성에서는 다음 역할로 나누었다.

- HAProxy: 공용 `443`의 TLS 종료와 HTTPS·TURN 프로토콜 분기
- Nginx: 기존 웹, API, WebSocket reverse proxy
- LiveKit: signaling, SFU, 내장 TURN 서버

별도 TURN 도메인을 사용하는 것이 더 단순하고 일반적인 구조다. 하지만 DNS와 방화벽을 직접 변경할 수 없고 기존 HTTPS `443`도 유지해야 하는 조건에서는 HAProxy를 이용한 프로토콜 분기가 하나의 대안이 될 수 있다.

마지막으로 `openssl`의 ALPN 결과만으로 완료라고 판단하면 안 된다. 실제 브라우저에서 `relay` candidate가 선택되고 영상과 음성이 양방향으로 전달되는지까지 확인해야 한다.

## 참고 자료

- [LiveKit 배포 문서](https://docs.livekit.io/transport/self-hosting/deployment/)
- [LiveKit v1.13.3 설정 예시](https://github.com/livekit/livekit/blob/v1.13.3/config-sample.yaml)
- [RFC 7443 - STUN/TURN ALPN](https://www.rfc-editor.org/rfc/rfc7443.html)
- [RFC 8489 - STUN](https://www.rfc-editor.org/rfc/rfc8489.html)
- [HAProxy Configuration Manual](https://docs.haproxy.org/2.8/configuration.html)
