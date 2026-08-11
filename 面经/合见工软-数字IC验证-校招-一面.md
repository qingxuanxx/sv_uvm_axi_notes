# 合见工软 IC验证岗 一面面经

> 来源：小红书 | base 上海 | 仅一轮技术面
>
> 面试官提问质量较高，很多问题回答不到位——本质是对知识点理解深度不足。

【一、验证实践】

- 模块级验证是否考虑注销逻辑
- 自身核心工作内容
- 验证点的设计思路

【二、AXI 协议】

- outstanding 的定义与验证方法
- outstanding 数量变化时是否需要新增测试用例
- VIP 内部如何实现支持不同 burst_size、burst_length 的 sequence

【三、ECC】

- ECC 的检错与纠错原理

【四、SystemVerilog】

| 问题 |
|------|
| packed 数组与 unpacked 数组的核心区别 |
| 动态数组的使用优势 |
| 函数重载（function overloading）的实现方式 |
| 枚举类型除变量索引外的其他索引方式 |

【五、UVM】

| 问题 |
|------|
| config_db 完成 set 后，若源变量发生变化，get 到的值是否同步更新 |
| virtual sequence 与物理接口的连接方法 |
| `uvm_info` 的消息等级分类 |
| config_db 中 set/get 操作的层级匹配规则 |

【关键收获】

- ECC 检错纠错是 EDA 公司高频考点
- config_db 源变量变化后 get 值**不同步更新**——set 是值拷贝
- packed vs unpacked、枚举索引、函数重载都是容易被追问深度的基础题
