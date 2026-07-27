# 워크플로 설정 안내

이 저장소의 GitHub Actions는 리소스팩 산출물(`pack.zip`)만 공개 릴리스로 배포합니다.
최적화 설정과 CI 스크립트, 원본 팩은 비공개 저장소에 있으며 워크플로 실행 중에만 내려받습니다.

## 실행 권한

두 워크플로 모두 `guard` 잡에서 실행자를 검증합니다.

- 수동 실행(`workflow_dispatch`): 저장소 협력자 중 `admin` / `maintain` / `write` 권한자만 통과합니다. 권한 조회 실패도 거부로 처리합니다.
- `repository_dispatch`: 호출 자체에 쓰기 권한 토큰이 필요하므로 통과합니다.
- `workflow_run`: 내부 트리거이므로 통과합니다.

`push`, `pull_request`, `pull_request_target` 트리거는 사용하지 않습니다. 포크에서 워크플로를 실행하거나 시크릿에 접근할 수 없습니다.

## 필수 시크릿

`Settings → Secrets and variables → Actions → Secrets`

| 이름 | 필수 | 용도 |
|---|---|---|
| `PRIVATE_REPO_TOKEN` | 예 | 비공개 저장소 clone 및 원본 ZIP 다운로드 (`contents: read` 범위) |
| `PACKSQUASH_DISCORD_WEBHOOK_URL` | 아니요 | 진행 상황 알림. 미설정 시 알림을 건너뜁니다 |
| `SSH_HOST` | 아니요 | 서버 반영용 SSH 호스트 |
| `SSH_PORT` | 아니요 | SSH 포트 (기본 22) |
| `SSH_USERNAME` | 아니요 | SSH 사용자명 |
| `SSH_PRIVATE_KEY` | 아니요 | SSH 개인키 (ed25519) |
| `SSH_KNOWN_HOSTS` | 아니요 | 호스트 키 고정 검증용. 미설정 시 첫 연결을 `accept-new`로 수용합니다 |
| `RCON_PASSWORD` | 아니요 | `COMMAND_TRANSPORT=rcon`일 때 필요 |

SSH 시크릿이 없으면 서버 반영 워크플로는 실패하지 않고 건너뜁니다.

## 변수

`Settings → Secrets and variables → Actions → Variables`

| 이름 | 기본값 | 용도 |
|---|---|---|
| `SOURCE_REPO` | `kooyhee/ResourcePack` | 원본 ZIP과 설정이 있는 비공개 저장소 |
| `PUBLIC_RELEASE_TAG` | `latest-resource-pack` | 공개 릴리스 고정 태그 |
| `COMMAND_TRANSPORT` | `panel` | 서버 명령 전송 방식 (`panel` 또는 `rcon`) |
| `PANEL_REMOTE_PORT` | `8765` | 패널 원격 포트 |
| `PANEL_COMMAND_DELAY_SECONDS` | `10` | 명령 사이 대기 시간(초) |

## 비공개 저장소에 있어야 하는 파일

- `packsquash.toml` (저장소 루트)
- `automation/ci/prepare_pack.py`
- `automation/ci/render_options.py`
- `automation/ci/normalize_png.py`
- `automation/ci/notify_discord.py`

## 원본 ZIP 업로드

```bash
gh release upload source-latest ./generated.zip --clobber --repo kooyhee/ResourcePack
```

## 자동 트리거 예시

```bash
curl -X POST https://api.github.com/repos/kooyhee/ResourcePack-Public/dispatches \
  -H "Accept: application/vnd.github+json" \
  -H "Authorization: Bearer <PAT>" \
  -H "X-GitHub-Api-Version: 2022-11-28" \
  -d '{
    "event_type": "source-pack-updated",
    "client_payload": {
      "source_tag": "source-latest",
      "source_asset": "generated.zip",
      "publish": true,
      "force": false
    }
  }'
```

`<PAT>`에는 이 저장소에 대한 `contents: write` 권한이 필요합니다.

## 첫 실행 권장 절차

Actions 탭에서 `리소스팩 최적화`를 수동 실행하면서 `force`를 `true`, `publish`를 `false`로 두면
릴리스에 영향을 주지 않고 최적화까지만 검증할 수 있습니다.
