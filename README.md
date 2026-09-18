# search-xiaohongshu

一个用于检索小红书（Xiaohongshu / RedNote / XHS）笔记的 Codex Skill。它接收当前 CUA 官方运行时提供的已登录页面能力，通过页面自身的搜索模块获取结果，不在 Skill 内处理浏览器连接、选择或切换。

## 能做什么

- 按关键词搜索小红书笔记
- 使用综合、最新或最热排序
- 筛选图文或视频笔记
- 返回标题、作者、互动量、图片和笔记链接
- 按需读取少量笔记正文并进行汇总

## 使用条件

- Codex 桌面版
- 当前 CUA 官方运行时能够提供小红书页面能力
- 页面已经登录小红书，或能够由用户完成登录

## 安装

```bash
git clone https://github.com/BigDoggle/search-xiaohongshu.git \
  ~/.codex/skills/search-xiaohongshu
```

安装后重启 Codex，或新建一个任务，让 Skill 出现在可用技能列表中。

## 使用示例

```text
搜索小红书中关于计算机秋招简历的内容。
```

```text
查找最近的小红书北京租房经验，读取前三篇并总结注意事项。
```

## 说明

- 默认返回 10 条结果，最多返回 40 条。
- 需要汇总正文时默认读取 3 篇，最多读取 5 篇。
- 不读取、导出或保存浏览器 Cookie、密码和登录令牌。
- 不绕过登录、验证码、访问频率限制或内容权限。
- 小红书页面升级后，搜索模块和正文选择器可能需要重新适配。

具体执行规则见 [SKILL.md](SKILL.md)，接口实现见 [references/xiaohongshu-api.md](references/xiaohongshu-api.md)。
