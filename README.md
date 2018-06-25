# BP 安全部署方案

<!-- vim-markdown-toc GFM -->

* [部署原则](#部署原则)
* [推荐架构](#推荐架构)
    * [架构图](#架构图)
    * [架构说明](#架构说明)
    * [出块节点](#出块节点)
        * [屏蔽 RPC](#屏蔽-rpc)
        * [限制 P2P](#限制-p2p)
        * [其他](#其他)
    * [同步节点](#同步节点)
        * [config.ini](#configini)
* [参考](#参考)

<!-- vim-markdown-toc -->

## 部署原则

- 隔离 BP 服务器
- 最小化开启功能插件
- 配置主从随时切换
- 避免单点故障

## 推荐架构

### 架构图

![bp_deploy](img/bp_deploy.jpg)

* 图示中的 AWS 可以替换成其他 VPS 供应商, 如: 阿里云, 腾讯云等.

* 冗余部署, 同时部署 Master 和 Slave 两套相同的方案, 分别部署到两个不同的机房。

* Slave BP 在 Master BP 正常出块的情况下, 配置为一个同步节点进行出块。

* 当 Master BP 出现问题(或者重启过程中)时, 切换为相应的出块节点.

### 架构说明

Slave 部署节点与 Master 部署节点, 每个对应的节点配置一致, 所以下面只讨论 Master 节点部署配置.

推荐使用 Docker 镜像 [eosforce/eos:v1.0](https://hub.docker.com/r/eosforce/eos/) 进行部署.

* Master BP 的 `p2p-peer-address` 地址只配置两个 p2p 地址，即与它相连的 Public Node 和 Secret Node.

* Public Node 用于与其他 BP 的 p2p 地址相连, 并且其 P2P 地址对外公开, 外部用户可以搭建同步节点与该地址连接进行同步区块.

* Secret Node 只用于与其他 BP 的 P2P 地址相连, IP 及端口号不对外公布, 不提供给外部用户的同步节点进行连接同步区块.

* Public Node 和 Secret Node 均至少各两台, 避免出现单点故障问题.

* Public Node 的 RPC 端口不直接对外提供服务, 而是使用 resty-gate 做代理, resty-gate 为一个基于 OpenResty 开发的项目, 用于对 RPC 端口做 HTTPS 代理, 限流, 限速, 防御等.

### 出块节点

#### 屏蔽 RPC

将 `http-server-address =` 置为空禁用 RPC。

```ini
http-server-address = # 空表示禁用
```

#### 限制 P2P

```ini
p2p-listen-endpoint = <inbound ip>:9876 # 只允许自己的同步节点连接, 实际部署时换成其他随机端口.
p2p-peer-address = xx.xx.xx.xx:9876 # 自己的 BP 的同步节点, 实际部署时换成其他随机端口.
```

#### 其他

```ini
enable-stale-production = false
max-clients = <同步节点数*2> # 即 (Public Node + Secret Node)*2
producer-name = <出块账户名称>
signature-provider = <pubkey>=KEY:<privkey>
```

### 同步节点

#### config.ini

```ini
filter-on = *
http-server-address = 127.0.0.1:8888 #  RPC 端口监听在本地, 使用代理对外提供服务.
access-control-allow-origin = *
access-control-allow-headers = Content-Type
access-control-allow-credentials = false
p2p-listen-endpoint = 0.0.0.0:9876 # 实际部署时换成其他随机端口.
p2p-server-address = <public ip>:9876 # 实际部署时换成其他随机端口.
p2p-peer-address = xx.xx.xx.xx:9876 # 自己的 BP 以及其他 BP 的同步节点
max-clients = 25 # 可根据自己机器配置情况来调大, 但不建议过大.
get-actions-on=true
plugin = eosio::chain_api_plugin
plugin = eosio::history_plugin
plugin = eosio::history_api_plugin
plugin = eosio::faucet_testnet_plugin # 如果想提供 faucet 服务, 可以进行下面配置, 否则不填. 该值可根据自己的情况调大来防止恶意创建账号.
faucet-create-interval-ms = 1000
faucet-name = account_name
faucet-private-key = ["pubkey", "privkey"]
```

## 参考

- https://github.com/slowmist/eos-bp-nodes-security-checklist
