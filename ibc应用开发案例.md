**1.**   用ignite生成一条新的区块链名字叫planet。

```
ignite scaffold chain planet --no-module
```

**2.**  使用ignite生成一个Blog的模块，并且集成IBC。
cd planet

```
ignite scaffold module blog --ibc
```

**3.** 给blog模块添加针对博文（post）的增删改查。

```
ignite scaffold list post title content creator --no-message --module blog

```

**4.** 添加已发生成功博文（sentPost）的增删改查。

```
ignite scaffold list sentPost postID title chain creator --no-message --module blog
```

**5.** 添加发送超时博文（timeoutPost）的增删改查。

```
ignite scaffold list timedoutPost title chain creator --no-message --module blog
```

**6.** 添加IBC发送数据包和确认数据包的结构。

```
ignite scaffold packet ibcPost title content --ack postID --module blog

```
  
**7.** 在proto/blog/packet.proto目录下修改`IbcPostPacketData`，添加创建人`Creator`， 并重新编译proto文件。在x/blog/keeper/msg_server_ibc_post.go。编译完成后在x/blog/keeper/msg_server_ibc_post.go中发送数据包前更新`Creator`。

```
ignite chain build
```

**8.** 修改keeper方法中的`OnRecvIbcPostPacket `。

```
id := k.AppendPost(
        ctx,
        types.Post{
            Creator: packet.SourcePort + "-" + packet.SourceChannel + "-" + data.Creator,
            Title:   data.Title,
            Content: data.Content,
        },
    )

    packetAck.PostID = strconv.FormatUint(id, 10)
```

**9.** 修改keeper方法中的`OnAcknowledgementIbcPostPacket `。

```
k.AppendSentPost(
            ctx,
            types.SentPost{
                Creator: data.Creator,
                PostID:  packetAck.PostID,
                Title:   data.Title,
                Chain:   packet.DestinationPort + "-" + packet.DestinationChannel,
            },
        )
```

**10.** 修改keeper方法中的`OnTimeoutIbcPostPacket `。

```
k.AppendTimedoutPost(
        ctx,
        types.TimedoutPost{
            Creator: data.Creator,
            Title:   data.Title,
            Chain:   packet.DestinationPort + "-" + packet.DestinationChannel,
        },
    )
```

**11.** 添加链启动的配置文件。

```
# earth.yml
accounts:
  - name: alice
    coins: ["1000token", "100000000stake"]
  - name: bob
    coins: ["500token", "100000000stake"]
validator:
  name: alice
  staked: "100000000stake"
faucet:
  name: bob
  coins: ["5token", "100000stake"]
genesis:
  chain_id: "earth"
init:
  home: "$HOME/.earth"
  
# mars.yml
accounts:
  - name: alice
    coins: ["1000token", "1000000000stake"]
  - name: bob
    coins: ["500token", "100000000stake"]
validator:
  name: alice
  staked: "100000000stake"
faucet:
  host: ":4501"
  name: bob
  coins: ["5token", "100000stake"]
host:
  rpc: ":26659"
  p2p: ":26658"
  prof: ":6061"
  grpc: ":9092"
  grpc-web: ":9093"
  api: ":1318"
genesis:
  chain_id: "mars"
init:
  home: "$HOME/.mars"
```


**12.** 分别启动两条链

```
ignite chain serve -c earth.yml

ignite chain serve -c mars.yml
```

**13.** 启动中继器

```
rm -rf ~/.ignite/relayer

ignite relayer configure -a \
  --source-rpc "http://0.0.0.0:26657" \
  --source-faucet "http://0.0.0.0:4500" \
  --source-port "blog" \
  --source-version "blog-1" \
  --source-gasprice "0.0000025stake" \
  --source-prefix "cosmos" \
  --source-gaslimit 300000 \
  --target-rpc "http://0.0.0.0:26659" \
  --target-faucet "http://0.0.0.0:4501" \
  --target-port "blog" \
  --target-version "blog-1" \
  --target-gasprice "0.0000025stake" \
  --target-prefix "cosmos" \
  --target-gaslimit 300000

ignite relayer connect
```
```
li@liyihangs-MBP ignite-test % ignite relayer configure -a \
  --source-rpc "http://0.0.0.0:26657" \
  --source-faucet "http://0.0.0.0:4500" \
  --source-port "blog" \
  --source-version "blog-1" \
  --source-gasprice "0.0000025stake" \
  --source-prefix "cosmos" \
  --source-gaslimit 300000 \
  --target-rpc "http://0.0.0.0:26659" \
  --target-faucet "http://0.0.0.0:4501" \
  --target-port "blog" \
  --target-version "blog-1" \
  --target-gasprice "0.0000025stake" \
  --target-prefix "cosmos" \
  --target-gaslimit 300000
------
Setting up chains
------

? Source Account default
? Target Account default

🔐  Account on "source" is default(cosmos1asxy8tzn2dvudlq6a4zk7zu77cxfnle39ptnpe)
 
 |· received coins from a faucet
 |· (balance: 100000stake,5token)

🔐  Account on "target" is default(cosmos1asxy8tzn2dvudlq6a4zk7zu77cxfnle39ptnpe)
 
 |· received coins from a faucet
 |· (balance: 100000stake,5token)

⛓  Configured chains: earth-mars

li@liyihangs-MBP ignite-test % 
ignite relayer connect
------
Paths
------

earth-mars:
    earth > (port: blog) (channel: channel-0)
    mars  > (port: blog) (channel: channel-0)

------
Listening and relaying packets between chains...
------

Relay 1 packets from earth => mars
Relay 1 acks from mars => earth
```

**14.** 从earth链向mars链发送博文数据包（注意修改channel id）

```
planetd tx blog send-ibc-post blog channel-0 "Hello" "Hello Mars, I'm Alice from Earth" --from alice --chain-id earth --home ~/.earth
```

```
li@liyihangs-MBP planetd % planetd tx blog send-ibc-post blog channel-0 "Hello" "Hello Mars, I'm Alice from Earth" --from alice --chain-id earth --home ~/.earth
auth_info:
  fee:
    amount: []
    gas_limit: "200000"
    granter: ""
    payer: ""
  signer_infos: []
  tip: null
body:
  extension_options: []
  memo: ""
  messages:
  - '@type': /planet.blog.MsgSendIbcPost
    channelID: channel-0
    content: Hello Mars, I'm Alice from Earth
    creator: cosmos1n0zafqz3x9hupt8u43r6a2pkcgplcza85pxjq0
    port: blog
    timeoutTimestamp: "1672494153594654000"
    title: Hello
  non_critical_extension_options: []
  timeout_height: "0"
signatures: []
confirm transaction before signing and broadcasting [y/N]: y
code: 0
codespace: ""
data: 12250A232F706C616E65742E626C6F672E4D736753656E64496263506F7374526573706F6E7365
events:
- attributes:
  - index: true
    key: ZmVl
    value: ""
  - index: true
    key: ZmVlX3BheWVy
    value: Y29zbW9zMW4wemFmcXozeDlodXB0OHU0M3I2YTJwa2NncGxjemE4NXB4anEw
  type: tx
- attributes:
  - index: true
    key: YWNjX3NlcQ==
    value: Y29zbW9zMW4wemFmcXozeDlodXB0OHU0M3I2YTJwa2NncGxjemE4NXB4anEwLzE=
  type: tx
- attributes:
  - index: true
    key: c2lnbmF0dXJl
    value: UEFCRk1SWXBmVERKRWF0VkVjY2NZNGR1c2dIbFZpU1kzUGlLbkdKQlpYWTEvQzE5dExmQ1RYQ3BVTXZ3ZkFuS1ZycEhFQSt1ZWxmZGhGcms4dDZCYWc9PQ==
  type: tx
- attributes:
  - index: true
    key: YWN0aW9u
    value: L3BsYW5ldC5ibG9nLk1zZ1NlbmRJYmNQb3N0
  type: message
- attributes:
  - index: true
    key: cGFja2V0X2RhdGE=
    value: EikKBUhlbGxvEiBIZWxsbyBNYXJzLCBJJ20gQWxpY2UgZnJvbSBFYXJ0aA==
  - index: true
    key: cGFja2V0X2RhdGFfaGV4
    value: MTIyOTBhMDU0ODY1NmM2YzZmMTIyMDQ4NjU2YzZjNmYyMDRkNjE3MjczMmMyMDQ5Mjc2ZDIwNDE2YzY5NjM2NTIwNjY3MjZmNmQyMDQ1NjE3Mjc0Njg=
  - index: true
    key: cGFja2V0X3RpbWVvdXRfaGVpZ2h0
    value: MC0w
  - index: true
    key: cGFja2V0X3RpbWVvdXRfdGltZXN0YW1w
    value: MTY3MjQ5NDE1MzU5NDY1NDAwMA==
  - index: true
    key: cGFja2V0X3NlcXVlbmNl
    value: MQ==
  - index: true
    key: cGFja2V0X3NyY19wb3J0
    value: YmxvZw==
  - index: true
    key: cGFja2V0X3NyY19jaGFubmVs
    value: Y2hhbm5lbC0w
  - index: true
    key: cGFja2V0X2RzdF9wb3J0
    value: YmxvZw==
  - index: true
    key: cGFja2V0X2RzdF9jaGFubmVs
    value: Y2hhbm5lbC0w
  - index: true
    key: cGFja2V0X2NoYW5uZWxfb3JkZXJpbmc=
    value: T1JERVJfVU5PUkRFUkVE
  - index: true
    key: cGFja2V0X2Nvbm5lY3Rpb24=
    value: Y29ubmVjdGlvbi0w
  type: send_packet
- attributes:
  - index: true
    key: bW9kdWxl
    value: aWJjX2NoYW5uZWw=
  type: message
gas_used: "65347"
gas_wanted: "200000"
height: "782"
info: ""
logs:
- events:
  - attributes:
    - key: action
      value: /planet.blog.MsgSendIbcPost
    - key: module
      value: ibc_channel
    type: message
  - attributes:
    - key: packet_data
      value: "\x12)\n\x05Hello\x12 Hello Mars, I'm Alice from Earth"
    - key: packet_data_hex
      value: 12290a0548656c6c6f122048656c6c6f204d6172732c2049276d20416c6963652066726f6d204561727468
    - key: packet_timeout_height
      value: 0-0
    - key: packet_timeout_timestamp
      value: "1672494153594654000"
    - key: packet_sequence
      value: "1"
    - key: packet_src_port
      value: blog
    - key: packet_src_channel
      value: channel-0
    - key: packet_dst_port
      value: blog
    - key: packet_dst_channel
      value: channel-0
    - key: packet_channel_ordering
      value: ORDER_UNORDERED
    - key: packet_connection
      value: connection-0
    type: send_packet
  log: ""
  msg_index: 0
raw_log: '[{"msg_index":0,"events":[{"type":"message","attributes":[{"key":"action","value":"/planet.blog.MsgSendIbcPost"},{"key":"module","value":"ibc_channel"}]},{"type":"send_packet","attributes":[{"key":"packet_data","value":"\u0012)\n\u0005Hello\u0012
  Hello Mars, I''m Alice from Earth"},{"key":"packet_data_hex","value":"12290a0548656c6c6f122048656c6c6f204d6172732c2049276d20416c6963652066726f6d204561727468"},{"key":"packet_timeout_height","value":"0-0"},{"key":"packet_timeout_timestamp","value":"1672494153594654000"},{"key":"packet_sequence","value":"1"},{"key":"packet_src_port","value":"blog"},{"key":"packet_src_channel","value":"channel-0"},{"key":"packet_dst_port","value":"blog"},{"key":"packet_dst_channel","value":"channel-0"},{"key":"packet_channel_ordering","value":"ORDER_UNORDERED"},{"key":"packet_connection","value":"connection-0"}]}]}]'
timestamp: ""
tx: null
txhash: 8A62194A6032FBD33D2B36386816204E61DE9A7CF2B1FAB5CF70B1F81848C99D
```
**15.** 通过rpc查询验证结果。

```
cd cmd/planetd/
go build

planetd q blog list-post --node tcp://localhost:26659

planetd q blog list-sent-post
```

```
li@liyihangs-MBP planetd % planetd q blog list-post --node tcp://localhost:26659
Post:
- content: Hello Mars, I'm Alice from Earth
  creator: blog-channel-0-
  id: "0"
  title: Hello
pagination:
  next_key: null
  total: "0"
li@liyihangs-MBP planetd % planetd q blog list-sent-post
SentPost:
- chain: blog-channel-0
  creator: ""
  id: "0"
  postID: "0"
  title: Hello
pagination:
  next_key: null
  total: "0"
```