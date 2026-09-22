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
없는 항목은 `charts/generic-service/values.yaml` 참고.

## 주의

이 레포는 public이다. **실제 비밀값(DB 비밀번호, API 키 등)은 절대 넣지 않는다** — 그런 값은
ExternalSecrets(AWS Secrets Manager)를 통해 별도로 주입한다. 여기엔 이미지 태그, 리소스 크기 같은
비민감 설정만 둔다.

`api-gateway`는 최초 CI 이미지가 준비될 때까지 `gitops` ApplicationSet에서 명시적으로 제외한다.
Jenkins가 실제 전체 Git commit SHA 태그를 기록하고 내부 통신 검증 준비가 끝난 뒤에만 제외 규칙을
별도 변경으로 해제한다.
