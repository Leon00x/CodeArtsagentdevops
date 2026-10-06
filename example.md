# 部署示例：把应用部署到华为云 ECS

> **示例应用**：AssetMgmt 固定资产管理系统（FastAPI 后端 + Vue 前端 + PostgreSQL + Redis）
>
> **部署方式**：Docker Compose 部署到华为云 ECS（Ubuntu 22.04）
>
> **目标**：通过公网 EIP 访问 Web 页面并完成管理员登录验证。
>
> 本文方法对任意"用 Docker Compose 编排的多容器应用"均适用，替换镜像与端口即可。

---

## 目录

1. [示例应用与目标架构](#1-示例应用与目标架构)
2. [前置条件](#2-前置条件)
3. [Step 1 本地 Docker 预验证](#3-step-1-本地-docker-预验证)
4. [Step 2 获取临时凭证（IAM Agency）](#4-step-2-获取临时凭证iam-agency)
5. [Step 3 Terraform 创建基础设施](#5-step-3-terraform-创建基础设施)
6. [Step 4 部署应用到 ECS](#6-step-4-部署应用到-ecs)
7. [Step 5 验证访问](#7-step-5-验证访问)
8. [Step 6 监控与账单](#8-step-6-监控与账单)
9. [Step 7 销毁与清理](#9-step-7-销毁与清理)
10. [排错经验](#10-排错经验)

---

## 1. 示例应用与目标架构

### 1.1 应用组成

| 组件 | 技术 | 端口 |
|------|------|------|
| 后端 API | FastAPI (uvicorn, 2 workers) | 8000 |
| 前端 Web | Vue + nginx | 80 |
| 数据库 | PostgreSQL 15 | 5432（内网） |
| 缓存 | Redis 7 | 6379（内网） |

### 1.2 目标架构

```
                          ┌──────────────────────────────────┐
   用户浏览器 ───────────► │      华为云 ap-southeast-3         │
   http://<EIP>/           │  EIP <EIP>  (5Mbps BGP)           │
                          │        ↓                          │
                          │  ECS  s6.large.2 (2vCPU/4GB)      │
                          │  Ubuntu 22.04 + Docker Engine     │
                          │  ┌─────────────────────────────┐  │
                          │  │ docker compose               │  │
                          │  │  ├─ frontend  (nginx:80)     │  │
                          │  │  ├─ backend   (uvicorn:8000) │  │
                          │  │  ├─ postgres  (5432)         │  │
                          │  │  └─ redis     (6379)         │  │
                          │  └─────────────────────────────┘  │
                          │  VPC 10.0.0.0/16                  │
                          │  Subnet 10.0.0.0/24               │
                          │  SG: 22 / 80 / 8000 入站           │
                          └──────────────────────────────────┘
```

### 1.3 资源规划

| 资源 | 规格 |
|------|------|
| VPC | 10.0.0.0/16 |
| 子网 | 10.0.0.0/24 |
| 安全组 | 入站 22(SSH) / 80(HTTP) / 8000(API) |
| EIP | 5 Mbps，按流量计费 |
| ECS | s6.large.2（2vCPU/4GB），Ubuntu 22.04，系统盘 40GB GPSSD，按需 |

---

## 2. 前置条件

- 已完成 [guide.md](./guide.md) 中的全部配置：hcloud、Terraform、Skills、SSH 密钥。
- 一个华为云国际站账号（本示例区域 `ap-southeast-3`）。
- 待部署应用已具备 `docker-compose.yml`。

```bash
# 快速自检
hcloud STS GetCallerIdentity --cli-region=ap-southeast-3
terraform version
```

---

## 3. Step 1 本地 Docker 预验证

**先在本地跑通，再上云**，避免在云上才发现应用本身的问题。

```bash
cd AssetMgmt

# 启动数据库与后端
docker compose up -d --build backend

# 健康检查
curl http://localhost:8000/health
# 期望: {"status":"ok","version":"1.0.0"}

# 管理员登录
curl -X POST http://localhost:8000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin123456"}'
# 期望: {"access_token":"eyJ...","refresh_token":"eyJ...","token_type":"bearer"}
```

### 常见"上云前必改"问题

| 问题 | 解决方案 |
|------|---------|
| 数据库连接硬编码 `ssl=require`，本地 PG 无 SSL | 改为按环境变量 `DB_SSL_MODE` 控制 |
| 迁移链缺失基础表（alembic 无法建全） | 启动时用 `Base.metadata.create_all` 兜底建表 |
| 无初始化管理员账号 | 新增 `init_data.py`，启动时幂等创建 admin/角色/权限 |
| uvicorn 多 worker 并发建表冲突 | 用 PostgreSQL `pg_advisory_lock` 串行化建表与 seed |
| 依赖版本冲突（如 pytest） | 修正 `requirements.txt` 版本 |
| 前端 `vue-tsc` 与 TS 版本不兼容 | 构建脚本直接用 `vite build`，跳过类型检查 |

---

## 4. Step 2 获取临时凭证（IAM Agency）

hcloud 的 AK/SK 是**加密存储**的，`terraform` 需要明文凭证。做法：创建一个 IAM Agency，
用 `CreateTemporaryAccessKeyByAgency` 换取**临时 AK/SK + security_token**。

```bash
export REGION=ap-southeast-3

# 1) 获取账号 ID
ACCOUNT_ID=$(hcloud STS GetCallerIdentity --cli-region=$REGION \
  | python -c "import sys,json;print(json.load(sys.stdin)['account_id'])")

# 2) 创建自委托 Agency（委托给自己账号）
AGENCY_ID=$(hcloud IAM CreateAgency --cli-region=$REGION \
  --agency.name=assetmgmt-deploy-agency \
  --agency.domain_id=$ACCOUNT_ID \
  --agency.trust_domain_id=$ACCOUNT_ID \
  | python -c "import sys,json;print(json.load(sys.stdin)['agency']['id'])")

# 3) 绑定项目级权限（先拿到 project_id）
#    说明：ECS/VPC/EIP/EVS 属于项目级服务，必须绑定到 project
PROJECT_ID=019ecb32142b7a35b6a3ce72b177877d   # 可从控制台或 GetCallerIdentity 关联资源获取

# 角色 ID 可从 KeystoneListPermissions 查询后填入：
#   ECS FullAccess / VPC Administrator / EIP FullAccess / EVS FullAccess
hcloud IAM AssociateAgencyWithProjectPermission --cli-region=$REGION \
  --agency_id=$AGENCY_ID --project_id=$PROJECT_ID --role_id=<ECS_FullAccess_ID>
hcloud IAM AssociateAgencyWithProjectPermission --cli-region=$REGION \
  --agency_id=$AGENCY_ID --project_id=$PROJECT_ID --role_id=<VPC_Admin_ID>
hcloud IAM AssociateAgencyWithProjectPermission --cli-region=$REGION \
  --agency_id=$AGENCY_ID --project_id=$PROJECT_ID --role_id=<EIP_FullAccess_ID>
hcloud IAM AssociateAgencyWithProjectPermission --cli-region=$REGION \
  --agency_id=$AGENCY_ID --project_id=$PROJECT_ID --role_id=<EVS_FullAccess_ID>

# 4) 换取临时凭证（有效期 15min ~ 24h）
hcloud IAM CreateTemporaryAccessKeyByAgency --cli-region=$REGION \
  --auth.identity.assume_role.agency_name=assetmgmt-deploy-agency \
  --auth.identity.assume_role.domain_id=$ACCOUNT_ID \
  --auth.identity.assume_role.duration_seconds=3600 \
  --auth.identity.methods.1=assume_role
# 输出：credential.access / credential.secret / credential.securitytoken
```

> 把结果保存到环境变量（**同一终端**内后续步骤都要用）：

```bash
hcloud IAM CreateTemporaryAccessKeyByAgency --cli-region=$REGION \
  --auth.identity.assume_role.agency_name=assetmgmt-deploy-agency \
  --auth.identity.assume_role.domain_id=$ACCOUNT_ID \
  --auth.identity.assume_role.duration_seconds=3600 \
  --auth.identity.methods.1=assume_role > cred.json

export TF_VAR_access_key=$(python -c "import json;print(json.load(open('cred.json'))['credential']['access'])")
export TF_VAR_secret_key=$(python -c "import json;print(json.load(open('cred.json'))['credential']['secret'])")
export TF_VAR_security_token=$(python -c "import json;print(json.load(open('cred.json'))['credential']['securitytoken'])")
```

---

## 5. Step 3 Terraform 创建基础设施

### 5.1 目录结构

```
terraform/
├── main.tf        # provider + 资源定义
├── variables.tf   # 变量（AK/SK/token/region）
└── outputs.tf     # 输出（EIP/ECS ID/私网 IP）
```

### 5.2 provider 与变量

```hcl
# variables.tf
variable "access_key"     { type = string  sensitive = true }
variable "secret_key"     { type = string  sensitive = true }
variable "security_token" { type = string  sensitive = true  default = "" }
variable "region"         { type = string  default = "ap-southeast-3" }
variable "availability_zone" { type = string default = "ap-southeast-3a" }
variable "ecs_admin_pass" { type = string  sensitive = true  default = "AssetMgmt@2026" }
```

```hcl
# main.tf（片段）
terraform {
  required_providers {
    huaweicloud = { source = "huaweicloud/huaweicloud", version = "1.97.2" }
  }
}

provider "huaweicloud" {
  access_key     = var.access_key
  secret_key     = var.secret_key
  security_token = var.security_token
  region         = var.region
}
```

### 5.3 核心资源

```hcl
# VPC + 子网
resource "huaweicloud_vpc" "vpc" {
  name = "assetmgmt-vpc"
  cidr = "10.0.0.0/16"
}

resource "huaweicloud_vpc_subnet" "subnet" {
  name       = "assetmgmt-subnet"
  cidr       = "10.0.0.0/24"
  gateway_ip = "10.0.0.1"
  vpc_id     = huaweicloud_vpc.vpc.id
}

# 安全组 + 入站规则
resource "huaweicloud_networking_secgroup" "secgroup" {
  name        = "assetmgmt-secgroup"
  description = "Security group for AssetMgmt deployment"
}

resource "huaweicloud_networking_secgroup_rule" "ssh" {
  direction = "ingress"  ethertype = "IPv4"  protocol = "tcp"
  port_range_min = 22    port_range_max = 22  remote_ip_prefix = "0.0.0.0/0"
  security_group_id = huaweicloud_networking_secgroup.secgroup.id
}
# 同理再加 http(80) 与 backend(8000) 两条规则

# EIP
resource "huaweicloud_vpc_eip" "eip" {
  publicip  { type = "5_bgp" }
  bandwidth { name = "assetmgmt-bandwidth"  size = 5  share_type = "PER"  charge_mode = "traffic" }
}

# ECS（用 admin_pass，避免密钥对与临时凭证 user_id 不匹配）
resource "huaweicloud_compute_instance" "ecs" {
  name              = "assetmgmt-ecs"
  image_id          = "<Ubuntu 22.04 镜像 ID>"
  flavor_id         = "s6.large.2"
  availability_zone = var.availability_zone
  admin_pass        = var.ecs_admin_pass
  security_group_ids = [huaweicloud_networking_secgroup.secgroup.id]

  network { uuid = huaweicloud_vpc_subnet.subnet.id }

  system_disk_type = "GPSSD"
  system_disk_size = 40
}

# 绑定 EIP 到 ECS
resource "huaweicloud_vpc_eip_associate" "eip_assoc" {
  port_id   = huaweicloud_compute_instance.ecs.network[0].port
  public_ip = huaweicloud_vpc_eip.eip.address
}
```

### 5.4 执行

```bash
cd terraform
terraform init
terraform plan
terraform apply -auto-approve

# 记录输出
terraform output
# ecs_eip        = "x.x.x.x"
# ecs_id         = "...."
# ecs_private_ip = "10.0.0.x"
```

---

## 6. Step 4 部署应用到 ECS

### 6.1 SSH 登录 ECS

```bash
EIP=<terraform output ecs_eip>
PASS="AssetMgmt@2026"
ssh root@$EIP      # 输入密码
```

### 6.2 安装 Docker

```bash
apt-get update -qq
apt-get install -y -qq docker.io docker-compose-v2
systemctl start docker && systemctl enable docker
docker --version && docker compose version
```

### 6.3 上传代码

```bash
# 本地打包（排除大目录与本地配置）
tar czf app.tar.gz \
  --exclude='node_modules' --exclude='__pycache__' --exclude='.git' \
  --exclude='*.pyc' --exclude='.venv' --exclude='terraform' \
  backend frontend docker-compose.yml .env

# 上传（scp 或 SFTP）
scp app.tar.gz root@$EIP:/root/

# ECS 上解压
ssh root@$EIP "mkdir -p /opt/app && tar xzf /root/app.tar.gz -C /opt/app"
```

### 6.4 调整环境配置

```bash
# 把 CORS 的 localhost 换成真实 EIP
ssh root@$EIP "sed -i 's|CORS_ORIGINS=.*|CORS_ORIGINS=[\"http://$EIP\"]|' /opt/app/.env"
```

### 6.5 启动服务

```bash
ssh root@$EIP "cd /opt/app && docker compose up -d --build"
# 或分步：先后端，再前端
# docker compose up -d --build backend
# docker compose up -d --build frontend
```

---

## 7. Step 5 验证访问

```bash
EIP=<terraform output ecs_eip>

# 前端页面
curl -o /dev/null -w "%{http_code}\n" http://$EIP/           # 期望 200

# 后端健康检查
curl http://$EIP:8000/health
# 期望 {"status":"ok","version":"1.0.0"}

# 经 nginx 反代登录
curl -X POST http://$EIP/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin123456"}'
# 期望 {"access_token":"eyJ...","refresh_token":"eyJ...","token_type":"bearer"}
```

浏览器打开 `http://<EIP>/`，用 `admin / admin123456` 登录即部署成功。

---

## 8. Step 6 监控与账单

### 8.1 监控 ECS（CES）

对话方式（`huawei-cloud-ces-ecs-monitoring`）：

```
查看 ECS <instance_id> 最近 1 小时的 CPU 和内存使用率
```

### 8.2 查询账单（BSSINTL）

> ⚠️ 国际站账单必须用 `BSSINTL` 服务 + `ap-southeast-1` 区域。

```bash
hcloud BSSINTL ShowCustomerMonthlySum \
  --bill_cycle=2026-10 --cli-region=ap-southeast-1 --cli-output=json
```

对话方式（`huawei-cloud-billing-scout`）：

```
查询我本月华为云消费，并定位持续扣费的资源
```

---

## 9. Step 7 销毁与清理

**验证完成后务必清理，避免持续计费。**

```bash
# 1) 销毁基础设施（重新获取临时凭证）
export TF_VAR_access_key=<临时AK>
export TF_VAR_secret_key=<临时SK>
export TF_VAR_security_token=<临时token>
cd terraform && terraform destroy -auto-approve

# 2) 删除 IAM Agency
hcloud IAM DeleteAgency --cli-region=ap-southeast-3 --agency_id=<agency_id>

# 3) 删除 KPS 密钥对（如有）
hcloud KPS DeleteKeypair --cli-region=ap-southeast-3 --keypair_name=assetmgmt-keypair

# 4) 核验无残留
hcloud VPC ListVpcs --cli-region=ap-southeast-3          # 应无 assetmgmt-vpc
hcloud EIP ListPublicips --cli-region=ap-southeast-3     # 应无对应 EIP
hcloud ECS NovaListServers --cli-region=ap-southeast-3   # 应无 assetmgmt-ecs
```

---

## 10. 排错经验

| 现象 | 根因 | 解决 |
|------|------|------|
| `terraform plan` 报 `Authentication failed` | 临时凭证缺失/过期 | 重新获取并 `export TF_VAR_*`；临时凭证默认 15min 起 |
| 创建 ECS 报 `keypair does not match the user_id` | KPS 密钥对属原始用户，与临时凭证 Agency 身份不匹配 | 去掉 `key_pair`，用 `admin_pass` 创建 |
| `create_all` 报 `pg_type_typname_nsp_index` 冲突 | uvicorn 多 worker 并发建表 | 用 `pg_advisory_lock` 串行化建表+seed |
| `roles_name_key` 唯一约束冲突 | 多 worker 并发 seed | 同上，seed 也纳入同一把 advisory lock |
| 前端构建报 `Search string not found: supportedTSExtensions` | `vue-tsc` 与 TypeScript 版本不兼容 | 构建脚本改为 `vite build` |
| 页面能开但接口 404 | nginx 反代路径与后端前缀不一致 | 统一 `/api` 前缀与 `proxy_pass` 目标 |
| 浏览器跨域报错 | `CORS_ORIGINS` 仍是 localhost | 改为 `["http://<EIP>"]` |
| 账单查询报 `不支持的服务名称:BSS` | 国际站用错服务 | 改 `BSSINTL` + `--cli-region=ap-southeast-1` |
| 内存不足导致构建失败 | ECS 规格偏小（如 2GB） | 用 4GB 及以上规格，或本地构建后推镜像 |

---

> 返回配置说明：[guide.md](./guide.md)