# AWS Solutions Architect 溫習筆記
(由 Hermes 同 Alfred 嘅對話整理 · 2026-08-27)

---

## 1. AWS Landing Zone

用嚟建立符合最佳實踐、可擴展、安全嘅**多帳號環境**。重點:帳號分層 + 中央管治 + 中央審計。

1. **設計帳號結構 (OU)** — 用 AWS Organizations:
   - Root ├─ Security (Log Archive + Security Tooling) ├─ Infrastructure (網絡/共享) └─ Workloads(Dev / Staging / Prod)
2. **起 Landing Zone** — 用 **AWS Control Tower**(托管)或手動 Organizations。Control Tower 自動起 Log Archive + 管理帳號兩個核心。
3. **用戶登入** — **IAM Identity Center**(前 SSO),單一登入,接 AD/SAML federation。
4. **集中審計 + 保安** — 中央 CloudTrail、GuardDuty、Security Hub、Config、AWS Budgets。
5. **Guardrails**:
   - **Preventive(阻止型)= SCP**(Service Control Policy)喺 OU 層禁止嘢
   - **Detective(偵察型)= Config** 規則,違規自動警報
6. **帳號自動化供應** — **Account Factory (AFT)** / Control Tower 統一規格開新帳號。

**考試必記:** ① Management Account 唔好放 workload,只做管治。② SCP 係 OU 層 white-list,只 allow/deny,唔可以 grant,要落喺 OU 層先有效。③ Control Tower guardrails 分 mandatory 同 strongly recommended。

---

## 2. EKS Upgrade

**硬規則:** 唔可以跳 minor version;control plane 同 node 最多差 1 個 minor(version skew,node 可等於或舊一版);Standard Support 約 14 個月,Extended ~26 個月;**冇得 rollback control plane**。

**流程:**
1. **評估** — 睇 Kubernetes changelog/deprecation(已廢棄 API、CRD/Webhook 要配新版)。
2. **Add-on 相容** — 確認 CoreDNS、kube-proxy、**VPC CNI**、EBS/EFS CSI 有新版。
3. **升 Control Plane** — EKS 自動 rolling(multi-AZ),短暫 outage,API 升級期間較慢。
4. **升 Node Group(最緊要)** — cordon → drain → 等 Pod 重排;**一定要有 PDB**,用 TopologySpreadConstraints / anti-affinity 分散副本。方法:rolling(逐批)或 blue-green(開新 node group 遷移)。
5. **升 Add-on + 驗證** — smoke test、清 Terminating/NotReady。

**Add-on 次序真相:** 唔係「一定最後」。加-on 同 control plane 係「互通相容」關係。**有相容風險嘅(尤其 VPC CNI / CoreDNS)反而建議喺升 control plane 前/同步就升**。其它 add-on 可喺 node group 後彈性處理,未必要一齊 done。

**考試考:** ① node 比 cp 舊多於 1 版唔 join。② 冇 PDB 滾動升級 = 服務中斷。③ 唔跳版本 + 14 個月。④ add-on 要獨立 trigger 升。⑤ 先喺 staging 升先。

---

## 3. CloudFormation

**IaC**,用 JSON/YAML template 宣告式建 AWS 資源,刪 template = 成批資源刪。

**Template sections:** AWSTemplateFormatVersion / Description / Parameters / Mappings / Resources(★必需) / Outputs / Metadata / Conditions。

**Deploy:** `aws cloudformation deploy --stack-name my-stack --template-file tpl.yaml --parameter-overrides Env=prod`。SAM 用 `sam deploy`。

**關鍵特性:**
- **Rollback 自動** — 建失敗自動回滾。
- **Change Set** — 部署前「預覽」改咩,唔會直接改資源,要另外 apply。
- **UpdatePolicy** — RollingUpdate / Recreate。
- **Drift Detection** — 手動 console 改咗資源唔經 CFN 會漂移;`detect-stack-drift` 偵測,但唔會自動修正。
- **Nested Stacks** — 大 stack 內嵌 `AWS::CloudFormation::Stack`(module 重用)。
- **Cross-stack reference** — `Outputs` + `Fn::ImportValue`。
- **Intrinsic Functions:** `!Ref` / `!GetAtt` / `!Sub` / `!Join` / `!FindInMap` 等。
- **CreationPolicy/WaitCondition** — 等 instance ready 先完成。

**同其他工具:** CFN(原生 declarative,有 Change Set/Drift)| Terraform(跨雲)| SAM(CFN 嘅 serverless 強化)| CDK(用 Python/TS 寫 code 生成 CFN template)| Beanstalk(PaaS)。

**Trap:** ① 刪 stack = 刪晒資源,要保留用 `DeletionPolicy: Retain`。② Change Set 只預覽唔 apply。③ Drift 只報告唔修正。④ Nested ≠ Cross-stack。

---

## 4. SAM 全名

**Serverless Application Model** — 建喺 CloudFormation 之上,用 `AWS::Serverless::*`(如 `AWS::Serverless::Function` 自動起埋 Lambda+IAM role)簡化 serverless 部署;`sam build` + `sam deploy` + `sam local invoke`;最終會轉化成交畀 CloudFormation。

---

## 5. CloudFormation Stack

Stack = 一個 template 部署一次嘅「實例」,管理成批 resources 生命周期。

**狀態:** CREATE/UPDATE/DELETE 各 *_IN_PROGRESS/*_COMPLETE/*_FAILED + ROLLBACK_*。REVIEW_IN_PROGRESS = 用 Change Set 建立時。

**Stack 互動兩方式:**
- **Outputs + ImportValue(跨 stack):** Export 名要成個 region 唯一;某人引用住就先刪唔到。
- **Nested Stack:** 一個 stack 內嵌另一個 template。

**特性:** 刪除按依賴倒序;`DeletionPolicy: Retain/Snapshot` 保留資源;**CFN 好處 = 冇 state 檔案**(AWS 幫你 track,唔似 Terraform 要自己管 state)。

---

## 6. StackSets

用**一個 operation 一次過喺多個帳號 × 多個 region** 部署同一 infra。每個「帳號×region」組合叫一個 **Stack Instance**。

- **Administration Account** 負責建 StackSet。
- 跨帳號要兩個 role:**AdministrationRole(assume)** + 每個 target 帳號嘅 **ExecutionRole**。
- **Service-managed:** 直接配 Organizations,新帳號/新 region 自動加入(推薦)。
- **Self-managed:** 手動指定 target accounts + regions。

**對比:**
| 概念 | 用途 |
|---|---|
| Stack | 單一帳號 × 單 region |
| StackSet | 多帳號 × 多 region |

**重要釐清:** StackSet 主要係「跨**帳號**」一次過管。**單一帳號 × 多 region**「唔一定要」用 StackSet —— 寫一個 template,喺每個 region 各 `deploy --region` 就好簡單。**考試:** 考「多帳號跨 region」→ StackSet;淨係「多 region」冇提多帳號 → 普通 stack 分開 deploy。

---

## 7. Route 53 Resolver (Inbound / Outbound)

Router 53 Resolver = 處理「VPC ↔ 外面」DNS 互通嘅中間人。三大件:Inbound Endpoint、Outbound Endpoint、Resolver Rules。

**Inbound Resolver ★** — **on-prem → 想查 VPC 內部 DNS**(如私有 hosted zone 嘅 RDS private name)。
- on-prem DNS client → [Inbound Endpoint ENI] → VPC 內 DNS
- 係一個 ENI 附喺 VPC 子網,開**至少 2 個 subnet** 做高可用;SG 控制邊啲 on-prem 可以 query。

**Outbound Resolver ★** — **VPC 內資源 → 想查 on-prem/自訂 DNS**(如 `corp.internal`)。
- VPC 內 instance → [Outbound Endpoint] → on-prem DNS
- 配 **Resolver Rules** 決定邊啲 domain forward 去邊(如 `corp.internal` → on-prem DNS IP)。

**對比:**
| | Inbound | Outbound |
|---|---|---|
| 方向 | on-prem → 入 VPC | VPC → 出外面 |
| 解決 | on-prem 查唔到 VPC internal DNS | VPC 查唔到 on-prem DNS |
| 例 | on-prem connect RDS private name | EC2 call 用 `corp.internal` name |

**常見場景:** Hybrid DNS 互通(兩個 endpoint 一齊建)、DR/多 region 私有 DNS 互通。

---

## 8. CloudFront Functions

輕量、超快、喺 CloudFront edge 行嘅 **JavaScript(ES5/ES6 子集)** 做 request/response 輕改寫。**兩個事件位:Viewer Request / Viewer Response。**

**特性:** <1ms 延遲、極平(每百萬 request ~$0.10 vs Lambda@Edge 貴好多)、儲存喺 edge、**冇 network/IO 存取**(純 CPU + header 改動)、分 DEVELOPMENT/LIVE。

**用途:** URL rewrite/redirect、Header 修改(CORS/cache-control)、JWT/Token 檢查、Geo/Device 分流。

**vs Lambda@Edge:**
| | CloudFront Functions | Lambda@Edge |
|---|---|---|
| 語言 | JS 子集 | Node/Python 等全語系 |
| 事件位 | 得 Viewer Request/Response | Viewer + Origin (更多) |
| 能力 | 極輕量改 header/URL,冇 IO/DB | 可 IO、DB、動態生成 |
| 成本 | 平好多 | 貴 |
| 延遲 | 幾 ms | 較大 |

**考:** 只要 Viewer 層輕 transform(改 header、redirect、token check)→ **CloudFront Functions**;要 Origin-level 事件/IO/DB → **Lambda@Edge**。

---

## 9. AWS Backup — 每日 EC2 備份 + Encrypted EBS

**基本:** 開 Backup Vault → 建 Backup Plan(set `ScheduleExpression` cron + Lifecycle DeleteAfterDays/ColdStorage)→ 用 IAM role + tag 指派 resources。

**加密 EBS:**
- **AWS Backup 會自動做加密備份**,唔使自己搞。
- 預設用**返原本 EBS 嘅 KMS key** 加密;但若 **Backup Vault 有 set 另一個 KMS key**,會 **override 用 vault 個 key**。

**注意位:** ① Cross-region/cross-account copy 用新 region/vault 嘅 key 重新加密。② Restore 出嚟可揀原 key 或改 key。③ 刪 KMS key → 備份 restore 唔到(**KMS key 唔好亂刪**;cross-account 要 set key policy/grant)。④ Default EBS 用 `aws/ebs` key 都得,但自己管嘅 **CMK** 更穩陣。

**建議架構:** 自己管 CMK → Backup Vault 加密;EC2 tag(如 `Backup:Daily`)tag-based selection 自動入 plan;Lifecycle 控成本。

---

*(完)*