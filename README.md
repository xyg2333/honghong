## 路径示例

路径统一为 `/{任意字母数字}=代理`（如 `/fdip=...`，`fdip` 可换成任意字母数字组合），`ed=2560` 放最后。

| 用途 | 路径 | worker.js | snippet.js | https.js | lite.js |
| --- | --- | :---: | :---: | :---: | :---: |
| 直连 / proxyip | `/fdip=1.2.3.4:443?ed=2560` | ✓ | ✓ | ✓ | ✓ |
| TXT 记录 | `/fdip=domain!txt?ed=2560` | ✓ | ✓ | ✓ | ✓ |
| socks5 | `/fdip=socks5://host:port?ed=2560` | ✓ | ✓ | — | — |
| http | `/fdip=http://host:port?ed=2560` | ✓ | ✓ | — | — |
| https（域名） | `/fdip=https://domain:port?ed=2560` | ✓ | ✓ | ✓ | — |
| https（IP） | `/fdip=https://ip:port?ed=2560` | ✓ | — | ✓ | — |
| https（强制 TlsClient） | `/fdip=https://host:port!ip?ed=2560` | ✓ | — | ✓ | — |
| sstp | `/fdip=sstp://host:port?ed=2560` | ✓ | ✓ | — | — |
| turn | `/fdip=turn://host:port?ed=2560` | ✓ | ✓ | — | — |
| turns（域名） | `/fdip=turns://domain:port?ed=2560` | ✓ | ✓ | — | — |
| turns（IP / `!ip`） | `/fdip=turns://ip:port?ed=2560` | ✓ | — | — | — |
| global | `/fdip={proxy}?global=1&ed=2560` | ✓ | ✓ | ✓ | — |
| auto | `/?auto=1&ed=2560`、`/fdip={proxy}?auto=1&ed=2560` | ✓ | ✓ | ✓ | ✓ |

---
## 节点示例

**Vless ws**
```ws
vless://495c7195-85b8-498a-bf20-2ea9ce9175b5@www.shopify.com:443?path=%2Ffdip%3D1.2.3.4%3A443%3Fed%3D2560&security=tls&encryption=none&insecure=0&host=vless.snippets.cf&fp=chrome&type=ws&allowInsecure=0&sni=vless.snippets.cf#ws
```

**Trojan ws**
```ws
trojan://495c7195-85b8-498a-bf20-2ea9ce9175b5@www.shopify.com:443?path=%2Ffdip%3D1.2.3.4.%3A443%3Fed%3D2560&security=tls&insecure=0&host=trojan.snippet.cf&fp=chrome&type=ws&allowInsecure=0&sni=trojan.snippet.cf#ws
```

**SS(notls) ws**
```ws
ss://YWVzLTEyOC1nY206NDk1YzcxOTUtODViOC00OThhLWJmMjAtMmVhOWNlOTE3NWI1@www.shopify.com:80?plugin=v2ray-plugin%3Bmode%3Dwebsocket%3Bhost%3Dss.snippets.cf%3Bpath%3D%2Ffdip%3D1.2.3.4%3A443%3Fed%3D2560%3Bmux%3D0#ws
```
<details>
<summary>xhttp extra（留空 或 填入以下内容 或 自行配置，效果自测）</summary>

```json
{
"extra": {
  "noGRPCHeader": true,
  "headers": {
    "Content-Type": "application/octet-stream"
  },
  "xPaddingBytes": "100-1000",
  "xPaddingObfsMode": true,
  "xPaddingMethod": "tokenish",
  "xPaddingPlacement": "queryInHeader",
  "xPaddingHeader": "X-Cache",
  "xPaddingKey": "_dc"
}
}
```
</details>

---
