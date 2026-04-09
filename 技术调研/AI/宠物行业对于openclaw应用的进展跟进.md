---
tags:
  - 调研
  - AI
  - 宠物行业
  - OpenClaw
  - 智能硬件
date: 2026-04-09
updated: 2026-04-09
status: 持续跟进
---

# 宠物行业对 OpenClaw 应用的进展跟进

## 背景

1. 定期调研宠物行业对 OpenClaw 的使用情况
2. 包含但不限于宠物平台服务、宠物智能硬件、宠物 App 等
3. 调研和宠物相关的 Skill

---

## 一、已确认的实际接入案例

### 1.1 AI宠宝（aipetbao）—— 宠物健康问答平台 × OpenClaw

| 项目       | 详情                                                |
| -------- | ------------------------------------------------- |
| **产品名**  | AI宠宝（aipetbao）                                    |
| **官网**   | [aipetbao.cn](https://aipetbao.cn/)               |
| **功能入口** | aipetbao.cn/xlkc（"龙虾养宠"功能）                        |
| **定位**   | AI 宠物健康全周期平台                                      |
| **技术底座** | **OpenClaw**（智能交互框架）+ **火山知识库**（专业垂类内容）+ AI宠宝自有数据 |
| **发布时间** | 2026年3月                                           |
|          |                                                   |

**具体功能：**

- **多轮对话问诊**：用户用日常语言提问，系统主动追问品种、症状持续时间等关键信息，提供精准解答
- **持久记忆 / 宠物画像**：长期存储宠物的品种、年龄、健康状况及饲养习惯，实现"千人千宠"个性化服务
- **定时提醒**：自主执行疫苗、驱虫等时间规划提醒
- **品种专属知识**：如折耳猫骨骼护理、法斗呼吸道养护等品种特定内容
- **与自有系统联动**：与 AI 宠宝已有的**宠物皮肤健康智能评估系统**、**智能体测评分系统**数据互通（如结合肥胖体测结果定制减脂方案）
- **临床标准**：皮肤问题解答遵循 ISVD、ACVD 国际诊疗标准
- **易感性数据库**：内置常见犬猫品种的皮肤、消化系统疾病等易感性数据，结合年龄体况做风险预判

> 来源：[掘金文章](https://juejin.cn/post/7615075978017570851)

---

### 1.2 VetClaw —— 开源兽医 AI 技能库（面向 OpenClaw）

| 项目 | 详情 |
|------|------|
| **项目名** | VetClaw |
| **开发方** | OpenVet |
| **定位** | 全球首个开源兽医 AI 技能库，专为 OpenClaw 平台设计 |
| **GitHub** | [OpenVet-Projects/VetClaw](https://github.com/OpenVet-Projects/VetClaw) |
| **官网** | [openvet.ai/resources](https://openvet.ai/resources) |
| **技能数量** | **51 个 AI Skills** |
| **开放方式** | 免费开源，面向研究人员、开发者和兽医专业人员 |

**具体功能方向：**

- 物种感知（Species-Aware）临床工作流 —— 根据不同动物物种提供定制化诊疗流程
- 药物安全（Drug Safety）—— 兽药剂量计算与安全检查
- 临床决策支持（Clinical Decision Support）
- 安全优先（Safety-First）设计，内置临床安全边界

> 来源：[Enterprise News 新闻稿](https://www.enterprisenews.com/press-release/story/90672/)、[LinkedIn 公告](https://www.linkedin.com/posts/sager_github-openvet-projectsvetclaw-the-first-activity-7440080084495040513-bYvb)

---

### 1.3 某宠物零食品牌（公开报道匿名案例）

- **来源**：[OpenClaw 在宠物零食行业的应用场景 - 博客园](https://www.cnblogs.com/sunzhenyong/p/19767746)
- **应用场景**：
  - 监控原材料价格波动 → 自动下单补货
  - 跟踪物流状态 → 协调供应商沟通
  - 智能客服（产品成分咨询、喂食建议、宠物过敏问题）
- **效果**：
  - 客服响应时间：平均 2 小时 → **15 分钟**
  - 电商运营效率提升 **60%**
  - 每月节省人力成本约 **2 万元**

### 1.4 某全国知名宠物用品超市（网易易盾案例）

- **来源**：网易副总裁阮良在企业级 AI 应用分析中提及 → [新浪财经](https://t.cj.sina.cn/articles/view/6522187104/184c0ad60001015s0m)
- **背景**：同步在线下门店和线上外卖平台销售，覆盖区域广、商品种类多
- **应用**：部署 OpenClaw 类 AI 智能体解决多平台商品管理与客服自动化问题
- **效果**：运营效率大幅提升（具体数值未披露）

### 1.5 飞书 × 宠物智能设备（场景示例）

- **来源**：[飞书官网 - OpenClaw 接入飞书后能做什么](https://www.feishu.cn/content/article/7618097619889343446)
- **具体能力**（通过 OpenClaw 接入宠物智能设备如智能项圈）：
  - 24小时监控宠物心率、体温、活动量
  - 异常数据自动预警
  - 生成健康周报，发现潜在问题
  - 提供个性化喂养建议
- **注意**：这是飞书官方给出的功能场景示例，非特定企业合作案例

### 1.6 OpenClaw Mobile 宠物护理自动化

- **来源**：[OpenClaw Mobile 宠物护理自动化指南](https://openclawmobile.ai/zh/blog/openclawmobile-pet-care-automation-guide-zh)
- **具体自动化场景**：
  - 每月1号凌晨：自动在宠物店下单猫粮和猫砂（使用存储的优惠券）
  - 每季度疫苗前一周：自动预约最近的宠物医院，把时间加到日历
  - 每三个月：驱虫药提醒

---

## 二、ClawHub 上面向真实宠物护理的 Skill

### 2.1 `dog` —— 狗狗全方位护理

| 项目 | 详情 |
|------|------|
| **作者** | `ivangdavila` |
| **ClawHub** | [clawhub.ai/ivangdavila/dog](https://clawhub.ai/ivangdavila/dog) |
| **安全评级** | Benign（高可信度），零外部网络请求 |
| **数据存储** | 全部本地 `~/dog/` |

**具体功能：**

- **紧急分诊（triage）**：遇到呼吸困难、癫痫、中毒、大出血等立即切换急救指导
- **每只狗独立档案**：profile / health / routines / behavior / training / logistics / timeline
- **遛狗记录** + **训练跟踪** + **行为分析**
- **旅行/寄养准备**：生成 sitter-pack（看护人交接文档）
- **购物补给**：自动追踪消耗品补货阈值
- **兽医协调**：就诊准备、随访问题清单
- 明确不做：不诊断疾病、不推荐药物剂量、不使用惩罚性训练

### 2.2 `pet-companion-journal` —— 宠物伴侣日记

| 项目 | 详情 |
|------|------|
| **作者** | `skills`（官方） |
| **ClawHub** | [clawhub.ai/skills/pet-companion-journal](https://clawhub.ai/skills/pet-companion-journal) |
| **数据存储** | `~/.pet-companion/` |

**具体功能：**

- **宠物档案**：名字、物种、品种、生日、绝育状态、性格标签
- **6种记录类型**：daily（日常）/ moment（成长瞬间）/ photo（照片+说明）/ feeding（喂食变化）/ health（症状、就诊、用药、疫苗、驱虫）/ reminder-note
- **提醒管理**：疫苗、驱虫、体检、美容、用药、复诊、生日
- **结构化查询**：按宠物、记录类型、时间范围、关键词过滤
- **导出报告**：`export_report.py` 可生成时间段摘要

### 2.3 `Homeclaw` —— 家庭管家（含宠物照顾模块）

| 项目 | 详情 |
|------|------|
| **来源** | [FindSkills](https://findskills.org/zh/skills/clawhub-homeclaw) |
| **更新时间** | 2026-03-10 |
| **安全级别** | 社区 |
| **功能** | 家庭模式智能管家，宠物照顾是其功能模块之一（健康监测、日程管理、家居控制等） |

---

## 三、未找到 OpenClaw 接入的具体宠物公司

以下品牌经搜索**未发现**与 OpenClaw 有公开合作/接入：

| 品牌/公司 | 业务 | 结果 |
|-----------|------|------|
| 波奇网 | 宠物社区电商 | ❌ |
| 小佩 Petkit | 宠物智能硬件 | ❌ |
| CATLINK | 智能猫砂盆 | ❌ |
| 萌爪医生 | 在线宠物医疗 | ❌ |
| 瑞鹏宠物医院 | 线下宠物医疗 | ❌ （有自己的 Vet1 大模型，但非基于 OpenClaw） |
| 一宠健康/萌邦AI | 宠物AI医疗 | ❌ （有自己的 petAIvet-R1，非基于 OpenClaw） |
| 迪亚智能 Vetidia | 宠物医疗AI | ❌ （VetiMed 是独立产品，非基于 OpenClaw） |

**结论**：截至 2026年4月，OpenClaw 在宠物行业的实际落地案例仍非常少。目前唯一有具体产品名和功能细节的面向 C 端宠物主的产品是 **AI宠宝（aipetbao）**，B 端有 **VetClaw** 开源兽医技能库和两个匿名企业案例。传统宠物公司（食品、用品、连锁医院）尚未出现公开的 OpenClaw 接入案例。

---

## 四、持续关注清单

- [ ] AI宠宝（aipetbao）用户增长与功能迭代（官网 aipetbao.cn）
- [ ] VetClaw GitHub 仓库更新——51个技能的具体列表与实际落地医院
- [ ] 波奇网、小佩、CATLINK 等头部宠物公司是否公开接入 OpenClaw
- [ ] ClawHub `/skills/pets` 分类页新增 Skill
- [ ] 宠物医疗 AI 赛道中是否有 OpenClaw 接入案例（区别于自有大模型的独立产品）
- [ ] OpenClaw 宠物护理自动化（OpenClaw Mobile）的实际用户反馈

---

## 参考来源

- [AI宠宝 × OpenClaw - 掘金](https://juejin.cn/post/7615075978017570851)
- [AI宠宝官网](https://aipetbao.cn/)
- [VetClaw 新闻稿 - Enterprise News](https://www.enterprisenews.com/press-release/story/90672/)
- [VetClaw LinkedIn](https://www.linkedin.com/posts/sager_github-openvet-projectsvetclaw-the-first-activity-7440080084495040513-bYvb)
- [VetClaw GitHub](https://github.com/OpenVet-Projects/VetClaw)
- [宠物零食行业应用 - 博客园](https://www.cnblogs.com/sunzhenyong/p/19767746)
- [宠物用品超市案例 - 新浪财经](https://t.cj.sina.cn/articles/view/6522187104/184c0ad60001015s0m)
- [飞书 × OpenClaw 宠物设备示例](https://www.feishu.cn/content/article/7618097619889343446)
- [OpenClaw Mobile 宠物护理指南](https://openclawmobile.ai/zh/blog/openclawmobile-pet-care-automation-guide-zh)
- [Dog Skill - ClawHub](https://clawhub.ai/ivangdavila/dog)
- [Pet Companion Journal - ClawHub](https://clawhub.ai/skills/pet-companion-journal)
- [Homeclaw - FindSkills](https://findskills.org/zh/skills/clawhub-homeclaw)
- [从OpenClaw到宠物门店数字化想象](https://www.industrysourcing.cn/article/474542)
