---
description: Game packet structures
---

# 报文结构

Game packets are composed of 3 distinct parts, the packet header, the segment header and a (sometimes) optional IPC header.

{% hint style="info" %}
虽然“IPC”是不正确的术语，但SE称其为IPC，因此保留了该名称。
{% endhint %}

## 报文头 Packet Header

```cpp
struct FFXIVARR_PACKET_HEADER
{
  uint64_t magic[2];
  uint64_t timestamp;
  uint32_t size;
  uint16_t connectionType;
  uint16_t segmentCount;
  uint8_t unknown_20;
  uint8_t isCompressed;
  uint32_t unknown_24;
};
```

| Field            | Description                                                                                                                                                                                                               |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `magic`          | <p>A magic value that identifies a packet.</p><p>This is <code>FF14ARR</code> if you read it both in it's hexadecimal and ascii representation at the same time.</p>                                                      |
| `timestamp`      | The number of milliseconds since the unix epoch when the packet was sent                                                                                                                                                  |
| `size`           | The size of the entire packet including its segments and data                                                                                                                                                             |
| `connectionType` | <p>The connection type. This will be 1 for zone channel connections and 2 for chat.</p><p>This is only sent on the initial connection now, previously this was sent with every packet but that is no longer the case.</p> |
| `unknown_20`     | Alignment most likely                                                                                                                                                                                                     |
| `isCompressed`   | Segments以及之后的数据是否被压缩。 该字段有三种取值，0 - 未压缩，1 - zlib，2 - oodles.                                                                                                                                                               |
| `unknown_24`     | Alignment                                                                                                                                                                                                                 |

## 段落头 Segment Header

```cpp
struct FFXIVARR_PACKET_SEGMENT_HEADER
{
  uint32_t size;
  uint32_t source_actor;
  uint32_t target_actor;
  uint16_t type;
  uint16_t padding;
};
```

| Field          | Description                                                                                                                                                                                                              |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `size`         | The size of this segment and its data (if any)                                                                                                                                                                           |
| `source_actor` | <p>The actor id of the actor who effectively caused this packet to be sent.</p><p>For example, if another player casts an action, the <code>source_actor</code> field will contain their actor id</p>                    |
| `target_actor` | <p>The actor id of the actor who is affected by the packet</p><p>This isn't used consistently, but the same <em>logical</em> rules apply as <code>source_actor</code>.</p>                                               |
| `type`         | <p>The type of segment, see <a href="packet-structure.md#segment-types">below for more detail</a>.</p><p>Based on the value of this field indicates what kind of data you'd expect to find after the segment header.</p> |
| `padding`      | Alignment                                                                                                                                                                                                                |

### 段类型 Segment Types

```cpp
enum FFXIVARR_SEGMENT_TYPE
{
  SEGMENTTYPE_SESSIONINIT = 1,
  SEGMENTTYPE_IPC = 3,
  SEGMENTTYPE_KEEPALIVE = 7,
  //SEGMENTTYPE_RESPONSE = 8,
  SEGMENTTYPE_ENCRYPTIONINIT = 9,
};
```

| Type                         | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `SEGMENTTYPE_SESSIONINIT`    | <p>Used to login to a world or chat server.</p><p>The packet that has a segment that has a type set to this will contain a correct <code>connectionType</code> set in the <a href="packet-structure.md#packet-header">packet header</a>. Use this to record what kind of connection it is.</p><p>Example implementation is in <a href="https://github.com/SapphireServer/Sapphire/blob/399d9b0dcd6d87aa624bd53c58415efa27bb1b1c/src/world/Network/GameConnection.cpp#L430-L452">Sapphire</a>.</p> |
| `SEGMENTTYPE_IPC`            | <p>Used for segments that contain data that should be handled by the packet router for the associated channel.</p><p>Chat messages, using actions and etc. will always be sent via this segment type and there will be a <code>FFXIVARR_IPC_HEADER</code> immediately following the segment header.</p>                                                                                                                                                                                           |
| `SEGMENTTYPE_KEEPALIVE`      | <p>to-do: can't remember where this is actually used - lobby?</p><p>As a note, world (and chat?) use IPCs to ping/pong. Because reasons.</p>                                                                                                                                                                                                                                                                                                                                                      |
| `SEGMENTTYPE_ENCRYPTIONINIT` | <p>Used to initialise blowfish for the <a href="channels.md#lobby">lobby channel</a>.</p><p>The client sends a packet to lobby with it's key phrase which is then used to ''''''secure'''''' a lobby session.</p><p>Spoiler alert: it's not secure</p>                                                                                                                                                                                                                                            |

## IPC Header

Only present when the parent segment type is set to `SEGMENTTYPE_IPC`.

游戏与客户端通信的主要报文内容，使用 machina.ffxiv 或 dalamud 的 network 接口均提供的是 IPC 报文内容。操作码 opcode 在每次发布 patch 时会被打乱。

```cpp
struct FFXIVARR_IPC_HEADER
{
  uint16_t reserved;
  uint16_t type;
  uint16_t padding;
  uint16_t serverId;
  uint32_t timestamp;
  uint32_t padding1;
};
```

| Field       | Description                                                                                                                              |
| ----------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `reserved`  |                                                                                                                                          |
| `type`      | This will contain the opcode of the packet which identifies which handler the packet data following this packet should go.               |
| `padding`   | potentially data here and not padding but it's probably not important 🙃                                                                 |
| `serverId`  | to-do: write about retail server architecture                                                                                            |
| `timestamp` | <p>A Unix timestamp in seconds since the epoch.</p><p>Not really sure why this exists here but it does and that's what it has in it.</p> |
| `padding1`  | Alignment                                                                                                                                |

## 具体内容格式

待完成，FFXIV_ACT_Plugin 解析插件包含了大部分有关战斗的报文格式。

## Decoding Packets

Decoding packets is reasonably simple and assuming you have a buffer that you write data in to for each connection, it's something like the following:

```
if buf.size < sizeof(FFXIVARR_PACKET_HEADER):
    return
    
header = &buf[0] as FFXIVARR_PACKET_HEADER:

if buf.size < header.size:
    return

data = slice buf from sizeof(FFXIVARR_PACKET_HEADER) to end of buf

if buf.isCompressed:
    data = zlib_inflate(data)

pos = 0
while true:
    segment = &data[pos] as FFXIVARR_PACKET_SEGMENT_HEADER
    
    if segment.size >= buf.size
        burn them

    if segment.size >= data.size:
        also burn them    
            
    pos = segment.size
    
    seg_hdr_size = sizeof(FFXIVARR_PACKET_SEGMENT_HEADER)
    
    if segment.type == SEGMENTTYPE_IPC:
        ipc_size = segment.size - seg_hdr_size
        ipc_data = slice segment from seg_hdr_size to ipc_size
        
        ipc_hdr = &ipc_data[0] as FFXIVARR_IPC_HEADER
        ipc_hdr_size = sizeof(FFXIVARR_IPC_HEADER)
        
        packet_data = slice ipc_data from ipc_hdr_size to remaining size
        process_channel_packet(ipc_hdr.type, packet_data)
        
    // other segment types depend on the type of channel, but it's more of the same
```

A lot of detail is omitted for brevity, but it's generally pretty straightforward.

A more comprehensive example of packet parsing can be found in Sapphire:

* [Packet container](https://github.com/SapphireServer/Sapphire/blob/develop/src/common/Network/GamePacket.h) (and some container-level parsing)
* [Packet parsing](https://github.com/SapphireServer/Sapphire/blob/develop/src/common/Network/GamePacketParser.cpp)
