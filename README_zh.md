# dsh-superagent

> **仅供研究参考的原型，不是 DeepSeek 的正式产品。** 这份代码长自我们团队的日常开发，其中大量工程由编程智能体完成。请把它当作研究代码，而不是可以上线的软件。它仍可能有缺陷，编码风格和工程做法也可能前后不一致。拿到你自己的环境里使用之前，先审阅，再按需要改。
>
> 为了公开发表，我们做过改编，所以它和内部版本不同，也和 *Exploring Super-agent Design: Towards a More Autonomous Agent System* 一文里描述的实现不同。文章里讨论的一些设计是以后的方向，这个原型里没有做。

dsh-superagent 是面向单个 GitHub 仓库的主动式编程智能体，打包成 [DeepSeek Harness](https://www.deepseek.com/en/harness/) 的一个插件包。它监听 GitHub，也可以监听飞书。允许名单上的人向智能体要一件工作时，它会开一张工单，在自己的智能体会话和目录里做，再回到原来的对话里答复，例如交回一个拉取请求。网页面板把需要人处理的事项摊开来。

它也是一种和智能体共事的方式：**尽可能少占用人的注意力，尽可能多给出价值。** 一个能替整个团队行动的智能体，也能浪费整个团队的注意力。所以它只在被人点名要求时行动，或在团队写成文字的规则允许时行动；只在必须有人动手、或有人用得上结果时才开口。[000 原则](docs/000-philosophy_zh.md) 说明这条原则，以及它要求操作者承担什么。

## 安装

安装的步骤很多，所以由智能体陪你做。请你的编程智能体：

> 阅读这个仓库里的 SETUP_zh.md，和我一起把 dsh-superagent 安装配置好。

它一条消息只问一个短问题，把完整计划展示一次，然后再安装、再配置。[安装配置](SETUP_zh.md) 就是它要遵循的那一份说明。

## 文档

按编号顺序读。每一篇只建立在它前面的篇章上。

- [000 原则](docs/000-philosophy_zh.md)
- [001 领域模型](docs/001-domain_zh.md)
- [002 授权](docs/002-authorization_zh.md)
- [003 执行](docs/003-execution_zh.md)
- [004 提示词](docs/004-prompts_zh.md)
- [005 面板](docs/005-panel_zh.md)
- [006 部署](docs/006-deployment_zh.md)：手工安装、配置、升级和排障
- [007 架构](docs/007-architecture_zh.md)
- [008 手册](docs/008-cookbook_zh.md)：增加一个来源

## 开发

`pnpm check` 会依次做类型检查、测试和构建。

## 许可

[MIT](LICENSE)
