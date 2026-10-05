# 主备话筒切换校验台（同声传译彩排）

一个**完全离线、纯前端**的校验台，用于彩排“主话筒切到备用”时耳返不静音、授权不错位。
技术栈：TypeScript + React 18 + Vite，无后端、无任何网络 API 调用（**禁止假接口**——
应用只使用浏览器原生 `navigator.mediaDevices` 与 `Web Audio API`）。

## 它解决的问题

- 主话筒切到备用若失败，耳返会静音；
- 连续点击可能让较早返回的授权 / 取流结果“反夺线路”；
- 切换中候选夭折、上下文恢复失败等异常路径必须可验证。

## 操作流程

1. **授权麦克风**：取一条流 → 恢复 `AudioContext` → 枚举音频输入设备 → 立即归还探测流。
2. 分别为**主路**、**备路**选择输入设备并**试听**：页面显示设备、轨道状态与实时电平。
3. 两路都试听在线后，**武装主路**：主路成为耳返活动输出；**武装期间禁止另开试听**。
4. **发起切换**：
   - 重新申请备用候选流，**备用就绪前主路持续输出**；
   - 候选就绪后做 **80ms 线性增减益交叉淡化**；
   - 交叉完成后才停止旧主路轨道、断开旧节点；
   - 在淡化尚未完成时可**撤销切换 / 保留主路**：按 `AudioContext.currentTime`
     与已排定曲线求出两路当前增益，取消未执行自动化，再从当前增益平滑回到主路全开、
     备用关闭；回退完成后才释放候选备用设备。
5. **停止**：停止全部相关轨道、断开全部音频节点并关闭 `AudioContext`，
   界面不残留任何麦克风占用状态。

## 状态与故障语义

| 情况 | 行为 |
| --- | --- |
| 候选被拒绝 / 提前结束 / `AudioContext.resume()` 失败 | 释放候选，**保留原主路**，可重新发起切换 |
| 淡化未完成时操作员撤销 | 从当前两路增益平滑回到主路 1 / 备用 0；回退完成后才释放候选，旧主路全程有效 |
| 切换已完成后点击撤销 | 按钮不可用；旧主路已经释放，不能作为可回退资源复活 |
| 活动主路（武装后的主路 / 已提升的备路）轨道结束 | 进入**故障态**，释放全部线路；**重新试听主、备后方可再次武装** |
| 故障自动接管开启且主路断轨时备用监听仍在线 | 已取得的备用流以当前代次裁决直接接管耳返（不再取流），80ms 增益切换完成后才释放旧主路；页面显示活动设备、代次与故障来源 |
| 故障自动接管开启但备用先断 / `AudioContext` 不可用 / 停止或手动切换进行中 | 不接管，按原故障清理规则释放资源 |
| 切换中的迟到候选 / 旧代次流 | 立即停止，不得复活线路、不得改写当前提示 |
| 停止 | 释放全部流与节点，麦克风占用指示熄灭 |

每次**试听、武装、切换、停止、故障**都递增“代次（generation）”，界面上实时显示 `#N`。
所有异步操作返回时都校验代次：只有最新代次可以接管线路。

## 浏览器能力

- 桌面 Chrome / Edge / Firefox（较新版本），桌面版 Chromium 内核最稳定；
- 必须运行在**安全上下文**：`https://…` 或 `http://localhost` / `http://127.0.0.1`。
  用 `http://<局域网IP>` 直接打开会被浏览器禁用麦克风，页面会明确显示原因；
- 缺少 `navigator.mediaDevices` / `getUserMedia`、**拒绝权限**或**所选设备不存在**时，
  页面均会给出中文原因；已工作的线路不会被错误清空；
- 浏览器标签栏的麦克风图标与页面右上角“麦克风占用中”指示一致，停止后必须熄灭。

> `getUserMedia` 必须由用户手势触发，本页所有取流均由按钮点击发起。

## 本地开发

```bash
npm install
npm run dev        # http://localhost:8080
npm run build      # 类型检查 + 产物到 dist/
npm run preview    # 本地预览构建产物
```

运行时零网络请求；`npm install` 仅为构建期依赖。

## 测试

```bash
npm run test       # Vitest：可控媒体替身核对轨道停止与节点断开
npm run e2e        # Playwright：Chromium 假媒体覆盖授权 / 拒绝 / 试听 / 武装 / 切换
```

- **Vitest**（`src/domain/SwitchbenchEngine.test.ts`）使用 `src/testing/fakes.ts` 中
  完全可控的假 `MediaStreamTrack` / `AudioContext` / 调度器，精确制造
  “候选提前结束”“迟到候选”“恢复失败”“快速操作”等竞态，并可推进假音频时钟
  核对淡化起点、中点、临近终点的连续增益；同时断言 `stop()` 与
  `disconnect()` 次数。
- **Playwright**（`e2e/authorization.spec.ts`）通过 `--use-fake-device-for-media-stream`
  与 `--use-fake-ui-for-media-stream` 走真实授权流程；拒绝用例用
  `--deny-permission-prompts` 确定性触发。端到端测试**不注入任何页面假接口**。

## Docker Compose

构建期需要访问 npm 源拉取依赖；容器运行时为纯静态文件、无需联网。

```bash
# 发布页面（宿主端口可用 WEB_PORT 覆盖，容器内固定 8080）
docker compose up --build web
WEB_PORT=9090 docker compose up --build web

# 一次性验收服务：build + Vitest + Playwright，跑完即退出（退出码即验收结果）
docker compose --profile verify run --rm verify
```

通过非 localhost 的 `http://<宿主IP>:<WEB_PORT>` 访问时，浏览器会因非安全上下文禁用
麦克风——请改用 localhost 端口转发或在前面加 https 反向代理。

## 目录

```
src/
  domain/              # 纯领域逻辑，不依赖 React
    types.ts             # 类型与不变量
    browserHost.ts       # 真实浏览器宿主（禁止假接口）
    SwitchbenchEngine.ts # 代次、线路、交叉淡化、故障态
    SwitchbenchEngine.test.ts
  testing/fakes.ts     # 仅测试用：可控媒体替身
  components/          # React 界面
  state/useEngine.ts
e2e/                  # Playwright 授权流程
```
