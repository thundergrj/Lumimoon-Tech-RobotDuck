# 05-配置 Microduck Studio

[返回首页](../README.md) · 上一章：[烧录系统](04-烧录系统.md) · 下一章：[标定 HOME 位与 IMU](06-标定HOME位与IMU.md)

**目标：**确认网页、主程序、15 颗舵机与 IMU 都有真实且新鲜的状态，核对模型部署包。此阶段不试走。

## 1. Studio 与 robotd 分工

| 组件 | 作用 | 怎么确认 |
| --- | --- | --- |
| Microduck Studio | 网页界面、标定向导、日志、模型管理 | HTTP 页面及实时状态正常 |
| robotd | 管理总线、IMU、标定、安全状态与策略 | 主程序回读更新，不一直报启动中 |
| root 小助手 | 受控总线交接、写标定与切换模式 | Studio 对应操作可完成并核验 |
| 模型部署包 | 给运行时提供动作策略及执行约定 | 导入、选用、执行状态分别确认 |

网页能打开不等于 robotd 就绪，服务显示 active 也不等于已读全舵机。停止 Studio 或 robotd 不能代替卸力确认。

## 2. 已有配套镜像：先检查，别重复安装

在瑞莎终端执行：

```bash
systemctl status microduck-studio.service robotd.service --no-pager
systemctl cat microduck-studio.service
systemctl cat robotd.service
sudo journalctl -u microduck-studio.service -b -n 60 --no-pager
sudo journalctl -u robotd.service -b -n 100 --no-pager
```

已有服务且只是 Studio 停止时：

```bash
sudo systemctl start microduck-studio.service
```

电脑浏览器打开 **`http://实际主控IP:8000/`**，保留 `http://` 和 `:8000`。不要误开摄像头调试端口 8080 或把 8443 信令端口当主页。浏览器中的旧页面可能仍显示断线前的画面，需看实时连接状态。

## 3. 手动安装 Studio：仅用于条件已满足的系统

已有预装服务或签名升级系统的客户跳过本节，避免覆盖现有版本。

前提：系统已有匹配的 **HD1910 robotd、模型、标定模式、socket 与权限配置**，且 Python 满足该 Studio 版本要求。下面参考源码提交 `e20877b8a099e8d669ef14e7f22275a946625f55`，其说明要求 Python 3.13；不能把当前通用源码直接放进 Python 3.11 后承诺可运行。

命令在瑞莎上以 `radxa` 用户运行，目标目录应尚不存在；已有目录先由维护人员备份、核对迁移：

```bash
python3 --version
git clone https://github.com/fanhao375/microduck-replica.git ~/microduck-source
git -C ~/microduck-source checkout e20877b8a099e8d669ef14e7f22275a946625f55
cp -a ~/microduck-source/software/studio ~/studio-dev
cd ~/studio-dev
python3 -I run.py --selfcheck
```

自检中主程序不可用、协议不支持或依赖报错时先处理，不继续装服务。确认后安装同一源码版本的小助手与服务：

```bash
cd ~/studio-dev/deploy
sudo sh install.sh radxa
sudo install -m 644 microduck-studio.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now microduck-studio.service
```

核对 robotd 的总线保护 drop-in 已安装。不要自行启用“保持扭矩交还”等尚未按配套版本验收的扩展设置；本教程在卸力状态交还总线。

源安装说明见 [Studio README](https://github.com/fanhao375/microduck-replica/blob/master/software/studio/README.md)与[小助手安装说明](https://github.com/fanhao375/microduck-replica/blob/master/software/studio/deploy/INSTALL.md)。原生系统缺少 robotd 时，这些命令只安装界面，不完成机器人运行时部署。

## 4. 第一次打开网页

1. 设自己的 Studio 登录密码，记录机器名称与实际 IP。
2. 确认页面为真实设备，没有“演示/模拟”标记。
3. 托稳机体，点“卸力”，等待所有关节都读回已卸力；不是只看按钮回执。
4. 查看总览：主程序状态、舵机数量、IMU、主控温度、电压与状态更新时间。
5. 打开日志，确认无持续重启、串口占用、超时或“读不出当前姿势”。

首次出厂可能处于 `commissioning` 标定模式，这是正常的准备阶段。一直停在 `configuring_bus`、`waiting_for_bus`，或多次读不到实时样本，就不是可以直接试走的状态。

## 5. 总线识别与接管

1. 机体已托住、已卸力，进入“舵机”页。
2. 通过 Studio 接管总线，等待主程序交出串口。不要手动再启动 robotd 或另开脚本。
3. 选择正确串口和 **1 Mbps**，读取预期 ID，必要时做一次完整扫描。
4. 应识别 **10–14、20–24、30–34** 共 15 颗。核对 ID 与实际部位，不能只看数量为 15。
5. ID 200 单独按 IMU 识别，不能作为普通舵机写位置或中位。
6. 连续多次回读，观察是否丢 ID、校验错误、超时或数据陈旧。
7. 测试完确认卸力，点“交还主程序”，等主程序与 IMU 状态恢复。

PING 通过只说明能通信，不能证明输出轴、IMU 原始数据或姿态转换正常。进入运动前还要完成下一章。

## 6. 核对 IMU 来源

在 IMU 页确认实际来源为总线板，地址和协议与随货说明一致。`auto` 仅在当前 robotd 明确支持时使用，不因板子名称含 DXL 就强行选择 Dynamixel 协议。

先看静止数据是否持续更新，再缓慢转动整个躯干，确认加速度、角速度或姿态有合理变化。尚未做安装方向标定时，不把倾角不对直接判为芯片损坏，也不把 ID 200 有回应当作全部合格。

## 7. 模型部署包的导入与选用

模型包至少应有同一次导出的：

```text
policy.onnx
manifest.json
deployment-contract.json
```

导出工具还生成执行参数、运行参数或说明文件时一并保留，不混入多个模型与训练日志。部署包需匹配舵机、结构、关节顺序、观测、HOME、限位、控制频率、P/D、动作幅度与滤波；同为 HD1910 或同为 61→14 并不足以确认兼容。

1. 在“模型管理”选择配套 ZIP/tar.gz/tgz 部署包，执行导入。
2. 查看文件与运行时预检结果。只有 ONNX 时先让模型提供方补齐合同，不能随手编造参数通过检查。
3. **已安装、当前选用、正在执行是三个状态。** 保持卸力，再按用途选择“使用此模型”。
4. 等主程序确认实际路径/身份和用途，再记录选用成功；站立、坐站、行走可能使用不同策略，不能把行走模型重复填入所有用途。
5. 合同不匹配、基础 HOME 或 P 参数需变化时，按配套迁移流程处理，不强行热切换。

导入说明见 [MODEL_IMPORT](https://github.com/fanhao375/microduck-replica/blob/master/software/studio/MODEL_IMPORT.md)。软件预检通过不代表实物行走已验收。

若提示“主程序连接或回读失败”，先排查真实连接、标定模式、模型运行器是否启用和协议版本，不因文件已导入就反复点“开始走”。

## 8. 本章验收

- [ ] 网页、主程序、小助手版本已登记，实时回读持续更新。
- [ ] 15 颗舵机身份、ID 200 来源、总线稳定性已检查。
- [ ] 接管与卸力交还流程可完成，没有两个程序抢串口。
- [ ] 模型来源、合同、选用用途和主程序回读已核对。
- [ ] 机体保持卸力，准备正式标定。

下一步：[06-标定 HOME 位与 IMU](06-标定HOME位与IMU.md)。
