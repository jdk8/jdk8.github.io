# 交易系统


![在这里插入图片描述](trade.svg)


## 订单薄算法

需求是价格优先、时间优先，订单到达订单薄后，如果能吃掉单，则成交，否则撤单或者挂在订单薄上。

关键数据结构：

```java
class Order {
    DepthLine depthLine; // 所在深度线
}

class DepthLine {
    long price;
    LinkedList<Order> allOrders; // 买方或者卖方所有订单List
    Order first;
    Order last;
}

// 删除订单复杂度O(1)-撤单，成交后删除
// 吃单时查找卖一或者买一，O(1)
// 挂单复杂度，可变基数树查找/插入O(K)，K常量，O（1）
class OrderBook{
    Map<Id, Order> orderId2Order;                // 订单ID快速查找订单
    Map<UserId, Map<Id, Order>> userId2OrderMap; // 快速查找用户订单
  
    TreeMap<price, DepthLine> buyBook;
    TreeMap<price, DepthLine> sellBook;    
  
    LinkedList<Order> buyOrders; // 所有买单列表
    LinkedList<Order> sellOrders;
  
}
```

![在这里插入图片描述](orderbook.svg)

