原因：

switch 的底层是使用 int 类型来进行判断，即便是枚举、String 类型、最终也是转变成 int 类型由于 long 类型表示范围大于 int 的区间，所以不支持。