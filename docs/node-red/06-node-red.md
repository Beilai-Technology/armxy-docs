<img width="487" height="644" alt="image" src="https://github.com/user-attachments/assets/7604c6d8-07c2-4935-bfd3-be1a4cd43a4e" /># 第6 篇 Node-RED Modbus RTU 数据采集及MQTT 云端上传演示

## 1. 项目概述

### 1.1 项目背景

本项目基于 Node-RED  可视化开发平台，实现 ARMxy  边缘网关通过 Modbus RTU  串口协议，周期性采集 Modbus  从站设备的寄存器数据。通过脚本完成原始数据解析与结构化封装后，基于 MQTT协议将标准化 JSON 数据上传至云端 MQTT  服务器，完成工业现场设备数据上云。

<img width="553" height="553" alt="image" src="https://github.com/user-attachments/assets/72202aea-520c-4068-8391-4f23ae31549c" />

整体流程采用边缘采集 - 数据处理 - 云端上报的标准链路，核心数据流如下：

Modbus RTU  数据读取 →  Function  脚本数据格式化处理 →  MQTT  云端发布上报

## 2. 环境准备

### 软件环境要求

本次演示所需软件及插件如下：

1. Node-RED ： ARMxy  自带，作为核心流程开发与数据处理载体


2. Modbus扩展节点: Node-RED节点管理安装 node-red-contrib-modbus


3. MQTT调试工具：MQTTX，用于订阅云端主题、验证上报数据有效性


4. Modbus仿真工具: Modbus Slave, 可虚拟 Modbus RTU从站设备, 完成流程调试


服务器采用 Linux系统私有化部署的EMQX 开源MQTT服务器


## 3.核心流程搭建与分步配置


### 3.1整体流程搭建


在 Node-RED画布中依次拖拽核心节点，并按数据流向顺序连线，搭建完整采集上云链路，核心节点包含：

<img width="474" height="64" alt="image" src="https://github.com/user-attachments/assets/7218b5ea-6b33-4336-9153-e51cc0a182ba" />

1. modbus-read节点: 负责串口 Modbus RTU寄存器数据周期性采集

2. function节点：完成原始数据换算、结构化JSON封装

3. mqtt out节点：连接云端MQTT服务器，发布结构化设备数据

### 3.2 Modbus Read节点配置

该节点为数据采集入口，核心配置串口参数、Modbus读写规则、轮询周期，具体步骤如下：双击打开 modbus-read节点,在最下方 Server行新建串口RTU服务,参数配置如图:

<img width="487" height="644" alt="image" src="https://github.com/user-attachments/assets/b99331c0-23eb-4b50-9985-cef371cbcf57" />

寄存器读取参数配置：

<img width="488" height="454" alt="image" src="https://github.com/user-attachments/assets/157314a6-c079-4378-be09-11214d990b98" />

配置保存后，节点将周期性采集寄存器数据，原始输出格式为数字数组。

### 3.3 Function节点配置

Modbus原始数据为纯数字数组,无业务含义、无法直接云端解析。通过 Function节点JavaScript脚本，完成结构化封装、时间戳绑定。

<img width="466" height="268" alt="image" src="https://github.com/user-attachments/assets/f13df767-45dd-488a-a322-1c873769a3e5" />

### 3.4 MQTT Out节点配置


双击 mqtt out节点，点击服务器编辑按钮，新增MQTT连接配置：

<img width="480" height="393" alt="image" src="https://github.com/user-attachments/assets/36ff36e0-be03-4c96-b501-0f616b9dc957" />

本次演示服务器未启用身份认证，因此无需填写用户名和密码。使用公有云IoT服务器必须填写安全标签页的用户名和密码。

消息发布参数配置：

<img width="391" height="306" alt="image" src="https://github.com/user-attachments/assets/d2fdc9f2-dfb2-4d22-9c46-6c4c82fa5c3c" />

1. Topic: iot/ modbus/ data(自定义)

2. QoS: 0(最多一次送达)

保存配置后，流程自动将格式化后的JSON数据周期性推送至云端MQTT服务器。

注：若使用公有云IoT服务器，必须改成官方标准 Topic。

## 4. MQTTX客户端连接配置

1.新建MQTT客户端连接，填写服务器地址，端口，直接建立连接。

<img width="554" height="226" alt="image" src="https://github.com/user-attachments/assets/00baf56d-4e53-4251-8920-4d6915eaf815" />

### 4.2主题订阅与数据验证

1.新建订阅任务, 订阅主题: iot/ modbus/ data, QoS=0

<img width="543" height="401" alt="image" src="https://github.com/user-attachments/assets/a5ef8d0b-0232-4c85-8122-0a8148efc713" />

2.返回 Node-RED界面，点击【部署】启动完整流程

3.观察MQTTX 消息窗口，可周期性接收标准化JSON设备数据，代表采集、处理、上云全链路正常

<img width="481" height="261" alt="image" src="https://github.com/user-attachments/assets/69a91944-da8f-4e3c-ab3b-a225e1fc3504" />

## 5. 功能扩展：边缘数据优化与阈值告警
基础流程为全量数据周期性上报，存在无效数据多、带宽浪费、无告警机制等问题。可通过优化Function  脚本，实现数据过滤、阈值告警、异常剔除等边缘预处理功能。

### 5.1 阈值告警上报代码（温度超限告警）

<img width="413" height="362" alt="image" src="https://github.com/user-attachments/assets/7b6fdc37-9186-4fa7-bb20-69f8ff2173b5" />

<img width="495" height="262" alt="image" src="https://github.com/user-attachments/assets/8860fb7a-6186-41a1-b8dc-1a2f570296c4" />

## 6. 常见故障排查方案

### 6.1 Modbus Read 节点串口报错/读取失败

1. 端口权限不足，执行授权命令
2. 参数不匹配：核对波特率、校验位、从站地址、寄存器地址与设备参数一致

### 6.2 MQTT 节点离线、连接断开

1. 端口拦截：本地防火墙、设备防火墙放行 MQTT  服务端口（默认 1883 ）
2. 连接配置错误：核对服务器地址、端口格式是否规范

### 6.3 MQTTX 无法接收上报数据

1. 主题不一致：发布主题与订阅主题严格区分大小写，必须完全一致
2. 流程未部署：修改 Node-RED  配置后需重新部署生效
3. 数据被拦截： Function  脚本逻辑异常导致消息终止，可通过节点调试窗口排查

## 售后支持：0755-29451836
