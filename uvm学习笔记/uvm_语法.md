# UVM 语法

> UVM 常用 API 与宏语法速查（持续更新）。
>
> 阅读配套：`uvm_ch1~ch9学习笔记.md`（UVM 实战学习笔记）

## 目录

（待积累，按下面方向慢慢补充）

## 待积累方向

- [ ] 注册宏：`uvm_object_utils` / `uvm_component_utils` / begin-end 版 / param 版
- [ ] 字段宏：`uvm_field_int` / `_enum` / `_string` / `_array_*` / `_sarray_*` / `_queue_*` / `_aa_*`
- [ ] 日志宏：`uvm_info` / `uvm_warning` / `uvm_error` / `uvm_fatal`（参数与 verbosity）
- [ ] config_db：set / get / wait_modified、类型与路径匹配
- [ ] 创建：`type_id::create`（object 与 component 两种形式）
- [ ] phase：build / connect / run-time / report 等回调签名
- [ ] objection：raise_objection / drop_objection / set_drain_time
- [ ] TLM 端口：put/get/transport 的 port/export/imp、analysis_port、FIFO
- [ ] sequence：`uvm_do` / `uvm_do_with` / start_item / finish_item / get_next_item / item_done
- [ ] factory：set_type_override_by_type / set_inst_override_by_type / print
- [ ] 寄存器模型：reg/field/block/map 的 configure、read/write/peek/poke、adapter
- [ ] 命令行参数：+UVM_TEST_NAME / +UVM_VERBOSITY / +UVM_PHASE_TRACE 等

---

> 积累日期：____-__-__
