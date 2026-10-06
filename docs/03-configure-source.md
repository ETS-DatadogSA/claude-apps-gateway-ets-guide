# 3. Entra 용 설정값 반영

원본은 Okta 기준이라 설정 세 곳이 Entra 와 맞지 않습니다. 두 곳은 이 리포에 이미 반영돼 있고, 어드민 그룹 GUID 한 곳만 직접 채웁니다. 리포 루트에서 실행합니다.

| 위치 | 원본 값 | 이 리포 값 | 이유 |
| --- | --- | --- | --- |
| `gateway/gateway.yaml` `oidc.userinfo_fallback` | `true` | `false` (반영됨) | Entra 의 userinfo 는 groups 를 주지 않음 |
| `gateway/gateway.yaml` `oidc.scopes` | `groups` 포함 | `groups` 제거 (반영됨) | Entra 에는 `groups` 스코프가 없음 ([1.3](01-entra-id.md#13-groups-클레임-활성화)) |
| `admin-console/app/auth.py` `ADMIN_GROUP_NAME` | `"claude-gateway-admins"` | 자리표시자 → **`$GRP` 로 교체** | Entra 는 그룹명이 아니라 GUID 를 보냄 |

<details>
<summary>키를 지우지 않고 <code>false</code> 로 둔 이유</summary>

Anthropic 공식 문서는 Entra 로 갈 때 `userinfo_fallback` 과 `groups` 스코프를 함께 제거하라고 안내합니다. 키를 생략했을 때의 기본값을 확인할 수 없어, 이 리포는 `false` 로 명시했습니다.
</details>

## 3.1 관리 콘솔의 어드민 그룹 지정

[1.5](01-entra-id.md#15-어드민-그룹-생성)의 그룹 GUID 를 넣습니다.

macOS:

```bash
sed -i '' "s/<ENTRA_ADMIN_GROUP_OBJECT_ID>/$GRP/" admin-console/app/auth.py && grep -n "^ADMIN_GROUP_NAME" admin-console/app/auth.py
```

Windows Git Bash / Linux:

```bash
sed -i "s/<ENTRA_ADMIN_GROUP_OBJECT_ID>/$GRP/" admin-console/app/auth.py && grep -n "^ADMIN_GROUP_NAME" admin-console/app/auth.py
```

> [!WARNING]
> 자리표시자를 그대로 두고 배포해도 배포는 성공하지만, 아무도 콘솔 관리자 권한을 받지 못합니다.

<details>
<summary>CDK 컨텍스트 <code>adminOktaGroupName</code> 만으로 안 되는 이유</summary>

컨텍스트 값은 게이트웨이 컨테이너 환경변수로만 들어가고 관리 콘솔 컨테이너에는 주입되지 않습니다. 그래서 콘솔은 소스에서 직접 맞춰야 합니다. Okta 에서도 기본값과 다른 그룹명을 쓰면 같은 문제가 생깁니다.
</details>

## 3.2 확인

```bash
grep -n "userinfo_fallback:\|^  scopes:" gateway/gateway.yaml && grep -n "^ADMIN_GROUP_NAME" admin-console/app/auth.py
```

기대 출력:

```
userinfo_fallback: false
scopes: [openid, profile, email, offline_access]
ADMIN_GROUP_NAME = "<GRP 의 GUID>"
```

`auth.py` 의 GUID 는 테넌트 고유값이라 커밋하지 않습니다. 되돌리려면 `git checkout -- admin-console/app/auth.py`.

App Roles 를 쓰는 대안도 있습니다. App Role 을 정의하고 `gateway.yaml` 에 `oidc.groups_claim: roles` 를 두면 GUID 대신 role 값이 들어옵니다(이 가이드에서는 미검증).

다음: [4. CDK 배포](04-deploy.md)
