not in在实际使用中，因为not in会转化成多表连接，而且不使用索引

因此应使用lefe join然后去掉null