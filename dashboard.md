---
dashboard: true
banner:
  quote: "The mind is everything. What you think you become."
  author: "Buddha"
  image: "https://images.pexels.com/photos/2307638/pexels-photo-2307638.jpeg"
  images:
    - "https://images.pexels.com/photos/2307638/pexels-photo-2307638.jpeg"
    - "https://images.pexels.com/photos/10664504/pexels-photo-10664504.jpeg"
columns:
  - name: Projects
    color: "#10b981"
    type: projects
  - name: Todo
    color: "#6366f1"
    type: todo
    height: 500
  - name: Memo
    color: "#f59e0b"
    type: memo
    height: 400
---

## Projects

### 我的项目与面经补充
id: card-mtjeighg
type: project
- [[智能客服与Agent面试问答.md]]
- [[面经补充.md]]
- [[对话请求完整链路.md]]
- [[实习介绍.md]]

## Todo

### 待办清单
id: card-mtmd1xgt
type: task
width: 364
- [ ] agent效果评测怎么做的
- [ ] agent项目中rag检索中RRF有什么参数，怎么设置的，为什么这么设置
- [ ] 个性化推荐为什么每次返回相同店铺？
- [x] agent项目中guardrail黑名单能不能自动从badcase中整理
- [x] 长期记忆中如果用户感冒以后说自己吃不了辣然后把不能吃辣存入了长期记忆，但是用户实际喜欢吃辣。这种情况怎么解决

### 待办清单
id: card-mtqledwx
type: task
width: 291
- [ ] 还没有发生OOM但是内存不断上涨，怎么排查不影响线上
- [ ] 看一下这些java开发中常见的问题的排查思路
- [ ] 自动化测试平台的线程池的参数是怎么设置的
- [ ] 接口响应很慢怎么排查
- [ ] 需求流转平台中的状态机是怎么实现不同分支分开执行的
- [ ] 聊不聊云平台部署给容器切换镜像的时候怎么让服务不中断
- [ ] 秒杀场景中在生产者端如果写本地消息表失败了，需要回滚redis但是redis回滚失败了怎么办
- [ ] claudecode，codex等智能体的上下文压缩方式有什么不同
- [ ] Redis的集群，主从同步的实现原理
- [x] codetop刷10道题
- [x] mysql的MVCC实现原理
- [x] JWT传输信息的话因为加密方式是明文传输，如果被破解了怎么办？别人能够伪造数据更改别人的数据，怎么防护？
- [x] http1.0,2,3的实现，TLS加密过程

### 待办清单
id: card-mu80t4dz
type: task
width: 273
- [ ] nginx，tomcat是怎么实现那么多请求都传来也不会崩溃的
- [ ] 能不能不用AOP也不用拦截器也不用spring框架中的机制来实现对请求的参数校验吗？
- [ ] 高并发场景中能不能不用线程池来实现异步并行操作，虚拟线程和内核线程是绑定的关系吗？
- [ ] 削峰填谷的常见方法
- [ ] 线程池怎么创建，怎么包装任务，怎么提交，怎么取值
- [ ] 一亿个数据导出，保证速度和稳定性，怎么做

## Memo

### 2026-09-07 备忘录
id: card-mtqqc1t9
agent的长期记忆怎么做，入库数据中必须存储这条记录的元数据（记忆来源、记录的时间、作用范围、置信度）
还要有记忆维护机制（更新、合并、删除）
