# 多进程Go游戏服务器
##网络拓扑关系纯配置控制

##简介

1、核心的tcp、http是项目正在用的，效果还可以，虽然还没被外网玩家操过

2、在此之上封了一层netConfig，便于网络连接关系的管理

3、支持崩溃重启

	(1)【1-1】关系中的"client"重启：game每次均会连接battle

	(2)【1-1】关系中的"server"重启：battle(tcp)重启，game的client.ConnectToSvr能检查到失败，循环重连

	(3)【1-N】关系中的"N"重启：     game每次均会去sdk注册

	(4)【1-N】关系中的"1"重启：     "http_server.go"会本地存储注册地址，重启时载入

4、使用方便

[config_net.go](https://github.com/3workman/Sundry/tree/master/go/src/netConfig/config_net.go)
--------------

	(1)"conf_net.csv"中非常容易指定连接方式
| module  | ip        | out_ip | tcp_port | http_port | max_conn | svr_id | connect |
| ------- | --------- | ------ | -------- | --------- | -------- | ------ | ------- |
| account | 127.0.0.1 |        | 7001     |           | 5000     |        |         |
| sdk     | 127.0.0.1 |        |          | 7002      |          | 1      |         |
| game    | 127.0.0.1 |        |          | 7010      |          | 1      | sdk/battle |
| chat    | 127.0.0.1 |        | 7020     |           | 5000     |        |         |
| battle  | 127.0.0.1 |        | 7030     |           | 5000     | 1      |         |
| battle  | 127.0.0.1 |        | 7031     |           | 5000     | 2      |         |
| client  |           |        |          |           | 5000     |        | game/sdk/battle |

	
	(2)统一了连接获取方式，业务层使用，只需加几个Cache接口即可
```go
var (
	g_cache_battle_conn *tcp.TCPConn
)
func SendToBattle(msgID uint16, msgdata []byte) {
	if g_cache_battle_conn == nil {
		g_cache_battle_conn = netConfig.GetTcpConn("battle", 0)
	}
	g_cache_battle_conn.WriteMsg(msgID, msgdata)
}
```
	
	(3)构建一个新的服务进程，配置完毕后，只两行代码+SendToModule接口就够啦
```go
func main() {
	//注册所有tcp消息处理方法
	RegBattleTcpMsgHandler()

	gamelog.Warn("----Battle Server Start-----")
	if netConfig.CreateNetSvr("battle") == false {
		gamelog.Error("----Battle NetSvr Failed-----")
	}
}
```

5、目前只配了game、battle、sdk、client网络模块，编译后运行bin目录的start_svr.bat可验证测试