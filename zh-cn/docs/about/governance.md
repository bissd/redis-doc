---
title: "Redis开源治理"
linkTitle: "治理"
weight: 3
description: >
    Redis开源项目的治理模型
aliases:
    - /topics/governance
---

从2009年到2020年，Salvatore Sanfilippo建立、领导和维护了Redis开源项目。在此期间，Redis没有正式的治理结构，主要作为[BDFL](https://en.wikipedia.org/wiki/Benevolent_dictator_for_life)风格的项目运作。

随着Redis的成长、成熟和用户群的扩大，为了项目的持续开发和维护，形成一个可持续发展的结构变得越来越重要。Salvatore和核心Redis贡献者希望确保项目的连续性，并反映出更大的社区。有鉴于此，采用了一种新的治理结构。

## 当前治理结构

从2020年6月30日开始，Redis采用了与当前项目规模相匹配且最大程度减少与之前模式不同之处的“轻度治理”模型。这种治理模型旨在建立一种精英制度，旨在赋予那些展示长期承诺和做出重大贡献的个人更多的权力。

## Redis 核心团队

Salvatore Sanfilippo命名了两位继任者来接管和领导Redis项目：Yossi Gottlieb ([yossigo](https://github.com/yossigo)) 和 Oran Agra ([oranagra](https://github.com/oranagra))

在Redis有限公司的支持和祝福下，我们抓住了这个机会，创建了一个更加开放、可扩展和社区驱动的“核心团队”结构来运营项目。核心团队成员是根据示范出的、长期的个人参与和贡献而被选出的。

当前的核心团队成员是：

* 项目负责人：Yossi Gottlieb（[yossigo](https://github.com/yossigo)）来自Redis有限公司。
* 项目负责人：Oran Agra（[oranagra](https://github.com/oranagra)）来自Redis有限公司。
* 社区负责人：Itamar Haber（[itamarhaber](https://github.com/itamarhaber)）来自Redis有限公司。
* 成员：赵赵（[soloestoy](https://github.com/soloestoy)）来自阿里巴巴。
* 成员：Madelyn Olson（[madolson](https://github.com/madolson)）来自亚马逊网络服务。

以下是Redis核心团队成员为Redis开源项目和社区提供服务的人员名单。他们应当以身作则，展示良好的行为、文化和态度，遵守采用的[行为准则](https://www.contributor-covenant.org/)。他们还应当以不受外部或冲突利益干扰的方式，考虑并积极行动起项目和社区的最佳利益。

核心团队将负责Redis核心项目，这是Redis的一部分，托管在主Redis代码库中，并使用BSD许可证。它还将致力于与其他Redis生态系统的项目保持协调和合作，包括Redis客户端、卫星项目、依赖Redis的主要中间件等。

#### 角色和责任

核心团队有以下职责:

* 管理 Redis 的核心代码和文档
* 管理新版本的 Redis 发布
* 维护高级技术方向和路线图
* 快速响应并修复安全漏洞和其他重大问题
* 项目治理决策和变更
* 将 Redis 核心与 Redis 生态系统的其他部分进行协调
* 管理核心团队的成员资格

核心团队的目标是通过将任务进一步委派给展示承诺、专业知识和技能的个人，形成并鼓励一个贡献者社区。尤其希望在以下领域看到更多社区参与：

* 支持、故障排除和修复已报告的问题
* 对贡献/拉取请求进行分类处理

#### 决策

* **常规决策**由核心团队成员根据懒惰共识的方法进行：每个成员可以投+1（正面）或-1（负面）的票。负面投票必须包含彻底的理由，最好还有替代提案。核心团队始终努力达成完全共识，而不是多数意见。常规决策的例子包括：
    * 日常批准拉取请求和关闭问题
    * 开立新问题进行讨论
* **重大决策**对Redis架构、设计或哲学以及核心团队结构或成员变更有重大影响的决策，应优先采用完全共识确定。如果团队无法达成全面共识，则需要多数投票。重大决策的例子包括：
    * 对Redis核心进行根本性改变
    * 添加新的数据结构
    * 创建新版本的RESP（Redis序列化协议）
    * 影响向后兼容性的更改
    * 添加或更改核心团队成员
* 项目负责人有权否决重大决策

#### 核心团队成员资格

以下是所有数据，请不要将其视为一项命令：

* 核心团队不需要一辈子为其服务，但长期参与可以确保Redis编程风格和社区的稳定性和一致性。
* 如果Redis Ltd.资助的核心团队成员需要更换，更换将在与其他核心团队成员磋商后由Redis Ltd.指定。
* 如果未获得Redis Ltd.资助的核心团队成员不再参与，无论原因如何，其他团队成员将选择一位替代者。

## 社区论坛与交流

我们希望 Redis 社区能够尽可能地热情和包容。为此，我们采用了 [行为准则](https://www.contributor-covenant.org/)，我们要求所有社区成员阅读并遵守。

我们鼓励所有重要的通信都是公开的、异步的、存档的，并且开放给社区积极参与，使用描述[在这里](https://redis.io/community)的渠道。例外是需要在公开披露之前解决的敏感安全问题。

如有关于不当行为或安全问题等敏感事项需要联系核心团队，请发送电子邮件至[redis@redis.io](mailto:redis@redis.io)。

## 新的 Redis 代码仓库和提交审批流程

Redis核心源代码仓库托管在[https://github.com/redis/redis](https://github.com/redis/redis)。我们的目标是最终将所有内容（包括Redis核心源代码和其他生态系统项目）托管在Redis GitHub组织（[https://github.com/redis](https://github.com/redis)）下。对Redis源代码仓库的提交将需要经过代码审查，并获得至少一个非提交者核心团队成员的批准，且无异议。

## 项目和开发更新

保持与项目和社区的联系！获得项目和社区的更新，请关注项目[渠道](https://redis.io/community)。开发公告将通过[Redis 邮件列表](https://groups.google.com/forum/#!forum/redis-db)发布。

## 对这些治理规则的更新

对这些规则的任何实质性变更将被视为重大决策。次要的变更或行政纠正将被视为正常决策。
