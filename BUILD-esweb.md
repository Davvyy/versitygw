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
printf 'v1.8.0+esweb.2\n' > VERSION

# 🔴 CGO_ENABLED=0 을 반드시 준다 - 아래 <릴리스와 같게 만드는 것> 참고
docker run --rm -v "$PWD:/src" -w /src \
  -e GOFLAGS=-buildvcs=false -e CGO_ENABLED=0 golang:1.26 \
  sh -c 'git config --global --add safe.directory /src && make build'
```

### 🔴 `VERSION` 을 빠뜨리면 안 되는 이유

Makefile 은 `VERSION` 파일이 없으면 `git describe --abbrev=0 --tags` 로 **`v1.8.0` 을
그대로 찍는다.** 그러면 `--version` 출력이 **릴리스판과 구별되지 않고**, 운영에서
어느 바이너리가 도는지 알 수 없게 된다.

```
$ ./versitygw --version
Version  : v1.8.0+esweb.2     <- 이래야 한다
Build    : 8d98aee6
BuildTime: 2026-09-21_12:45:50AM
```

### 🔴 `CGO_ENABLED=0` — 릴리스와 같게 만드는 유일한 조건

**`make build` 만으로는 릴리스와 같은 바이너리가 안 나온다.** 릴리스는 Makefile 이
아니라 `.goreleaser.yaml` 로 만들고, 거기에만 이 설정이 있다. 주석이 이유를 적어 두었다.

```yaml
env:
  # disable cgo to fix glibc issues: https://github.com/golang/go/issues/58550
  - CGO_ENABLED=0
```

golang 이미지에는 gcc 가 있어 **기본값이 `CGO_ENABLED=1`** 이다. 그대로 빌드하면
**glibc 에 동적 링크된** 바이너리가 나온다.

| | 릴리스 `v1.8.0` | CGO 켠 빌드 | **CGO 끈 빌드** |
|---|---|---|---|
| 크기 | 57.2 MiB | 79.7 MiB | **58.1 MiB** |
| 동적 로더 참조 | 없음 | 🔴 있음 | **없음** |
| `GLIBC_` 버전 문자열 | 0 | 🔴 **63** | **0** |

🔴 **드러나는 시점이 나쁘다.** 컨테이너 안에서는 잘 돌고 단위 테스트도 다 통과하고,
반입해서 운영 노드에서 **처음 실행할 때** 터진다 — 그것도 우리 패치 탓처럼 보인다.

```
./versitygw: /lib64/libc.so.6: version `GLIBC_2.38' not found
```

빌드 노드(Debian 계열)의 glibc 가 운영 노드(RHEL 계열)보다 높으면 그렇게 된다.

✅ **덤으로 테스트도 좋아진다.** CGO 를 켜면 `s3api` 와 `rdma/rcroutes` 가
`ld: cannot find -l:librcserver.a` 로 링크에 실패하는데(RDMA 용 정적 라이브러리는 별도
빌더 이미지로 만든다), 끄면 `s3api` 는 정상 통과하고 rdma 쪽은 아예 대상에서 빠진다 —
cgo 경로라 맞는 동작이다.

⚠ **이 문서의 첫 판(`+esweb.1`)이 정확히 이것을 밟았다.** "상류 Makefile 을 그대로
쓰니 릴리스와 같은 방식" 이라고 적었는데 틀렸다. 그 산출물은 폐기했다.

### ⚠ 사내망에서 `go mod download` 가 x509 로 막힌다

TLS 검사 장비(Somansa)가 재서명하기 때문이다. **검증을 끄지 말고** 사내 루트 CA 를
컨테이너 신뢰 저장소에 넣는다.

```bash
# Windows 에서 내보내기: Cert:\LocalMachine\Root 의 "CN=Somansa Root CA"
docker run ... -v "$PWD/ca:/ca:ro" ... \
  sh -c 'cp /ca/somansa-root.crt /usr/local/share/ca-certificates/ && update-ca-certificates && make build'
```

## 검증 (2026-09-21 · `golang:1.26` / go1.26.8 linux/amd64 / `CGO_ENABLED=0`)

```
sha256  4b62c5fade3fca6afb3a0725a3b9b93b79ab791914baabfe32b9888c98231956
크기    60,916,254 B  (58.1 MiB)
```

- **패치 핵심 테스트 전부 PASS** — `TestContentLengthReader`(truncated_body · EOF with
  final bytes · one byte per read · empty body but length announced · body longer than
  Content-Length) · `TestContentLengthReaderPassesThroughOtherErrors` ·
  `TestUnsignedChunkReaderContentLengthMismatchStopsAtDecodedLength` ·
  `TestDrainRequestBody_*` 8건
- `TestUnsignedChunkReader*` 6건
- **`go test ./...` — 29개 패키지 `ok`, 테스트 실패 0건, 빌드 실패 0건**

> 참고로 **CGO 를 켜고 돌렸을 때는** `s3api` 와 `rdma/rcroutes` 가
> `ld: cannot find -l:librcserver.a` 로 링크에 실패했다. 손대지 않은 `v1.8.0` 에서도
> 똑같이 재현되는 환경 문제였고, `CGO_ENABLED=0` 으로 바꾸면서 사라졌다.

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
