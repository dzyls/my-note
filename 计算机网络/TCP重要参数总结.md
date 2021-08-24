### TCP参数

---

- `tcp_syn_retries` ：syn报文的尝试次数 【5】
- `tcp_synack_retries` ：synack的尝试次数 【5】
- `tcp_retries1` ：放弃回应tcp请求前的次数 【3】
- `tcp_retries2`：丢弃一个TCP需要重试多少次 【15】
- `tcp_keepalive_time` ：保活机制的keeptime的频率
- `tcp_keepalive_probes` ：无响应时探测报文的重试次数【9】
- `tcp_keepalive_intvl` ：无响应发送探测报文的间隔【75s】
- `tcp_tw_max_buckets` ：tw的最大数量，多了丢弃
- `tcp_tw_reuse` ：根据时间戳重用tw的socket
- `tcp_syncookies` : 三次握手的cookies，此参数应该打开，一是避免了三次握手，二是防止SYN Flood
- `tcp_fastopen` ：第一次连接生成cookies，后续再连接直接传cookies避免三次握手
- `tcp_sack` ：是否打开sack

