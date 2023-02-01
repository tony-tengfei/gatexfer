# 跨交易所资产划转

## 签名算法

### API 验签请求接口发送要求

+ 向客服人员申请一对API Key和API Secret, 并提供需要绑定的ip白名单
+ 在发送请求头部传入KEY,即API密钥对的 Key
+ 在发送请求头部传入Timestamp,即请求发送的时间,格式是秒级精度的 Unix 时间戳.同时该时间不能与当前时间差距超过 60 秒
+ 在发送请求头部传入SIGN,即将请求生成签名字符串并用 API Secret 加密后生成的签名
+ 加密算法为 HexEncode(HMAC_SHA512(secret, signature_string)),即通过 HMAC-SHA512 加密算法,将 API Secret 作为加密密钥，签名字符串作为加密消息， 生成加密结果的
  16 进制输出
+ 确保发送请求的客户端IP地址在所使用的密钥的IP地址白名单里

### 签名字符串生成方式

+ 签名字符串按照如下方式拼接生成 (Query String或Request Payload没有则用空字符串“”替代)

```
Request Method + "\n" + Request URL + "\n" + Query String + "\n" + HexEncode(SHA512(Request Payload)) + "\n" + Timestamp
```

### Request Method

+ 请求方法，全大写, 如 POST, GET

### Request URL

+ 不包括服务地址和端口，如 /api/spot/withdraw

### Query String

+ 没有使用 URL 编码的请求参数，请求参数在参与计算签名时的顺序一定要保证和实际请求里的顺序一致。 如 status=finished&limit=50

+ 如果没有请求参数，使用空字符串 ("")

### HexEncode(SHA512(Request Payload))

+ 将请求体字符串使用 SHA512 哈希之后的结果。如果没有请求体，使用空字符串的哈希结果，即
  cf83e1357eefb8bdf1542850d66d8007d620e4050b5715dc83f4a921d36ce9ce47d0d13c5d85f2b0ff8318d2877eec2f63b931bd47417a81a538327af927da3e

### Timestamp

+ 设置在请求头部 Timestamp 里的值 格式是秒级精度的 Unix 时间戳 数据类型为整型

## API 接口

### 划转接口

交易所名称需要大写: BINANCE, HUOBI, GATE, OKEX

```
请求
POST https://${url}/api/spot/withdraw
Content-Type: application/json
KEY: <api-key>
Timestamp: 1234567890
SIGN: <sign>

{
  "withdrawExchange": "BINANCE",
  "depositExchange": "GATE",
  "withdrawMainAccountId": "",
  "withdrawSubAccountId": "binance@gatexfer.com",
  "depositMainAccountId": "",
  "depositSubAccountId": "123456789",
  "currency": "usdt",
  "amount": 100000
}

返回
{  
  "code": 代码
  "data": 请求成功 返回任务ID 用来查询状态
  "msg": 请求成功返回success 请求失败返回错误信息
}
```
>充提的母账户&子账户ID 每个请求需要传两个 规则如下

| 提现账户类型 | 充值账户类型 | 传参                                            |
|--------|--------|-----------------------------------------------|
| 母账户    | 子账户    | withdrawMainAccountId + depositSubAccountId   |
| 母账户    | 母账户    | withdrawMainAccountId + depositMainAccountId  |
| 子账户    | 子账户    | withdrawSubAccountId +  depositSubAccountId   |
| 子账户    | 母账户    | withdrawSubAccountId +   depositMainAccountId |


### 请求状态查询

```
请求
GET https://${url}/api/spot/withdraw/{id}

失败的划转任务查询返回 
{
  "code": 0,
  "data": {
    "id": "606e037ab0e57",
    "status": "-4",
    "msg": "error processing withdraw:User has insufficient balance",
    "txId": ""
  },
  "msg": "success"
}
成功的划转任务查询返回
{
  "code": 0,
  "data": {
    "id": "c0dbe274c2a58",
    "status": "9",
    "txId": "0x12345678987654321abcdefghijklmnoqprstuvwxyz",
    "msg": "task completed successfully"
  },
  "msg": "success"
}
```
状态列表
|值|描述|
|--|---|
|-1|失败|
|-2|提现账户的内部划转失败|
|-4|主账户提现失败|
|-7|充值账户的内部划转失败|
|1|新建请求|
|2|提现账户的内部划转请求已发出|
|3|提现账户的内部划转已完成|
|4|主账户的提现请求已发出|
|5|链上转账进行中|
|6|主账户的提现已完成|
|7|充值账户的内部划转请求已发出|
|8|充值账户的内部划转已完成|
|9|任务成功完成|
|0|已取消|

### 系统时间接口

```
请求
GET https://${url}/api/public/ping
```

### 返回格式

```
所有HTTP请求都会按照以下格式返回数据

{  
  "code": 代码
  "data": 数据
  "msg": 信息
}
```

## sub-account-id 说明

| Exchange | SubAccountId        |
|----------|---------------------|
| BINANCE  | Sub-account email   |
| HUOBI    | Sub user’s UID      |
| GATE     | Sub account user ID |
| OKEX     | sub-account name    |
