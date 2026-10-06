# 7. 관리 콘솔 사용법

관리 콘솔에는 사용액 대시보드, 비용 한도 관리, 모델 접근 관리 세 가지 기능이 있습니다. 이 문서는 각 기능의 사용법과,
특히 기능마다 내부 동작이 달라 감사 기록이 어떻게 남는지가 달라진다는 점을 다룹니다. 원본은
[original/04-admin-console-guide.md](original/04-admin-console-guide.md) 입니다.

![관리 콘솔의 두 가지 쓰기 경로와 감사 기록](images/07-admin-console.drawio.png)

| 기능 | 경로 | 실제 동작 | 기록이 남는 곳 |
| --- | --- | --- | --- |
| 사용액 대시보드 | `/spend/dashboard` | 게이트웨이 `/v1/organizations/effective` 조회 | (읽기) |
| 비용 한도 | `/spend/limits` | 게이트웨이 관리 API `/v1/organizations/spend_limits` 호출 | 게이트웨이 감사 로그, 관리자 본인 이름 |
| 모델 접근 | `/models` | 콘솔 IAM 롤이 `ecs:UpdateExpressGatewayService` 호출 | CloudTrail, 콘솔 ECS task role 이름 |

## 7.1 사용액 대시보드와 비용 한도

**대시보드**(`/spend/dashboard`)는 선택한 기간의 실사용액을 범위별(조직, RBAC 그룹, 사용자)로 보여 줍니다. 값은
게이트웨이 자체 API `/v1/organizations/effective` 에서 실시간으로 읽습니다.

**한도**(`/spend/limits`)에서는 비용 한도를 만들고, 보고, 지웁니다. 범위는 조직 전체, RBAC 그룹, 개별 사용자 중
하나이고, 한도마다 금액과 기간(`daily`, `weekly`, `monthly`)을 정합니다.

여기서 하는 모든 작업(생성, 삭제)은 게이트웨이 관리 API(`/v1/organizations/spend_limits`)를 직접 호출합니다. 인증에는
**사인인한 관리자 본인에게 게이트웨이가 발급한 bearer 토큰**을 씁니다. 이 토큰은 Claude Code CLI 가 개발자를
사인인시킬 때와 같은 device 인증 흐름으로 받습니다. 콘솔은 고정된 관리자 자격증명을 들고 있지 않습니다. 그 결과는
다음과 같습니다.

- 모든 비용 한도 변경이 게이트웨이 감사 로그에 남고, 행위자는 `oidc:<관리자의 Entra subject>` 로 기록됩니다.
- 관리자가 Entra 어드민 그룹에서 빠지면, 게이트웨이가 다음에 확인하는 시점부터 그 브라우저 세션은 이 호출을 할 수
  없습니다. 따로 회수할 자격증명이 없습니다.

## 7.2 모델 접근

**모델**(`/models`)은 배포 리전의 Amazon Bedrock 에서 실시간으로 가져온 Anthropic 모델 카탈로그 전체를 보여 주고,
각 모델이 게이트웨이에서 현재 켜져 있는지 표시합니다. 고정 목록이 아닙니다. 새로 배포한 직후에는 `cdk/lib/gateway-stack.ts`
의 기본값인 `claude-opus-4-8`, `claude-sonnet-4-6`, `claude-haiku-4-5` 세 개가 켜져 있습니다.

개발자에게 열어 줄 모델을 고르고 **Apply changes** 를 누릅니다. 모델마다 바로 바뀌는 토글이 아니라 일부러 묶어서 적용하는
방식입니다. 적용하면 게이트웨이가 실제로 다시 배포되기 때문입니다(환경변수를 바꾼 새 ECS 태스크). 여러 선택을 한 번에
적용하면 불필요한 재배포를 여러 번 하지 않아도 됩니다. 새 태스크가 뜨고 이전 태스크가 빠지는 동안 화면에 "Applying..."
이 보이고, 자리 잡으면 "Active" 로 바뀝니다. 보통 2분 안쪽입니다.

### 모델 카탈로그를 실시간으로 바꾸는 방식

게이트웨이 설정 형식(`gateway.yaml`)에는 환경변수 하나로 YAML 목록을 채우는 기능이 없습니다. 지원하는 것은 스칼라 값의
`${VAR}` 치환뿐입니다. 그래서 컨테이너 단계에서 해결했습니다. 게이트웨이의 엔트리포인트(`gateway/entrypoint.sh`)가
게이트웨이 프로세스가 시작해 파일을 읽기 전에 환경변수 `AVAILABLE_MODELS_RAW` 하나를 YAML 의 `availableModels` 목록으로
펼칩니다.

이 장치 하나로 "개발자가 쓸 수 있는 Claude 모델" 이 평범한 ECS 환경변수 갱신(`ecs:UpdateExpressGatewayService`)이
됩니다. 같은 컨테이너 이미지에 변수 값만 바뀌므로 재빌드, 배포 파이프라인, CI 대기가 없습니다. 콘솔에서 **Apply changes**
를 누르면 몇 분 안에 조직 전체에 새 카탈로그가 적용됩니다.

### 비용 한도와 다른 감사 방식

비용 한도와 달리 모델 접근 변경은 게이트웨이 API 호출이 **아닙니다**. 게이트웨이에는 자체 모델 허용 목록을 바꾸는 런타임
API 가 없습니다. 대신 콘솔 자신의 AWS IAM task role 이 게이트웨이 ECS 서비스에 `ecs:UpdateExpressGatewayService` 를 직접
호출합니다. 그래서 다음과 같습니다.

- 모델 카탈로그 변경은 게이트웨이 감사 로그에 **남지 않습니다**.
- AWS CloudTrail 에는 남지만, 행위자는 브라우저에서 Apply changes 를 누른 관리자가 아니라 관리 콘솔의 ECS task role 입니다.
  이 작업에 관리자별 기록이 필요하면 콘솔 자체에 애플리케이션 로그를 추가해야 합니다. 이 참조 구현에는 들어 있지 않습니다.

의도된 절충입니다. 인프라 변경에 가까운 작업을 위해 게이트웨이에 새 관리 API 를 여는 대신, 콘솔의 IAM 권한을 게이트웨이
서비스 하나에 대한 ECS 작업으로 좁게 유지합니다(`cdk/lib/admin-console-stack.ts` 의 task role 정책 참고).

다음: [8. 커스텀 추론 프로파일 (선택)](08-custom-inference-profile.md)
