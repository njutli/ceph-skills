# Note 33 优化建议

## 1. `waiting_for_map` 的触发细节
- **原文错误**：原文暗示 Op 携带的 Epoch 大于 OSD 本地 Epoch 时会进入 `waiting_on_map` 等待。
- **改正**：实际上，如果 Op 携带的 Epoch **远小于** OSD 的 `osdmap_epoch`，OSD 会直接返回 `EAGAIN` 让客户端更新 Map，而不会无限期等待。只有当 Op 的 Epoch **略新于** OSD（例如 OSD 正在处理新 Map 的间隙）时，才会短暂进入 `waiting_on_map`。
