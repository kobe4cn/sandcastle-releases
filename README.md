# sandcastle-pipeline releases

Sandcastle 流水线的发布包。源码不公开，这里只放 Release。

## 安装

需要 Node.js ≥ 22.18。

```bash
npm i -g https://github.com/kobe4cn/sandcastle-releases/releases/latest/download/sandcastle-pipeline.tgz
```

装指定版本：把 `latest/download` 换成 `download/v0.1.0`。

## 使用

```bash
cd <你的项目>
sandcastle-pipeline onboard
sandcastle-pipeline doctor --fix
sandcastle-pipeline baseline
```

升级：`sandcastle-pipeline upgrade`，然后在各项目里跑 `sandcastle-pipeline update`。

完整说明在安装包里的 README.md，版本变化见每个 Release 的说明。
