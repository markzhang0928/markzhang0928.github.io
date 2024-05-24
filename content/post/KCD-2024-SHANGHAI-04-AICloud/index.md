---
title: KCD 2024 SHANGHAI - 04 AICloud
summary: 演讲者主要介绍了AI gateway的特性, 给出了Kong GatewayOperator项目的一页PPT介绍。
date: 2024-05-24

authors:
  - admin

tags:
  - LLM
  - Kong AI Gateway
  - Kong Gateway Operator
---

## 用 Operator 将 LLM 和 Gateway 结合的更容易 - 张晋涛，Kong Inc.

**总结:** 
- 演讲者先介绍了调用 LLM API 的时候需要注意的问题，包括安全性、成本、可靠性、可观测性与配额等多方面，并以此引入了 AI Gateway 的概念。
- AI Gateway通过插件的方式，集成了多种在线和私有化部署的模型，进行认证授权密钥的管理，以及可观测服务的集成，甚至无须编码将LLM应用与后端服务集成。
- 此外，他还介绍了Kong Gateway Operator，通过 CRD 以及 Kubernetes 的 Gateway API 以更加便利的形式管理 AI Gateway 的资源，更好地与 Kubernetes 的生态系统集成。

### 为什么LLMs 需要 Gateway

#### 基本开发场景
- 1. 调用 LLM的API
- 2. 传递参数/prompt
- 3. 处理响应结果

#### 调用 LLM的API需要注意什么
```
- 1. 安全性
     - 1.1 认证信息API Key的管理
     - 1.2 请求内容的安全性/合规性
- 2. 成本
     - 2.1 缓存请求响应(可选)
- 3. 可靠性
     - 3.1 供应商API的SLA
- 4. 可观测性
     - Requests
     - Tokens/Cost
- 5. 限制与配额
     - RPM & RPD
     - TPM & TPD
- 6. 版本兼容性
     - API/模型版本升级等
- 7. 错误处理
     - 异常重试
     - Fallback Response
- 8. 依赖管理
     - 多种备用模型
     - 避免单一依赖
```

### AI Gateway的能力

{{< figure src="ai_proxy.png" caption="ai_proxy" theme="light" >}}

#### 认证和授权

* LLM API 的密钥可通过AI Gateway统一管理
     - Environment variables
     - AWS Secrets Manager
     - GCP Secrets Manager
     - Azure Key Vaults
     - HashiCorp Vault

* 可使用Kong AI Gateway原生的多种认证插件进行集成

#### 无须编码即可将LLM应用到原有API调用流程中
- 1. AI Response Transformer: 设置prompt直接处理响应
- 2. AI Response Transformer: 设置prompt直接处理请求

{{< figure src="flow_chart.png"  theme="light" >}}

#### AI Prompt Guard – prompt 防火墙
- 通过 Allow Patterns/Deny Patterns进行特定范围 prompt的管理
- 防止一些违规或者不安全的内容
- AI Prompt Decorator
  - 统一规则设置
  - 用于为所有请求设置统一的prompt规则
  - 合规
  - 排除敏感信息
  - ...

{{% callout note %}}
以上主要介绍了AI Gateway的一些特性
{{% /callout %}}

### Kong Gateway Operator带来的便利

全托管的Gateway
- 通过申明式配置，直接定义
- 创建gateway资源，自动管理

{{< figure src="kgo_config_1.png"  theme="light" >}}
{{< figure src="kgo_config_2.png"  theme="light" >}}

- 通过AIGateway进行第一类支持
  - 直接申明所需配置，自动部署 Gateway 并配置相关 plugin
  - 减少运维/维护成本
- 弹性伸缩
- AIGateway的生命周期管理
  - https://github.com/Kong/gateway-operator/

### 发展方向
- Kong Gateway Operator：通过 configRef 等机制，复用配置/屏蔽一些配置字段
- AIGateway：基于 Token 的计费和限制
- 添加更多 LLM 的集成