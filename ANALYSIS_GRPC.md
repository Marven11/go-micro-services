# gRPC通信分析

internal/services/frontend/frontend.go会将search.NearbyRequest通过TCP发送

提取TCP流到./search_nearby_request.bin

```xxd
00000000: 0000 2f01 0400 0000 0f83 86c9 c8c7 c6c5  ../.............
00000010: 7ea6 6a58 db65 e1be 569e 706d 95db 8e4a  ~.jX.e..V.pm...J
00000020: 391d 69a0 36f0 9b68 1247 71a9 636d 9786  9.i.6..h.Gq.cm..
00000030: f95a 79c1 b657 6e07 0000 2700 0100 0000  .Zy..Wn...'.....
00000040: 0f00 0000 0022 0d7f 1917 4215 bcd6 f4c2  ....."....B.....
00000050: 1a0a 3230 3135 2d30 342d 3039 220a 3230  ..2015-04-09".20
00000060: 3135 2d30 342d 3130                      15-04-10
```

从0d7f开始为实际的protobuf数据，分别包含经纬度和开始结束时间

可是protobuf前面的数据又是什么？

追踪到.../go/pkg/mod/google.golang.org/grpc@v1.51.0/stream.go的prepareMsg函数

其输入m是传入的`search.NearbyRequest`，返回的payload和header(hdr)会直接被发送到transport中，这应该就是生成以上payload的函数

看到.../go/pkg/mod/google.golang.org/grpc@v1.51.0/rpc_util.go中的msgHeader函数，其会为数据生成5个字节的header

hdr[0]是压缩的标志
hdr[1..5]是消息长度

所以紧邻着0d7f的`00 0000 0022`就是这里的header，0x22就是后面protobuf数据的长度

这里提出一个假设: 实际的数据被包裹在了HTTP2的DATA frame中

验证一下：如果是的话，那前面的数据就是

- 长度字段: `0000 27`
- 类型字段: 00
- 标志字段: 01
- 保留位和流标识符: `0000 000f`

可以看到其中的长度正好比上方的0x22多了5个字节，符合验证

所以从`0000 2700`开始直到末尾是一个HTTP2的DATA frame，包含protobuf payload

既然是HTTP2的话，那DATA frame前面应该还有其他 frame，看看第一个frame:

- 长度: 0000 2f
- 类型: 01 - 是HEADERS frame
- 标志: 04 - END_HEADERS
- 保留位和流标识符: `0000 000f`

然后后面有47(0x2f)字节的数据

## 搜索服务的响应

再来看一下搜索服务响应的内容

```xxd
00000000: 0000 0408 0000 0000 0000 0000 2700 0008  ............'...
00000010: 0600 0000 0000 0204 1010 090e 0707       ..............
```

### 第一个帧

- 长度: 000004
- 类型: 08 - WINDOW_UPDATE
- 标志: 00
- 保留位和流标识符: `00 0000 00`
- 窗口增量: `00 0000 27`

### 第二个帧

- 长度: `00 0008`
- 类型: 06 - PUSH_PROMISE
- 标志: 00
- 保留位和流标识符: `0204 1010`

剩下8字节的数据 `090e 0707`

没有对应的protobuf数据
