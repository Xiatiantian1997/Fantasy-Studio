# 2D ⇄ 3D 及总控集成协议

> 当前为接口与调度设计。**总控尚没有可运行的本地服务；3D 导出未实施；2D 生成接口已写代码但 GPU 实测未完成。**

## 设计原则：中央编排、各自执行

```text
用户在总控 UI 选择角色 / 画风 / 姿势
    |
    +--- 主控模块：确认素材权限、任务 ID、路径和模块状态
    |           |
    |           +--> 3D 模块：在 Blender 中加载人体/姿势/相机
    |           |       产生控制包及 manifest.json
    |           |
    |           +--> 2D 模块：加载风格配方与授权素材
    |                   未来从控制包建立 ControlNet 工作流
    |                   交 ComfyUI 生成并返回结果
    |
    +--- 主控：展示生成结果，保存关联的角色/配方/姿势与版本
```

### 现有 2D 能力（核对当前源代码）

2D 开发服务当前使用本地环回地址 `127.0.0.1:7865`，ComfyUI 使用 `127.0.0.1:8190`：

- `GET /api/status`：检查生成后端是否可连接
- `GET /api/models`：可用 checkpoint 与 LoRA
- `POST /api/upload`：传入图像作为本机参考
- `POST /api/generate`：构造基础文生图或整图 img2img 的节点图
- `GET /api/job?id=<id>`：返回任务状态和图片引用
- `POST /api/interrupt`：停止生成请求

这些是 2D 现有服务的**开发接口**；总控尚未调用。接口将来如改动应记录兼容方式，不能直接让两个项目引用彼此 Python 内部函数。

### 3D 数据包（由 3D 仓库单独维护）

3D 模块负责输出一个目录：

```text
exports/<export_id>/
├── manifest.json
├── rgb.png
├── depth.exr
├── normal.exr
├── pose-internal.json
└── pose-preview.png
```

唯一规范来源：[3D interface-v0.1.md](https://github.com/Xiatiantian1997/Fantasy-Studio-3D/blob/main/docs/interface-v0.1.md) 和 [export-manifest.schema.json](https://github.com/Xiatiantian1997/Fantasy-Studio-3D/blob/main/schemas/export-manifest.schema.json)。

本总控做读取、路径安全检查、版本匹配、模块启动和任务状态管理，**不在这里维护第二份冲突 schema**。RGB、深度、法线和姿势必须同机位、同分辨率和同一帧；各 ControlNet 所需的编码转换由 2D 的适配层承担。

### 与本地素材的关系

统一库只保存资源索引和权限，不强制重复复制同一份图。推荐所有资源拥有稳定 `asset_id`，由本地索引解析到允许访问的图片路径（包含原图与已筛选派生图）。

- `public_reference`：可由普通创作工作流引用的资料。
- `curated_ai`：经筛选、允许 AI 创作模块读取。
- `private_user`：只在用户手动选择及符合适用规则的操作中访问，不授予默认代理。

这只是未来索引的命名草案，具体数据库尚未落地。云端 GitHub 不存真实私有索引文件。

### 失败与日志

当 3D 不存在、未安装或结构包不合格时，**2D 仍可独立生成**；主控不能因 3D 故障让基础 2D 完全失效。

必须区分“模块未安装”“接口版本不支持”“生成出错”“资源权限不足”及“用户中断”。日志和历史中避免明文保存私人素材的内容。

### 将来的安全重点

- 当前 2D 服务只监听回环地址；保留此默认值。
- 用户要从手机/平板访问时必须有认证及 Origin/CSRF 等保护，不直接暴露 ComfyUI API。
- 所有路径在主控侧验证并限制工作目录；不允许模块传入任意路径读写其他项目资料。
