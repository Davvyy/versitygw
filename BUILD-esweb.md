# build/v1.8.0-esweb — 사내 반입용 빌드 브랜치

> 이 브랜치는 **상류에 올리기 위한 것이 아니다.** 폐쇄망 운영 노드에 넣을 바이너리를
> 만들기 위해 `v1.8.0` 에 상류 수정 3건만 얹은 것이다.

## 왜 자체 빌드인가

versitygw `v1.8.0` 은 Elasticsearch 의 `_snapshot/<repo>/_analyze` 를 통과하지 못한다.
결함은 **`abortWrite` 하나**로, ES 가 중단한 업로드를 게이트웨이가 **보이게 남기는** 것이다.

```
repository_verification_exception:
  upload of blob was aborted, but blob was erroneously found by at least one node
  blob analysis [... length=65536 ... abortWrite=true]
```

수정은 **상류 `main` 에 병합됐다**(2026-09-21). 그러나 아직 **어느 릴리스에도 안 실렸다**
(최신 `v1.8.0`, 2026-09-04). 그래서 릴리스가 나올 때까지만 쓰는 임시 빌드다.

🔑 **`v1.9.0` 에 실리면 이 브랜치와 산출물은 버린다.** 배포 경로·설정이 그대로라
릴리스 바이너리로 갈아끼우기만 하면 된다. 자체 빌드를 영구 자산으로 만들지 않는다.

## 무엇을 얹었나

`main` 전체(104 커밋 · 198 파일 · +11,666)가 **아니라** 관련 3건만 체리픽했다.
자체 빌드는 보안 심사를 우리가 떠안는다는 뜻이므로 **심사 면적을 최소로** 한다.

| | 파일 | 삽입/삭제 |
|---|---|---|
| `main` 전체 | 198 | +11,666 / −1,781 |
| **이 브랜치** | **13** | **+763 / −38** |

```
efd7bbd4  fix: close connections with unread chunked bodies
          <- c36696620ab0d1f3f72e9affe336257f32235c5d
2ed48408  fix: track how the request body ended before closing the connection
          <- d383e1f32f9c238e6bd8596a333433e5623085ef
41a18167  fix: reject a PutObject whose body ends before Content-Length
          <- d03ac3c29992ed8222042c8729f05821a66541da
```

셋 다 비머지 커밋이고 `-x` 로 체리픽해 **원본 SHA 가 커밋 메시지에 박혀 있다.**
충돌은 없었다.

## 재현

```bash
git fetch origin --tags
git checkout -B build/v1.8.0-esweb v1.8.0
git cherry-pick -x c3669662 d383e1f3 d03ac3c2

# 🔴 VERSION 을 반드시 만든다 (.gitignore 대상이라 브랜치에 없다)
printf 'v1.8.0+esweb.1\n' > VERSION

docker run --rm -v "$PWD:/src" -w /src -e GOFLAGS=-buildvcs=false golang:1.26 \
  sh -c 'git config --global --add safe.directory /src && make build'
```

### 🔴 `VERSION` 을 빠뜨리면 안 되는 이유

Makefile 은 `VERSION` 파일이 없으면 `git describe --abbrev=0 --tags` 로 **`v1.8.0` 을
그대로 찍는다.** 그러면 `--version` 출력이 **릴리스판과 구별되지 않고**, 운영에서
어느 바이너리가 도는지 알 수 없게 된다.

```
$ ./versitygw --version
Version  : v1.8.0+esweb.1     <- 이래야 한다
Build    : 41a18167
BuildTime: 2026-09-21_12:14:43AM
```

### ⚠ 사내망에서 `go mod download` 가 x509 로 막힌다

TLS 검사 장비(Somansa)가 재서명하기 때문이다. **검증을 끄지 말고** 사내 루트 CA 를
컨테이너 신뢰 저장소에 넣는다.

```bash
# Windows 에서 내보내기: Cert:\LocalMachine\Root 의 "CN=Somansa Root CA"
docker run ... -v "$PWD/ca:/ca:ro" ... \
  sh -c 'cp /ca/somansa-root.crt /usr/local/share/ca-certificates/ && update-ca-certificates && make build'
```

## 검증 (2026-09-21 · `golang:1.26` / go1.26.8 linux/amd64)

```
sha256  20d3eb12fa2fac7e062c03b25be60fe44d5ff4c661bceca2f4aeb7cfeba04602
크기    83,545,440 B
```

- **패치 핵심 테스트 전부 PASS** — `TestContentLengthReader`(truncated_body · EOF with
  final bytes · one byte per read · empty body but length announced · body longer than
  Content-Length) · `TestContentLengthReaderPassesThroughOtherErrors` ·
  `TestUnsignedChunkReaderContentLengthMismatchStopsAtDecodedLength` ·
  `TestDrainRequestBody_*` 8건
- **`go test ./...` 테스트 실패 0건** (16개 패키지 `ok`)
- ⚠ **빌드 실패 2건은 환경 문제다** — `s3api` · `rdma/rcroutes` 가
  `ld: cannot find -l:librcserver.a` 로 링크에 실패한다. RDMA 용 정적 라이브러리는
  별도 빌더 이미지(`build/vgwrdma-builder/`)로 만드는 것이라 평범한 golang 컨테이너에
  없다. **손대지 않은 `v1.8.0` 에서 똑같이 재현되는 것을 확인했다**(대조군). 우리가
  쓰는 posix 백엔드와 무관하고, 실제 산출물 `cmd/versitygw` 는 정상 빌드되며 그 패키지
  테스트도 `ok` 다.

### 🔴 단위 테스트 통과는 **소스**를 검증한 것이다

이 **바이너리**가 고쳐졌다는 증명이 아니다. 실제 노드에서 `_analyze` 로 확인한다.

```
POST /_snapshot/backup/_analyze?blob_count=100&max_blob_size=1mb&rare_action_probability=0.5&timeout=600s
```

**5회 돌린다.** 기본 확률 `0.02` 로는 abort 가 아예 안 나는 회차가 있어 **`v1.8.0` 조차
3회 중 1회는 통과했다** — 한 번 통과는 근거가 아니다.

## 배포

```bash
install -D -m 755 versitygw ~/vgw/bin/versitygw    # root 불필요
```

🔴 **롤백용으로 `v1.8.0` 릴리스 바이너리를 노드에 남겨 둘 것.**
`versitygw_v1.8.0_Linux_x86_64.tar.gz`
sha256 `2ba2c734d10d2c4e651d03182cb4b246656bc735a2f282db7b0b73fba6073467`
