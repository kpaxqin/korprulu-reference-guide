# 表单

表单在后台管理类系统中非常常见，同时也是复杂度的重灾区，仅一个校验，就有单字段校验、多字段联动、联合校验、异步校验、blur校验、submit校验等种种细分场景。使用local state或自行封装的方案，不仅重复工作量大，更容易在抽象时忽略某些场景导致踩坑。

更好的选择是寻找社区优秀的方案，本项目中我们选择了[redux-form](redux-form.com)，它已经迭代到第6个版本，维护相对活跃，对各种场景都有相对完善的考虑。

同时它的[示例](http://redux-form.com/6.7.0/examples/)也比较完善，常见的场景都有覆盖，在此就没必要赘述了。

以下三个示例比较典型，需要重点关注：

* 同步校验: http://redux-form.com/6.7.0/examples/syncValidation/
* 异步校验: http://redux-form.com/6.7.0/examples/asyncValidation/
* 动态表单: http://redux-form.com/6.7.0/examples/fieldArrays/