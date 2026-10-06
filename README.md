# 福州福兴妇产医院 HIS 系统架构设计方案

## 项目背景

福州福兴妇产医院是民营二级妇产医院，HIS 系统支撑预约挂号、收费结算、医保接口、药房管理等业务。随着就诊业务增长，系统访问压力上升，出现数据库性能瓶颈，同时原有架构缺少容灾能力。

核心目标：
- 优化平台稳定性，保障诊疗业务不间断
- 满足医疗行业数据安全合规要求
- 构建高可用、可扩展、可审计的本地化 IT 平台

---

## 系统整体架构图

```mermaid
flowchart TB
    subgraph U[用户访问层]
        A[医生工作站]
        B[护士站]
        C[收费站]
        D[药房系统]
        E[医保接口]
        F[微信小程序挂号]
    end

    subgraph N[网络接入层]
        G[外网接入]
        H[内网接入]
        I[医保专线/VPN]
    end

    subgraph HA[高可用入口层]
        J[Keepalived VIP]
        K[Nginx LB 1]
        L[Nginx LB 2]
    end

    subgraph APP[应用服务层]
        M[Kubernetes Cluster]
        M1[Master 1]
        M2[Master 2]
        M3[Master 3]
        W1[Worker Node 1]
        W2[Worker Node 2]
        W3[Worker Node 3]
        SVC[HIS 业务服务]
        PAY[支付服务]
        INS[医保接口服务]
        EMR[电子病历服务]
    end

    subgraph VM[传统虚机业务层]
        V1[旧版 HIS VM]
        V2[LIS VM]
        V3[PACS VM]
    end

    subgraph DB[数据存储层]
        DB1[MySQL Master]
        DB2[MySQL Slave 1]
        DB3[MySQL Slave 2]
        DB4[MySQL Slave 3]
        R1[Redis Master]
        R2[Redis Slave 1]
        R3[Redis Slave 2]
        R4[Redis Slave 3]
        BAK[备份存储 NAS/OSS]
    end

    subgraph DEV[CI/CD与运维管理层]
        GL[GitLab]
        JN[Jenkins]
        HARBOR[Harbor]
        WH[Webhook]
        K8SDEP[K8s 滚动更新 / 回滚]
    end

    subgraph OBS[可观测性层]
        PROM[Prometheus]
        GRA[Grafana]
        ELK[ELK Stack]
        AM[AlertManager]
    end

    A --> G
    B --> H
    C --> H
    D --> H
    E --> I
    F --> G

    G --> J
    H --> J
    I --> J

    J --> K
    J --> L
    K --> M
    L --> M

    M --> M1
    M --> M2
    M --> M3
    M --> W1
    M --> W2
    M --> W3

    W1 --> SVC
    W2 --> PAY
    W3 --> INS
    W3 --> EMR

    SVC --> DB1
    PAY --> DB1
    INS --> DB1
    EMR --> DB1

    DB1 --> DB2
    DB1 --> DB3
    DB1 --> DB4
    SVC --> R1
    PAY --> R1
    INS --> R1
    EMR --> R1

    R1 --> R2
    R1 --> R3
    R1 --> R4

    DB1 --> BAK
    R1 --> BAK

    GL --> WH
    WH --> JN
    JN --> HARBOR
    JN --> K8SDEP
    K8SDEP --> M

    PROM --> GRA
    PROM --> AM
    ELK --> GRA
    K --> PROM
    L --> PROM
    W1 --> PROM
    W2 --> PROM
    W3 --> PROM
    DB1 --> PROM
    DB2 --> PROM
    R1 --> PROM
    SVC --> ELK
    PAY --> ELK
    INS --> ELK
    EMR --> ELK

    V1 --> J
    V2 --> J
    V3 --> J
```

---

## 架构分层说明

### 1. 用户访问层
- 医生工作站
- 护士站
- 收费站
- 药房系统
- 医保接口
- 微信小程序挂号

### 2. 网络接入层
- 外网接入：面向患者和医务人员的互联网入口
- 内网接入：院内局域网访问
- 医保专线 / VPN：实现与医保平台安全互联

### 3. 高可用入口层
- Keepalived：保证 VIP 自动漂移，支持主备切换
- Nginx：负责流量分发、静态资源处理、反向代理
- 采用双机热备模式，降低单点故障风险

### 4. 应用服务层
- 基于 Kubernetes 容器化部署服务组件
- 业务模块包括：
  - HIS 业务服务
  - 支付服务
  - 医保接口服务
  - 电子病历服务
- 传统虚拟机环境继续承载遗留系统（LIS、PACS、旧版 HIS）

### 5. 数据存储层
- MySQL 主从复制架构
  - Master：负责写入和关键事务处理
  - Slave：负责读流量分担、报表查询和备份
- Redis 作为缓存层
  - Session 缓存
  - 热点数据缓存
  - 访问加速
- 采用定时物理备份和异地数据存储方案

### 6. CI/CD 与运维管理层
- GitLab：代码托管与版本管理
- Jenkins：持续集成与自动化发布
- Harbor：企业级私有镜像仓库
- Webhook：实现代码提交后自动构建镜像
- Kubernetes：实现滚动更新、灰度发布和回滚

### 7. 可观测性层
- Prometheus：采集基础设施和应用指标
- Grafana：实时监控大屏和告警展示
- ELK：日志收集、聚合和分析
- AlertManager：告警规则与推送

---

## 核心设计原则

1. 高可用：采用主备切换、故障转移和自动恢复机制。
2. 稳定性：提升系统吞吐、优化数据库与缓存性能。
3. 安全合规：对医疗数据访问与审计进行集中控制。
4. 自动化：通过 CI/CD 和监控平台减少人工干预。
5. 可扩展：支持业务增长及后续融合更多医院系统。

---

## 适用场景

- 挂号业务高峰期访问优化
- 缴费结算与医保接口的稳定性提升
- 电子病历和处方流转的性能保障
- 数据恢复、容灾验证和系统演练

---

## 结论

该架构结合了本地化高可用入口、容器化应用部署、MySQL/Redis 数据层优化以及 Prometheus + ELK 全链路监控能力，能够在保障医疗业务连续性与数据安全的同时，提升系统可扩展性和运维效率。

---

## 文档结构

- [高可用架构设计](docs/ha-architecture.md)
- [容器与 Kubernetes 方案](docs/k8s-deployment.md)
- [CI/CD 流水线](docs/cicd-pipeline.md)
- [监控与日志体系](docs/monitoring-logging.md)
- [数据库与缓存设计](docs/database-cache.md)
- [备份与灾难恢复](docs/backup-recovery.md)
- [运维 SOP](docs/operation-sop.md)

---

更新时间：2026-10-06
