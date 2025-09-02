<!-- BEGINNING OF PRE-COMMIT-TERRAFORM DOCS HOOK -->
## 입력

| 이름 | 설명 | 유형 | 기본값 | 필수 |
|------|-------------|------|---------|:--------:|
| associations | Resources to associate the policy to | `list(string)` | n/a | yes |
| name | Hierarchical policy name | `string` | n/a | yes |
| parent | Where the firewall policy will be created (can be organizations/{organization\_id} or folders/{folder\_id}) | `string` | n/a | yes |
| rules | Firewall rules to add to the policy | <pre>map(object({<br>    description             = string<br>    direction               = string<br>    action                  = string<br>    priority                = number<br>    ranges                  = list(string)<br>    ports                   = map(list(string))<br>    target_service_accounts = list(string)<br>    target_resources        = list(string)<br>    logging                 = bool<br>  }))</pre> | `{}` | no |

## 출력

| 이름 | 설명 |
|------|-------------|
| id | n/a |

<!-- END OF PRE-COMMIT-TERRAFORM DOCS HOOK -->
