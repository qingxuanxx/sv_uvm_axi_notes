# 行云集成电路-数字IC验证-实习-面试记录

> 来源：小红书 ｜ 1月22日面试，1月23日收到 OC，最终拒绝；时长 50 分钟，HR 和面试官体验很好，问题具备引导性。

## 面试流程

- 自我介绍
- 是否了解闩锁效应、与非门的 MOS 管搭建结构

## 项目提问

两个项目。

| 项目 | 考点 |
|------|------|
| 小型项目 | DUT 和验证结构、接口协议、接收端设置 sequencer 的原因、DUT 上游反压的实现方式、FIFO full 信号保障不丢数据的原理、转发正确性的验证方法、预留问题 |
| AXI 接口 IP 项目 | DUT 与所用接口、验证结果、agent 是否使用 VIP、多包区分数据包源头的方法、测试流程、是否使用 monitor 抓包 |

## 八股

- task phase 和 function phase 的理解、属于 task phase 的类型
- build 和 connect phase 的顺序
- run phase 和 12 个子 phase 的关系，能否同时使用
- 面向对象多态的理解、static 的用法
- fork join / join_any / join_none 三者区别
- axi4 对比 axi3 的差异、outstanding / 乱序 / 交织的定义、axi4 是否支持乱序与交织
- axi-full 和 axi-lite 的区别，axi-lite 相比 full 缺少的信号，axi-lite 是否支持 outstanding / 乱序 / 交织，axi-lite 是否存在 wid 信号
- 不同 id 与同 id 场景下协议的保序要求

## 反问

- 实习生培养模式、项目方向、面试优化建议
