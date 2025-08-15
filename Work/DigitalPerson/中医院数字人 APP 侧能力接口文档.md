## 1. 文档概述

本文档定义了中医院数字人 H5 与移动端 APP 之间的能力调用接口。

接口暂定基于浏览器环境中的 `window` 对象调用和回调。

| 序号  | 功能    | 调用方      | 说明            |
| :-: | ----- | -------- | ------------- |
|  1  | 语音播报  | H5 → APP | 通知 APP 播放合成语音 |
|  2  | 录音上传  | APP → H5 | 将录音文件推送给 H5   |
|  3  | 开关数字人 | H5 → APP | 启用/禁用数字人展示    |

### 流程图

![[数字人交互流程图-更新版本-0627-流程图 1.jpg]]

---

## 2. 接口详情

### 语音播报（H5 → APP）

 **功能说明**  ：H5 获取到后端对话接口返回的 TTS 音频后，调用此接口，将音频推送给 APP 播报。
 
 需要通过 APP 端注册该方法进入 window 中供 H5 进行调用。

**方法签名（暂定）**

```js
window.app.onTTSNotify(params: string): void
```

**请求参数（暂定）** (`params` 为 JSON 字符串)
 
|参数名|类型|必填|说明|
|---|---|---|---|
|requestId|string|是|本次播报唯一标识，用于对应回调|
|audioBase64|string|是|Base64 编码的音频内容|
|mimeType|string|是|音频格式，如 `audio/mp3`、`audio/wav`|
|text|string|否|文本内容（可选）|

#### 回调方法（APP → H5）

H5 定义该方法并挂载进入 windows 对象，由 App 侧在播报方法执行结束后进行回调

```js
window.h5.onTTSResult(JSON.stringify({
    requestId: string,
    code: number,
    message: string
}));
```

- `code = 0`：播报成功
- `code ≠ 0`：播报失败，`message` 返回错误描述

---

### 录音文件推送（APP → H5）

**功能说明**  ：APP 在用户结束录音并生成文件后，调用此接口将录音以 Base64 形式传给 H5 保存和处理。

 需要通过 H5 端注册该方法进入 window 中供 APP 进行调用。需要 APP 端实现录音长按-对话-松开-生成录音文件，并调用该方法。

**参数说明（暂定）**

|参数名|类型|必填|说明|
|---|---|---|---|
|requestId|string|是|本次录音唯一标识|
|audioBase64|string|是|Base64 编码的录音文件内容|
|duration|number|是|录音时长（秒）|
|mimeType|string|是|文件格式，如 `audio/mp3`|
|fileName|string|是|文件名，包含后缀|

---

### 开关数字人（H5 → APP）

**功能说明**  ：控制移动端是否展示数字人动画/视频流（预留接口，后续可扩展）。