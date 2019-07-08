首先判断是不是只有一个fiber，只有一个fiber的话就让q1等于这个值，然后q2克隆q1
如果是有俩个fiber，则q1等于当前实例的fiber.updateQueue，q2就等于alternate.updateQueue；
如果两个fiber都没有更新队列。则q1，q2都创建新的。
只有一个fiber有更新队列。克隆以创建一个新的。
俩个fiber都有更新队列。总之就是，q1和q2都需要有一个fiber。



当q1与q2是相等时，一位置实际上只有一个fiber，将此fiber插入到更新队列；
若q1和q2有一个是非空队列，则两个对列都需要更新；
当q1和q2两个队列都是非空，由于结构共享，两个列表中的最后一次更新是相同的。因此，只需q1添加到更新队列即可；
最后将q2的lastUpdate指针更新。

