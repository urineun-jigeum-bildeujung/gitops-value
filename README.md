# gitops-value

골라주개냥 서비스별 배포 값(values) 레포. `gitops` 레포의 `applications/appset.yaml`이 이 레포를
스캔해서 `values/{env}/services/*` 폴더 하나당 서비스 ArgoCD Application을 자동 생성한다.

## 구조

```
values/
  dev/
    services/
      _template/       새 서비스 추가 시 복사해서 시작 (자동 생성 대상에서 제외됨)
      api-gateway/values.yaml
      auth-service/values.yaml
      member-service/values.yaml
      notification-service/values.yaml
      order-service/values.yaml
      payment-service/values.yaml
      product-service/values.yaml
      review-service/values.yaml
```

각 `values.yaml`은 `gitops` 레포의 `charts/generic-service` 차트를 덮어쓰는 값이다. 차트 기본값에
없는 항목은 `gitops` 레포의 `charts/generic-service/values.yaml` 참고.

## 새 서비스 추가하는 법

1. `values/dev/services/_template/`을 `values/dev/services/<새-서비스명>/`으로 복사한다.
   `_`로 시작하는 폴더는 `appset.yaml`의 git generator에서 제외되므로, 새 폴더 이름은 절대
   `_`로 시작하면 안 된다 — 그러면 ArgoCD Application 자체가 안 생긴다.
2. `values.yaml`의 `image.repository`를 실제 ECR 경로로, `image.tag`를 실제 커밋 SHA로 바꾼다
   (최초 1회는 수동으로 채워야 함 — 이후엔 Jenkins가 자동 갱신, 아래 참고).
3. 네임스페이스는 폴더 이름과 동일해야 한다 — `gitops` 레포의 `platform/05-namespaces`가
   미리 만들어둔 이름과 다르면 배포가 안 된다.
4. mTLS/Kafka SASL/캐노리 등 옵션이 필요하면 `_template/values.yaml`에 주석 처리된 예시를 참고해서 켠다.
5. PR 머지 후 `applications/appset.yaml`이 자동으로 새 폴더를 스캔해서 `<env>-<서비스명>` 이름의
   Application을 만든다 — 별도로 `gitops` 레포를 건드릴 필요는 없다(appset.yaml 자체를 고칠
   필요가 있는 특수한 경우 제외).
6. `sync-wave: "10"`이라 platform 레이어(네임스페이스/RBAC/observability/CNPG/KEDA/ExternalSecrets)가
   전부 준비된 뒤에 뜬다 — DB/시크릿이 아직 없는 상태에서 새 서비스가 먼저 떠서
   CrashLoopBackOff 나는 걸 막기 위함.

## 이미지 태그는 어떻게 갱신되나

**사람이 직접 `image.tag`를 고치는 일은 거의 없다.** `sever` 레포의 Jenkinsfile이 `dev` 브랜치
빌드에 성공하면 전체 Git commit SHA를 태그로 ECR에 push하고, 같은 CI가 이 레포에 직접 커밋해서
`image.tag`를 갱신한다(PR 아님, 바로 push — `git log`에 보이는 `chore: deploy <서비스명> @
<SHA>` 커밋들이 이거다). Jenkins가 이 레포에 쓰려면 `gitops-value-push`라는 이름의 GitHub
Credential이 필요하고, 이건 `gitops` 레포의 `task bootstrap:jenkins-git-credentials`가 복구한다.

사람이 직접 `image.tag`를 고치는 경우는: (a) 신규 서비스 최초 배포 시 임시로 채울 때, (b) 특정
커밋으로 강제로 롤백/고정해야 할 때 정도다.

## PR 컨벤션

이 레포는 `gitops`와 달리 **이슈 템플릿이 없다** — 이슈 없이 바로 PR을 열어도 된다. 다만
관례상 지켜온 패턴:

- 브랜치명: `fix/<짧은-설명>` (예: `fix/auth-redis-mtls`), 이슈 번호는 안 붙임
- PR 제목: `fix: <한글 설명>` / `chore: deploy <서비스명> @ <SHA>`(CI 자동 커밋)
- 리소스/설정 값을 사람이 직접 고칠 땐 **반드시 PR로** — Jenkins의 자동 `chore: deploy` 커밋만
  예외적으로 `main`에 직접 push됨
- 새 브랜치는 항상 최신 `origin/main`에서 딴다 — 오래된 브랜치 위에서 새로 파면 이미지 태그
  커밋과 충돌 나기 쉽다(Jenkins가 수시로 `main`에 직접 커밋하기 때문에 다른 레포보다 더 자주
  뒤처짐)

## 값 수정 시 체크리스트

- YAML 문법 확인: `python3 -c "import yaml; yaml.safe_load(open('values/dev/services/<서비스명>/values.yaml'))"`
- `resources.limits.memory`를 바꿀 땐 `requests`는 그대로 두는 게 보통 맞다 — request는 노드
  스케줄링 예약량에 영향을 주고, limit은 런타임 상한에만 영향을 준다(둘을 같이 올리면 클러스터
  CPU/메모리 예약량이 타이트한 상황에서 스케줄링 실패로 이어질 수 있음).
- mTLS(`mtls.enabled`)를 새로 켤 땐, 그 서비스를 호출하는 다른 서비스의 URL(`*_SERVICE_URL`)도
  `https://...:8443`으로 같이 바꿔야 한다 — 한쪽만 켜져 있으면 연결 실패.
- `canary.steps`/`blueGreen`을 바꿀 땐 `gitops` 레포 README의 "Argo Rollouts 승격·재시작" 절차를
  참고 — `pause: {}`(duration 없음)가 있으면 수동 promote가 필요해진다.

## 주의

이 레포는 public이다. **실제 비밀값(DB 비밀번호, API 키 등)은 절대 넣지 않는다** — 그런 값은
`gitops` 레포의 `platform/91-external-secrets-config`가 정의하는 ExternalSecrets(AWS Secrets
Manager)를 통해 별도로 주입한다. 여기엔 이미지 태그, 리소스 크기, 환경변수 이름 같은 비민감
설정만 둔다. 시크릿을 참조할 땐 `valueFrom.secretKeyRef`로 이미 만들어진 K8s Secret 이름만
적는다(실제 값은 절대 안 적음).
