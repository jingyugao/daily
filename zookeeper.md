## leader election
基本概念：
1. leader:zab协议中的leader属于管理leader，不储存数据，只负责发起proposal和ack。而raft的leader则储存数据。
2. zxid:逻辑时间戳，高32位为epoch id，相当于raft的term。低32位是自增id，更换leader后从0开始，而raft的index不会归零。


## 状态机
每个进程都处于 *LOOKING* , *FOLLOWING* , *LEADING* 阶段之一。
1. 初始状态所有进程处于 *LOOKING* 阶段，
2. 选举定时器溢出后，开始进行选举。
 1. 发起投票，包含(zxid,sid),第一轮投自己。
 2. 等待反馈。如果对面的(zxid,sid)比自己“大”，投别人。
 3. 多轮以后
 

