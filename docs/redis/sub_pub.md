# <center>Redis发布订阅
Redis的发布订阅是一种消息通信模式，通过channel来实现发布者publish消息，消费者subscribe消息。每个Redis客户端可以订阅任意数量的channel。

* psubscribe pattern [pattern...]
  > 订阅任意个指定模式匹配到的channel
* publish channel message
  > 向指定的channel发送消息
* punsubscribe pattern [pattern...]
  > 取消订阅模式匹配到的所有channel
* subscribe channel [channel...]
  > 订阅指定的channel
* unsubscribe channel [channel...]
  > 取消订阅指定的chanel