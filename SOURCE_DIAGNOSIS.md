# 源码整改与诊断记录（2026-11-01，Dimitri-Yu）

本次改动把开发方式改为个人 fork、独立分支、PR 审核，并纠正旧交接对真值测试的解读。此记录不宣称估计器漂移、队友 NMPC 故障或正式赛道已修复。

## 已执行的调整

- Mac 工作区的组员启动入口恢复原估计器，删除运行时生成替代估计器会话的代码。显式传入 ground_truth 会在启动前报错。
- 原先用 ROS spawn_entity 替代 gz model 的本地脚本已退役。专用 Ubuntu 虚拟机的 spawn_drone.sh 已恢复为该仓库 HEAD 原文；生成脚本的差异为零。
- 历史 fix-runtime.sh 已退役，不再追加环境变量或强杀进程。旧内容与日志保存在本地，不随 PR 上传。
- 本次 PR 不搬运旧适配包或整包覆盖 race_node；旧本地工作目录保留，整改分支从上游 7f9726a 起步。

## 本机证据

环境：Ubuntu 22.04 ARM64、ROS Humble、源码构建 Gazebo Classic 11.10.2、REAL_UAV=false。没有操作真机。

| 检查 | 实测 | 边界 |
| --- | --- | --- |
| 原生 Gazebo 生成 | 空场景中，原生 gz model 生成小方块成功；查询位置为 `0 0 1 0 -0 0`；两条命令均退出0 | 过去挂起的根因仍未证明；当前环境不能复现，不应继续保留替换命令 |
| 原始 Crazyflie 生成脚本 | 完整仿真中 PX4 启动、DDS 通信、相机及原估计器输入正常 | 不代表飞行通过 |
| 无起飞的50秒被动采样 | PX4 odometry 6203条、hw_api odometry 4123条、MEKF输出3310条、真值4878条；494个采样快照 | 起飞前还在等预检，前段没有hw_api/MEKF输出；不能据此报全程频率 |
| 静止末段 | 地面真值z约-0.94218；hw_api/MEKF z同步在厘米量级变化 | 两者坐标原点不同，不能直接相减认定近1米误差 |
| 退役入口检查 | Bash语法检查通过；真值替代输入启动前返回2 | 不是源码修复后的飞行验收 |

原始日志保留在本地 `logs/source-audit-20261101/`，不覆盖此前历史测试。

锁定版本：

- LUS：ef820aa947dacefe50650872dac72685961ace3a
- simulation：7162d93e4b00c79227fc020d46fbf2b5c6abf858
- estimators：5ddc17d9f748bf2c5e58bf44569ce7f08597e626
- hw_api：59378fe224b15271db0d0a0f5b8afbe7df1c2fe9
- managers：7a1964bdd01d6bda6bcd40abcd1252428ed1431b
- Crazyflie：05f55a5cff530c12dccfcf17a288d7a5ea02b258

## 原估计器短流程复现失败

恢复原估计器与原始生成脚本后，本轮短流程到达 `IDLE → TAKEOFF → RACE → LANDING`，随后因 `position outside bounded integration region` 中止，进入 `ABORTED`，仿真已自动停止。失败原始日志单独保留为 `logs/source-audit-20261101/team-original-estimator.json`。这不是正式通过，也没有用真值输入继续完成流程。同步的原始输入采样正在用于定位偏离来源。

## 源码调查尚未完成

下一步用同一时段的原始 PX4 odometry、hw_api odometry、MEKF输出、独立真值、时钟、预检/解锁状态定位首次偏离，再检查相应源码。不能先假定 MEKF 是源头，也不能把不同机器的高度异常归为同一故障。

本机 managers 的预测时间间隔、对新旧消息的处理，以及 hw_api 的坐标原点/预检门控是待核查项，不是已证实根因。还需审查构建依赖适配及旧 Docker bringup 的自动源码修改，后者目前不应作为推荐修复路径执行。

此前真值下的短流程只能说明替代定位输入时其余链路有过成功记录；不等于原估计器通过，也不是实体7门、三圈或视觉导航验收。本次继续采用有时限的仿真测试，结果未完成前不预报通过。

## 审核请求

请组长先审核团队协作规则及上述证据边界。后续修复应携带“可复现失败 → 源码原因 → 最小源头改动 → 回归结果”，经 PR 审核后再合并。本 PR 保持 draft，不自动合并。
