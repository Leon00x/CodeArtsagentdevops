# 基于华为云 CodeArts Agent 构建华为云运维助手

> **华为云运维助手** = CodeArts Agent + 华为云 Skills + hcloud CLI（KooCLI）+ Terraform
>
> 通过自然语言对话即可完成华为云资源的**查询、创建、部署、监控、成本分析与清理**，
> 无需在多个控制台之间切换，也无需手写复杂 API 调用。

---

## 目录

1. [什么是华为云运维助手](#1-什么是华为云运维助手)
2. [工具链总览](#2-工具链总览)
3. [安装与配置 hcloud CLI](#3-安装与配置-hcloud-cli)
4. [安装与配置 Terraform](#4-安装与配置-terraform)
5. [安装与配置 Skills](#5-安装与配置-skills)
6. [配置 SSH 免密登录（COC）](#6-配置-ssh-免密登录coc)
7. [环境自检清单](#7-环境自检清单)
8. [常见问题（FAQ）](#8-常见问题faq)

---

## 1. 什么是华为云运维助手

华为云运维助手是一套**面向代码智能体的云运维工具组合**，由四层能力构成：

| 层 | 组件 | 职责 |
|----|------|------|
| 交互层 | **CodeArts Agent** | 理解自然语言指令，编排多步运维任务 |
| 能力层 | **华为云 Skills**（15 个） | 把华为云 API 封装为"可对话的技能"，覆盖计算/网络/存储/监控/账单 |
| 执行层 | **hcloud CLI（KooCLI）** | 实际调用华为云 OpenAPI 的官方命令行工具 |
| 编排层 | **Terraform** | 声明式基础设施即代码（IaC），可复用、可销毁 |

**典型使用场景：**

- 查询："现在有哪些 ECS 在运行？""本月消费多少？""哪些 EIP 是闲置的？"
- 部署："把 VPC + 子网 + EIP + ECS 搭建起来，区域 ap-southeast-3"
- 运维："查看 ECS CPU 使用率""诊断 8000 端口为什么不通"
- 排障："ECS 创建失败帮我定位原因"
- 清理："销毁所有资源，避免继续计费"

---

## 2. 工具链总览

| 组件 | 作用 | 安装位置（本机示例） | 验证命令 |
|------|------|---------------------|---------|
| hcloud | 华为云 API 命令行 | `C:\Users\<你>\.local\bin\hcloud.exe` | `hcloud version` |
| Terraform | IaC 编排 | `C:\Users\<你>\AppData\Local\Terraform\terraform.exe` | `terraform version` |
| 华为云 provider | Terraform 华为云插件 | `%APPDATA%\terraform.d\plugins\...` | `terraform providers` |
| Skills | 华为云技能 | 用户级 `~/.codeartsdoer/skills/` 或工作区级 `.agents/skills/` | `skillhub list` |
| SSH 密钥 | 登录 ECS | `~/.ssh/id_rsa` / `id_rsa.pub` | `ls ~/.ssh/` |

> **建议顺序**：先装 hcloud 并配好凭证 → 再装 Terraform → 最后装 Skills（Skills 依赖 hcloud 执行真实操作）。

---

## 3. 安装与配置 hcloud CLI

hcloud（KooCLI）是华为云官方命令行工具，是整套装的核心执行引擎。

### 3.1 安装

**Windows（PowerShell）：**

```powershell
# 创建安装目录
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.local\bin" | Out-Null

# 下载 KooCLI（中国站镜像）
Invoke-WebRequest `
  -Uri "https://cn-north-1-hcli-obs.obs.cn-north-1.myhuaweicloud.com/hcli/hcloud.exe" `
  -OutFile "$env:USERPROFILE\.local\bin\hcloud.exe"

# 加入 PATH（当前会话生效）
$env:Path += ";$env:USERPROFILE\.local\bin"

# 验证
hcloud version
```

**Linux / macOS：**

```bash
mkdir -p ~/.local/bin
curl -fsSL https://cn-north-1-hcli-obs.obs.cn-north-1.myhuaweicloud.com/hcli/hcloud \
  -o ~/.local/bin/hcloud
chmod +x ~/.local/bin/hcloud
export PATH="$HOME/.local/bin:$PATH"
hcloud version
```

### 3.2 配置 AK/SK 凭证

AK/SK 获取路径：华为云控制台 → 右上角头像 → **我的凭证** → **访问密钥**。

```bash
# 交互式初始化
hcloud configure init

# 按提示填写：
#   Authentication mode : AKSK
#   Access Key ID (AK)  : <你的 AK>
#   Secret Access Key   : <你的 SK>
#   Region              : ap-southeast-3   # 国际站新加坡；中国站可用 cn-north-4
```

> **安全说明**：hcloud 会把 AK/SK **加密**存储在 `~/.hcloud/config.json`，
> 加密材料见 `~/.hcloud/cipher.json`。因此 `hcloud configure show` **只显示脱敏值**
> （如 `HPU****G9I`），这是设计如此，不是配置错误。

### 3.3 验证认证

```bash
# 查看配置（脱敏）
hcloud configure show

# 测试身份（会真实调用 STS API）
hcloud STS GetCallerIdentity --cli-region=ap-southeast-3
# 期望返回 account_id / principal_urn / principal_id
```

### 3.4 区域与端点说明（重要）

| 场景 | 服务名 | region | 说明 |
|------|--------|--------|------|
| 通用资源（ECS/VPC/EIP/EVS…） | `ECS` `VPC` `EIP` … | `ap-southeast-3` | 国际站主区域 |
| **账单/费用** | **`BSSINTL`** | **`ap-southeast-1`** | 国际站账单**仅**支持 BSSINTL 且仅 ap-southeast-1 |
| 身份认证 | `IAM` / `STS` | 跟随 profile region | 支持全局端点 |

> ⚠️ 国际站账号调用账单时，**不要**用中国站的 `BSS` 服务，
> 否则会报 `[USE_ERROR]不支持的服务名称:BSS`。正确用法：

```bash
hcloud configure set --cli-lang=cn    # 首次建议设置语言
hcloud BSSINTL ShowCustomerMonthlySum --bill_cycle=2026-10 --cli-region=ap-southeast-1 --cli-output=json
```

### 3.5 常用自检命令

```bash
# 列出可用区
hcloud ECS NovaListAvailabilityZones --cli-region=ap-southeast-3

# 查询 2vCPU/4GB 规格
hcloud ECS NovaListFlavors --cli-region=ap-southeast-3 --minRam=4096

# 查询 Ubuntu 22.04 公共镜像
hcloud IMS ListImages --cli-region=ap-southeast-3 \
  --__imagetype=gold --__os_type=Linux --__platform=Ubuntu

# 列出当前 VPC
hcloud VPC ListVpcs --cli-region=ap-southeast-3
```

---

## 4. 安装与配置 Terraform

Terraform 用于声明式地创建/销毁一整套基础设施（VPC、子网、安全组、EIP、ECS 等）。

### 4.1 安装 Terraform CLI

```powershell
# Windows：下载 zip 并解压
Invoke-WebRequest -Uri "https://releases.hashicorp.com/terraform/1.15.2/terraform_1.15.2_windows_amd64.zip" -OutFile terraform.zip
Expand-Archive terraform.zip -DestinationPath "$env:LOCALAPPDATA\Terraform" -Force

# 加入 PATH
$env:Path += ";$env:LOCALAPPDATA\Terraform"
terraform version   # 期望 v1.15.2
```

> 也可以用技能一键完成：对话输入 **"安装 Terraform 1.15.2 + 华为云 provider"**，
> 由 `huawei-cloud-terraform-installer` 自动处理。

### 4.2 配置华为云 provider 本地镜像

国内/内网环境直接 `terraform init` 拉取 provider 可能超时，推荐配置**文件系统镜像**：

```powershell
# 1) 准备目录（版本号与所需一致）
$mirror = "$env:APPDATA\terraform.d\plugins\registry.terraform.io\huaweicloud\huaweicloud\1.97.2\windows_amd64"
New-Item -ItemType Directory -Force -Path $mirror | Out-Null

# 2) 把 provider 二进制放入该目录
#    下载地址示例：
#    https://releases.hashicorp.com/... 或华为云镜像站
#    下载后重命名为 terraform-provider-huaweicloud_v1.97.2.exe
```

### 4.3 配置 terraform.rc（使用本地镜像）

在 `%APPDATA%\terraform.rc` 写入：

```hcl
provider_installation {
  filesystem_mirror {
    path    = "C:/Users/<你的用户名>/AppData/Roaming/terraform.d/plugins"
    include = ["registry.terraform.io/huaweicloud/*/*"]
  }
  direct {
    exclude = ["registry.terraform.io/huaweicloud/*/*"]
  }
}
```

### 4.4 验证

```bash
cd <你的 terraform 目录>
terraform init      # 应显示 "Installed huaweicloud/huaweicloud v1.97.2"
terraform version
```

> 💡 `terraform init` 阶段**不需要** AK/SK，只有 `plan/apply` 才需要凭证。

---

## 5. 安装与配置 Skills

Skills 是华为云运维助手的"技能包"，每个 Skill 封装一类华为云能力，让 Agent 能直接操作真实云资源。

### 5.1 什么是 Skill

- 一个 Skill = 一份 `SKILL.md`（操作指引）+ 可选参考文档/脚本。
- 存放位置（两种作用域）：
  - **用户级**：`~/.codeartsdoer/skills/`（当前用户所有项目可用）
  - **工作区级**：`<项目>/.agents/skills/`（仅当前项目可用）
- Agent 会根据你的自然语言指令自动匹配并加载对应 Skill。

### 5.2 安装方式

**方式 A：用 skillhub CLI（推荐批量安装）**

```bash
# 搜索技能
skillhub search huawei-cloud

# 安装单个技能（slug 为技能名）
skillhub install huawei-cloud-computing-query

# 查看已安装
skillhub list

# 升级全部已装技能
skillhub upgrade
```

**方式 B：在 CodeArts Agent 对话中安装**

直接说：

```
帮我安装华为云技能
```

由 `huawei-cloud-find-skills` 负责搜索、发现并安装所需技能。

### 5.3 技能清单与用途（15 个）

| # | 技能名称 | 用途 |
|---|---------|------|
| 1 | `huawei-cloud-find-skills` | 搜索、发现、安装其他华为云技能 |
| 2 | `huawei-cloud-computing-query` | 查询计算资源（ECS/BMS/IMS/AS 规格、镜像、配额） |
| 3 | `huawei-cloud-ecs-manage` | ECS 全生命周期（创建/启停/重启/删除/创建失败诊断） |
| 4 | `huawei-cloud-ecs-passwordless-login` | 通过 COC 配置免密 SSH 登录 ECS（批量） |
| 5 | `huawei-cloud-ces-ecs-monitoring` | CES 云监控（CPU/内存/磁盘/网络指标） |
| 6 | `huawei-cloud-lts-log-inspector` | LTS 日志检查（流量统计、上下文查询、采集巡检） |
| 7 | `huawei-cloud-terraform-generator` | 生成华为云 Terraform 配置 |
| 8 | `huawei-cloud-terraform-installer` | 安装 Terraform CLI + 华为云 provider |
| 9 | `huawei-cloud-solution-ops` | 综合运维（检查/规划/操作/排障/验证） |
| 10 | `huawei-cloud-network-query` | 查询网络资源（VPC/子网/安全组/EIP/ELB/NAT/VPN/DNS） |
| 11 | `huawei-cloud-vpc-network-diagnosis-management` | VPC 网络诊断（连通性/端口/CIDR 冲突） |
| 12 | `huawei-cloud-evs-disk-create` | 创建 EVS 云硬盘 |
| 13 | `huawei-cloud-eip-cost-optimizer` | EIP 成本优化（闲置分析/审计/监控） |
| 14 | `huawei-cloud-deployment-task-management` | CodeArts Deploy 部署任务管理 |
| 15 | `huawei-cloud-billing-scout` | BSS 账单只读查询（余额/月账单/扣费归因/对账） |

### 5.4 按能力分类

```
┌─ 资源发现 ─── find-skills · computing-query · network-query
├─ 基础设施 ─── ecs-manage · vpc-network-diagnosis-management · evs-disk-create
├─ IaC 编排 ─── terraform-installer · terraform-generator
├─ 部署运维 ─── ecs-passwordless-login · deployment-task-management · solution-ops
├─ 监控观测 ─── ces-ecs-monitoring · lts-log-inspector
└─ 成本账单 ─── eip-cost-optimizer · billing-scout
```

### 5.5 技能使用速查

| 场景 | 技能 | 对话示例 |
|------|------|---------|
| 查找技能 | find-skills | "帮我找管理 ECS 的 skill" |
| 查规格 | computing-query | "查询 ap-southeast-3 可用的 2vCPU 4GB flavor" |
| 查镜像 | computing-query | "查询 Ubuntu 22.04 公共镜像 ID" |
| 建 ECS | ecs-manage | "创建 ECS：s6.large.2, Ubuntu 22.04, 40GB" |
| 免密登录 | ecs-passwordless-login | "配置到 ECS <id> 的免密 SSH" |
| 看监控 | ces-ecs-monitoring | "查看 ECS CPU 使用率" |
| 查日志 | lts-log-inspector | "查看采集状态" |
| 生成 IaC | terraform-generator | "生成创建 VPC+ECS+EIP 的 Terraform 代码" |
| 装 Terraform | terraform-installer | "安装 Terraform + 华为云 provider" |
| 网络诊断 | vpc-network-diagnosis-management | "诊断到 8000 端口不通的原因" |
| 查网络 | network-query | "列出所有 VPC 和子网" |
| 建磁盘 | evs-disk-create | "创建 100GB GPSSD 数据盘" |
| EIP 成本 | eip-cost-optimizer | "分析闲置 EIP 并生成报告" |
| 部署任务 | deployment-task-management | "列出所有部署应用和任务" |
| 查账单 | billing-scout | "查询我的华为云本月账单" |

---

## 6. 配置 SSH 免密登录（COC）

部署到 ECS 后需要 SSH 登录。推荐用 COC（云运维中心）推送公钥实现免密登录。

### 6.1 本地生成密钥对

```bash
ssh-keygen -t rsa -b 2048 -f ~/.ssh/id_rsa -N ""
cat ~/.ssh/id_rsa.pub
```

### 6.2 通过 COC 推送公钥

两种方式：

1. **对话方式（推荐）**：使用 `huawei-cloud-ecs-passwordless-login` 技能：
   ```
   帮我把本地公钥推送到 ECS <instance_id>，配置免密登录
   ```
2. **手动方式**：登录 ECS 后用 `authorized_keys` 追加公钥，或在创建 ECS 时指定 `key_pair`。

### 6.3 验证

```bash
ssh -i ~/.ssh/id_rsa root@<EIP> "hostname && docker --version"
```

> ⚠️ **注意**：若 ECS 是用**临时凭证（IAM Agency）**创建的，则用**原始 AK/SK** 创建的 KPS 密钥对
> 会因 user_id 不匹配而无法绑定。此时可先用 `admin_pass` 创建 ECS，登录后再推送公钥。

---

## 7. 环境自检清单

配置完成后，逐项确认：

```bash
# 1) hcloud 可用
hcloud version

# 2) hcloud 认证通过
hcloud STS GetCallerIdentity --cli-region=ap-southeast-3

# 3) Terraform 可用
terraform version

# 4) 华为云 provider 就绪（在 terraform 目录执行）
terraform init

# 5) Skills 已安装
skillhub list

# 6) SSH 密钥存在
ls ~/.ssh/id_rsa ~/.ssh/id_rsa.pub
```

全部通过后，即可开始用自然语言运维华为云资源。

---

## 8. 常见问题（FAQ）

| 现象 | 原因 | 解决 |
|------|------|------|
| `hcloud configure show` 显示 `HPU****G9I` | AK/SK 加密存储，脱敏显示属正常 | 无需处理，直接使用 |
| `[USE_ERROR]不支持的服务名称:BSS` | 国际站账单不支持中国站 `BSS` | 改用 `BSSINTL` + `--cli-region=ap-southeast-1` |
| `terraform init` 卡住/超时 | provider 需联网下载 | 配置 `filesystem_mirror`（见 4.2/4.3） |
| Terraform 报 `Authentication failed` | 未传凭证或临时凭证过期 | 设置 `TF_VAR_access_key/secret_key/security_token` |
| Agency 临时凭证无权建资源 | Agency 未绑定权限 | 绑定项目级权限（ECS/VPC/EIP/EVS FullAccess） |
| KPS 密钥对无法绑定 ECS | 密钥对 user_id 与临时凭证不匹配 | 用 `admin_pass` 创建，登录后推送公钥 |
| 技能未生效 | 未安装或作用域不对 | `skillhub list` 检查；确认 `~/.codeartsdoer/skills/` 或 `.agents/skills/` |
| `terraform.rc` 改了不生效 | 需新开终端 | 关闭并重开终端后重试 |

---

> 下一步：参考 [example.md](./example.md) 了解如何把一个真实应用部署到华为云。