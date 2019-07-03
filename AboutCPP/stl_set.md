# STL-SET

---

容器set存储的是独立的值。一旦值被放入到set中，该值将不能再被修改(相当于const value)。

set有两类——set和unorder_set。前者有利于在一定范围内的遍历，而后者方便随机的查询。

一般set使用二分查找树实现，而unorder_set使用哈希桶完成。

set需要value能够提供operator <用于比较不同的值的存放位置。而unorder_set需要提供hash方法用于不同值的放置。