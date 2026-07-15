# 一、内容简介

在工业现场中，网关作为边缘数据处理节点，通过 OPC UA  协议连接 PLC 、工业控制设备或其他支持 OPC UA  的数据源，实现现场设备数据的标准化采集；随后利用 MQTT  协议将采集后的数据经过格式转换后安全、高效地上传至华为云 IoT  平台，实现设备数据的远程监控与管理。

本教程基于钡铼技术 BL118  边缘计算网关搭建一个完整的边缘数据采集与上云流程：使用网关内置 Node-RED  平台，通过 OPC UA  协议采集工业设备数据（使用本地模拟 OPC UA服务器），再通过 MQTT  协议将数据上报到华为云 IoT  平台。

# 二、前置准备 

## 2.1 OPC UA 插件安装 

打开 Node-RED  右上角菜单 - 管理面板 - 安装，搜索插件： node-red-contrib-opcua ，点击安装； 

安装完成后左侧节点栏出现 opcua-server 、 opcua-client 、 opcua  工具类节点，用于搭建本地OPC UA  服务与采集客户端。 

## 2.2 华为云IoT 平台准备 

1.  登录华为云 IoTDA ，创建产品、注册设备，获取设备 ID 、产品 ID ；
2.  使用华为云 MQTT  客户端生成工具（地址： https://iot-tool.obs-website.cn-north-4.myhuaweicloud.com/ ）生成 MQTT  连接客户端 ID 、用户名、密码； 
3.  记录 IoTDA  接入域名、 MQTT  端口（标准 1883  非加密 /8883 TLS  加密）。

# 三、OPC UA 完整配置流程 

## 3.1 创建本地模拟OPC UA Server

1.  拖拽 opcua-server  节点至画布，双击打开配置面板：

<img width="365" height="748" alt="image" src="https://github.com/user-attachments/assets/78500024-8569-4460-b34b-3169bf601284" />

<img width="142" height="54" alt="image" src="https://github.com/user-attachments/assets/723d7d84-ab7a-4905-8079-abcdb18fd950" />

2.部署流程后可以看到OPC UA服务器正常运行。



OPC UA server

running

## 3.2 OPC UA Client客户端配置


从 Node-Red左侧面板中拖一个OpeUa-Client节点到工作区, 双击OpcUa-Client节点, 配置如下：

<img width="377" height="424" alt="image" src="https://github.com/user-attachments/assets/0562112e-2d2a-4fbb-b280-66d87213557d" />

点击 Endpoint后的"笔型"按钮,打开 Endpoint编辑页面, IP地址为OPC UA Server所在设备地址，配置如下：

<img width="380" height="290" alt="image" src="https://github.com/user-attachments/assets/e85f14b9-96b4-4601-a833-725f69cf55d0" />

## 3.3 创建OPC UA测试变量

1. 拖拽 inject  注入节点，命名 add value ，配置如下：

<img width="554" height="121" alt="image" src="https://github.com/user-attachments/assets/6c45d641-15bb-4277-9d4b-c908287e4b07" />

2.  连接 debug  调试节点到 server  节点后面，部署流程；
3.  手动点击 inject  节点触发，调试窗口输出变量创建成功日志

<img width="157" height="123" alt="image" src="https://github.com/user-attachments/assets/9f3cc0eb-5b1a-4aee-ad17-f4f2258614be" />

## 3.4 订阅OPC UA 变量实时数据 
1.  复制 add value  节点，重命名为 Subscribe ，修改消息配置：

<img width="421" height="138" alt="image" src="https://github.com/user-attachments/assets/a0a20875-56c3-439b-b5e9-56306fcf7031" />

2.  将 Subscribe  节点输出连接至 opcua-client  输入端；
3.  部署流程后，服务端会重启，需先执行 add value  重建变量，再触发 Subscribe  订阅；

<img width="174" height="211" alt="image" src="https://github.com/user-attachments/assets/1bceb361-498a-4b93-9702-97f9d64be8e7" />

## 3.5 取消订阅 


1. 复制 Subscribe节点, 命名 Unsubscribe, 修改 payload:

<img width="428" height="149" alt="image" src="https://github.com/user-attachments/assets/944f6cad-2456-433d-9d37-2bc7f3ceb596" />

2.触发节点后日志提示 unsubscribing，停止接收该变量推送。

# 四、MQTT节点对接华为云IoTDA配置

## 4.1m qtt节点配置

拖入一个 mqtt out节点，双击配置如下：

<img width="384" height="302" alt="image" src="https://github.com/user-attachments/assets/dac98efe-ff41-4d23-8c61-76e46db2f035" />

上报主题: 华为云标准属性上报主题$ oc/ devices/{device _ id}/ sys/ properties/ report.(device _ id}替换为真实设备ID;

点击服务端后的"笔型"按钮，配置接入信息

<img width="464" height="317" alt="image" src="https://github.com/user-attachments/assets/dcbd4761-107b-4e1c-9d98-46242b1f9c4d" />

<img width="493" height="133" alt="image" src="https://github.com/user-attachments/assets/2f329f2a-633d-419a-a5c1-90e2c034006b" />

## 4.2 Function数据格式转换节点

OPC UA采集到的原始数据不能直接上报，需要转换为华为云IoTDA要求的JSON格式。在OpeUa-Client和 mqtt out节点之间拖入一个 Function节点, 双击打开编辑, 填入以下代码：

```js
//转换OPC UA采集数据为华为云标准属性上报报文

let data= msg. payload. value;

let res={
" services":[],
" properties": {
" temperature": data // 对应平台产品模型定义的属性名},
"eventTime": new Date(). getTime(). toString()


};

msg. payload= JSON. stringify(res);

return msg;
```

# 五、云端验证

全部节点部署完成，先点击订阅，此时还未赋值，执行 add value 创建变量后，再触发Subscribe 开启实时采集，此时云端变量值变为设定的30；

<img width="208" height="120" alt="image" src="https://github.com/user-attachments/assets/735f8899-18dd-418f-8f15-e12ebc735c81" />

<img width="200" height="122" alt="image" src="https://github.com/user-attachments/assets/c9872aa8-6d8e-4b10-b1e5-bd13747e1203" />

# 六、OPC UA、MQTT优缺点对比

| 对比项 | OPC UA | MQTT |
| 主要用途 | 工业设备数据采集 | 工业设备数据采集 |
| 通信模型 | 客户端/服务器 | 发布/订阅 |
| 实时性 | 高 | 高 |
| 安全性 | 支持证书、加密 | 依赖TLS认证 |
| 数据模型 | 丰富，支持节点模型 | 简单消息模型 |
| 典型场景 | PLC、SCADA、MES | loT平台、云端监控 |

OPC UA和MQTT并不是替代关系，而是上下游协作关系。OPC UA负责工业现场数据标准化采集，MQTT负责数据高效传输到云平台。

# 七、总结

在工业物联网场景中，OPCUA与MQTT组合是一种常见的数据上云架构。OPCUA负责现场设备数据采集，MQTT 负责边缘侧与云平台的数据传输，两者结合能够兼顾工业兼容性和云端接入效率。


```json
[
    {
        "id": "f3f15bacdde76a69",
        "type": "OpcUa-Server",
        "z": "e65dce2b3df1277b",
        "port": "4840",
        "name": "",
        "endpoint": "",
        "users": "",
        "nodesetDir": "",
        "autoAcceptUnknownCertificate": true,
        "registerToDiscovery": false,
        "constructDefaultAddressSpace": true,
        "allowAnonymous": true,
        "endpointNone": true,
        "endpointSign": true,
        "endpointSignEncrypt": true,
        "endpointBasic128Rsa15": true,
        "endpointBasic256": true,
        "endpointBasic256Sha256": true,
        "maxNodesPerBrowse": 0,
        "maxNodesPerHistoryReadData": 0,
        "maxNodesPerHistoryReadEvents": 0,
        "maxNodesPerHistoryUpdateData": 0,
        "maxNodesPerRead": 0,
        "maxNodesPerWrite": 0,
        "maxNodesPerMethodCall": 0,


        "maxNodesPerRegisterNodes": 0,
        "maxNodesPerNodeManagement": 0,
        "maxMonitoredItemsPerCall": 0,
        "maxNodesPerHistoryUpdateEvents": 0,
        "maxNodesPerTranslateBrowsePathsToNodeIds": 0,
        "maxConnectionsPerEndpoint": 20,
        "maxMessageSize": 4096,
        "maxBufferSize": 4096,
        "maxSessions": 20,
        "x": 420,
        "y": 140,
        "wires": [
            [
                "0beeecb74fd4c7ea"
            ]
        ]
    },
    {
        "id": "2fa06dcd0efae16a",
        "type": "OpcUa-Client",
        "z": "e65dce2b3df1277b",
        "endpoint": "3b672bd8c700a6a4",
        "action": "read",
        "deadbandtype": "a",
        "deadbandvalue": 1,
        "time": 10,
        "timeUnit": "s",


        "certificate": "n",
        "localfile": "",
        "localkeyfile": "",
        "securitymode": "None",
        "securitypolicy": "None",
        "useTransport": false,
        "maxChunkCount": 1,
        "maxMessageSize": 8192,
        "receiveBufferSize": 8192,
        "sendBufferSize": 8192,
        "setstatusandtime": false,
        "keepsessionalive": false,
        "name": "",
        "applicationName": "",
        "applicationUri": "",
        "x": 360,
        "y": 320,
        "wires": [
            [
                "46d1590b12e81355",
                "3f95a8c49fe83119"
            ],
            [],
            []
        ]
    },
    {


        "id": "3f95a8c49fe83119",
        "type": "debug",
        "z": "e65dce2b3df1277b",
        "name": "debug 1",
        "active": true,
        "tosidebar": true,
        "console": false,
        "tostatus": false,
        "complete": "true",
        "targetType": "full",
        "statusVal": "",
        "statusType": "auto",
        "x": 540,
        "y": 260,
        "wires": []
    },
    {
        "id": "0beeecb74fd4c7ea",
        "type": "debug",
        "z": "e65dce2b3df1277b",
        "name": "debug 2",
        "active": true,
        "tosidebar": true,
        "console": false,
        "tostatus": false,
        "complete": "false",
        "statusVal": "",




        "y": 140,
        "wires": [
            [
                "f3f15bacdde76a69"
            ]
        ]
    },
    {
        "id": "9307d4d78d376da3",
        "type": "inject",
        "z": "e65dce2b3df1277b",
        "name": "Subscribe",
        "props": [
            {
                "p": "payload"
            },
            {
                "p": "topic",
                "vt": "str"
            }
        ],
        "repeat": "",
        "crontab": "",
        "once": false,
        "onceDelay": 0.1,
        "topic": "ns=1;s=TestAddVariable",
        "payload": "{ \"action\": \"subscribe\"}",


        "payloadType": "json",
        "x": 140,
        "y": 300,
        "wires": [
            [
                "2fa06dcd0efae16a"
            ]
        ]
    },
    {
        "id": "7bc50aa9e07b3450",
        "type": "inject",
        "z": "e65dce2b3df1277b",
        "name": "Unsubscribe",
        "props": [
            {
                "p": "payload"
            },
            {
                "p": "topic",
                "vt": "str"
            }
        ],
        "repeat": "",
        "crontab": "",
        "once": false,
        "onceDelay": 0.1,


        "topic": "ns=1;s=TestAddVariable",
        "payload": "{ \"action\": \"unsubscribe\"}",
        "payloadType": "json",
        "x": 130,
        "y": 340,
        "wires": [
            [
                "2fa06dcd0efae16a"
            ]
        ]
    },
    {
        "id": "513a0236a53a64b2",
        "type": "mqtt out",
        "z": "e65dce2b3df1277b",
        "name": "",
        "topic": "$oc/devices/6a4df1276b6c4d5f8d6731e1_test/sys/properties/report",
        "qos": "0",
        "retain": "",
        "respTopic": "",
        "contentType": "",
        "userProps": "",
        "correl": "",
        "expiry": "",
        "broker": "9ff3171cd70e07aa",
        "x": 860,
        "y": 320,


        "wires": []
    },
    {
        "id": "46d1590b12e81355",
        "type": "function",
        "z": "e65dce2b3df1277b",
        "name": "function 7",
        "func": "//  取出  OPC UA  推送的数值 \nvar value = msg.payload;\n\n//  华为云属\n性上报  JSON  格式 \nmsg.payload = {\n    \"services\": [\n        {\n            \"service_id\": \"s1\",          //  改为你实际的服务 ID\n            \"properties\": {\n                \"hum\": value       //  改为你实际的\n属性标识符 \n            }\n        }\n    ]\n};\n\nreturn msg;",
        "outputs": 1,
        "timeout": 0,
        "noerr": 0,
        "initialize": "",
        "finalize": "",
        "libs": [],
        "x": 540,
        "y": 320,
        "wires": [
            [
                "513a0236a53a64b2"
            ]
        ]
    },
    {
        "id": "3b672bd8c700a6a4",


        "type": "OpcUa-Endpoint",
        "endpoint": "opc.tcp://127.0.0.1:4840",
        "secpol": "None",
        "secmode": "None",
        "none": true,
        "login": false,
        "usercert": false,
        "usercertificate": "",
        "userprivatekey": ""
    },
    {
        "id": "9ff3171cd70e07aa",
        "type": "mqtt-broker",
        "name": "",
        "broker": "8292c9706d.iotda-device.cn-south-4.myhuaweicloud.com",
        "port": "1883",
        "tls": "",
        "clientid": "6a4df1276b6c4d5f8d6731e1_test_0_1_2026070811",
        "autoConnect": true,
        "usetls": false,
        "protocolVersion": "4",
        "keepalive": "60",
        "cleansession": true,
        "autoUnsubscribe": true,
        "birthTopic": "",
        "birthQos": "0",
        "birthRetain": "false",


        "birthPayload": "",
        "birthMsg": {},
        "closeTopic": "",
        "closeQos": "0",
        "closeRetain": "false",
        "closePayload": "",
        "closeMsg": {},
        "willTopic": "",
        "willQos": "0",
        "willRetain": "false",
        "willPayload": "",
        "willMsg": {},
        "userProps": "",
        "sessionExpiry": ""
    }
]
```

## 售后支持: 0755-29451836
