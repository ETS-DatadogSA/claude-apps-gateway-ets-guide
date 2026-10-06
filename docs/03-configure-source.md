# 3. Entra 용 설정값 반영

업스트림은 Okta 기준으로 짜여 있어 설정값 세 곳이 Entra 와 맞지 않습니다. 이 세 곳만 바꾸고 나머지 소스는
건드리지 않습니다. 리포 루트에서 실행합니다(macOS `sed` 기준).

| 위치 | 업스트림 값 | Entra 값 | 이유 |
| --- | --- | --- | --- |
| `gateway/gateway.yaml` `oidc.userinfo_fallback` | `true` | `false` | Okta org 서버의 thin id_token 대응용. Entra 의 userinfo 는 groups 를 주지 않음 |
| `gateway/gateway.yaml` `oidc.scopes` | `groups` 포함 | `groups` 제거 | Entra 에 `groups` 스코프가 없음. 그룹은 optional claim 으로 옴([1.3](01-entra-id.md#13-groups-클레임-활성화)) |
| `admin-console/app/auth.py` `ADMIN_GROUP_NAME` | `"claude-gateway-admins"` | `"$GRP"` (GUID) | 콘솔이 그룹명을 하드코딩해 비교하는데, Entra 는 GUID 를 내보냄 |

Anthropic 공식 문서는 Entra 로 갈 때 `userinfo_fallback` 과 `groups` 스코프를 **함께** 제거하라고 안내합니다.
이 가이드는 키를 지우는 대신 `false` 로 명시합니다. 생략했을 때의 기본값을 확인할 수 없기 때문입니다.

## 3.1 userinfo fallback 끄기

```bash
sed -i '' 's/^  userinfo_fallback: true$/  userinfo_fallback: false/' gateway/gateway.yaml && grep -n "userinfo_fallback:" gateway/gateway.yaml
```

## 3.2 groups 스코프 제거

```bash
sed -i '' 's/^  scopes: \[openid, profile, email, offline_access, groups\]$/  scopes: [openid, profile, email, offline_access]/' gateway/gateway.yaml && grep -n "^  scopes:" gateway/gateway.yaml
```

## 3.3 관리 콘솔의 어드민 그룹 지정

```bash
sed -i '' "s/^ADMIN_GROUP_NAME = \"claude-gateway-admins\"$/ADMIN_GROUP_NAME = \"$GRP\"/" admin-console/app/auth.py && grep -n "^ADMIN_GROUP_NAME" admin-console/app/auth.py
```

이 값은 CDK 컨텍스트로 넘기는 `adminOktaGroupName` 과 별개입니다. 컨텍스트 값은 게이트웨이 컨테이너의 환경변수로만
전달되고 관리 콘솔 컨테이너에는 주입되지 않습니다. 그래서 콘솔 쪽은 소스에서 직접 맞춰야 합니다. Okta 에서도 기본값과
다른 그룹명을 쓰면 같은 문제가 생깁니다.

## 3.4 확인

세 명령 모두 `grep` 으로 바뀐 줄을 다시 출력합니다. 기대 출력은 다음과 같습니다.

```
userinfo_fallback: false
scopes: [openid, profile, email, offline_access]
ADMIN_GROUP_NAME = "<GRP 의 GUID>"
```

값이 그대로라면 업스트림이 해당 줄을 고친 것입니다. 파일을 열어 같은 키를 직접 고칩니다.

변경 범위가 이 두 파일뿐인지 확인합니다. [2.2](02-aws-preparation.md#22-의존성과-번들러)에서 esbuild 를 넣었다면
`cdk/package.json` 과 `cdk/package-lock.json` 도 함께 보입니다.

```bash
git status --short
```

> App Roles 를 쓰는 방법도 있습니다. Entra 앱에 App Role 을 정의하고 `gateway.yaml` 에 `oidc.groups_claim: roles`
> 를 두면 claim 에 GUID 가 아닌 role 값이 들어옵니다. role 이름을 `claude-gateway-admins` 로 맞추면
> `auth.py` 를 고치지 않아도 될 수 있으나, 이 가이드에서는 검증하지 않았습니다.

다음: [4. CDK 배포](04-deploy.md)
