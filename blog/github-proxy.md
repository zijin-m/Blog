# ssh 协议代理

```bash
Host github.com
    User git
    ProxyCommand nc -X5 -x 127.0.0.1:1086 %h %p
```
