# Fix PureTavern zip preview high memory hang

## Goal

彻底查明并以最小侵入方式修复在启动高内存占用模式（快速导入模式）下，选择 PureTavern 数据包进行读取与预览时，界面卡在“正在快速读取并预览 PureTavern 数据包（内存占用较高）…”的根本原因，确保变更可向上游仓库提交高质量 PR。

## Problem & Root Cause Analysis

1. **非安全上下文下 `crypto.subtle` 抛错导致中断**：
   - `apps/web/src/features/import-export/application/archive-codec.ts` 中的 `sha256Hex` 依赖 `crypto.subtle.digest`。
   - Web Cryptography API（`crypto.subtle`）在 W3C 规范中仅在安全上下文（HTTPS / localhost）可用。
   - 当用户通过局域网 IP（`http://192.168.x.x:8899`）、移动端/部分 WebView 或 HTTP 反代使用 PureTavern 时，`crypto.subtle` 为 `undefined`。
   - `sha256Hex` 调用 `crypto.subtle.digest` 立即抛出 `TypeError: Cannot read properties of undefined (reading 'digest')`。
   - 对比发现：`archive-service.ts` 与 `streaming-archive-reader.ts` 均统一使用 `js-sha256` 纯 JS 实现，唯独 `archive-codec.ts` 引入了对 `crypto.subtle` 的脆弱依赖。

2. **UI 错误恢复机制缺失，导致视觉上“一直卡住”**：
   - 在 `apps/web/src/features/import-export/runtime/index.js` 的 `previewImport` 中：
     进入时设置了 `target.textContent = '正在快速读取并预览 PureTavern 数据包（内存占用较高）…'`。
   - 当预览链路发生任何异常（如 `crypto.subtle` 报错、ZIP 格式校验异常 `extra-file`、解压异常等），外层 `bindFilePicker` 捕获后仅调用 `notify('error', error.message)`。
   - `target.textContent` 从未被重置或更新为错误信息，确认按钮保持 `disabled`。
   - 若环境无 toastr 或弹窗稍纵即逝，用户界面将永久停留在“正在快速读取并预览…”文案上，视觉表现即“永久卡住”。

3. **高内存模式下重复内存克隆与无效 IndexedDB 查询**：
   - `archive-codec.ts` 在 `decodeArchive` 与 `sha256Hex` 中对解压后的每一个文件执行多重 `.slice()` 内存深拷贝，成倍放大了内存峰值。
   - `archive-participant-registry.ts` 的 `participant.preview` 遍历所有条目逐一查询 IndexedDB，即使该模块未勾选（`selected === false`）也执行了全量冲突查询，加剧了主线程与 IndexedDB 的阻塞。

## Requirements

1. **环境兼容性（跨上下文 SHA-256）**：
   - 将 `archive-codec.ts` 中的 `sha256Hex` 统一使用已有的 `js-sha256` 库（与 `streaming-archive-reader.ts` 和 `archive-service.ts` 保持一致），消除对 `crypto.subtle` 的上下文依赖。
2. **UI 容错与状态恢复**：
   - 在 `runtime/index.js` 的 `previewImport` 与 `previewTauriTavernImport` 中增加异常捕获，发生错误时直接在 preview 区域显示“读取或预览失败：[具体错误信息]”，避免视觉假死。
   - 同样在 `importArchive` 与 `importTauriTavern` 中增强错误文案显示。
3. **内存与性能轻量优化（最小侵入）**：
   - 移除 `archive-codec.ts` 中多余的 `data.slice()` 内存冗余复制。
   - 在 `archive-participant-registry.ts` 中增加 `if (selected)` 条件判断，未选择的模块跳过逐条 IndexedDB 冲突查询（与流式预览行为保持完全一致）。
4. **单元测试与上游 PR 友好**：
   - 补充针对非安全上下文（无 `crypto.subtle`）的单元测试。
   - 补充 UI 预览出错时的状态呈现测试。
   - 保证现有 500+ 个测试用例 100% 通过，无破坏性修改。

## Acceptance Criteria

- [x] 在 `crypto.subtle` 不可用的环境（模拟非安全上下文）下，`sha256Hex` 和 `decodeArchive` 均能正常完成哈希计算与解压。
- [x] 选择数据 ZIP 预览或导入遇到错误时，预览区域能够清晰展示错误原因，而非永久卡在“正在快速读取并预览…”文案。
- [x] 运行 `pnpm --filter @pure-tavern/web test` 全部通过（75 个测试文件、505 个测试全部通过）。
- [x] 变更代码维持最小侵入原则，代码风格与上游严格统一。
