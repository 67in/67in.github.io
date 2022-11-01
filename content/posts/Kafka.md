---
title: "Kafka"
date: 2022-07-05T23:15:06+08:00
draft: false
---

## Kafka Consumer API

> 独立消费者 Standalone Consumer 每次都会从第1条消息开始消费，一直到消费完全部小新，不会记录offset，妥妥的重复消费，需要借助 OffsetManager 来完成。

## 未使用OffsetManager的StandaloneConsumer

### 单分区消费

```go
func SinglePartition(topic string) {
	config := sarama.NewConfig()
	host := "x.x.x.x:xx"
	consumer, err := sarama.NewConsumer([]string{host}, config)
	if err != nil {
		log.Fatal("NewConsumer err:", err)
	}
	defer consumer.Close()

	//独立消费者中 sarama.OffsetOldest 会从最旧的消息开始消费，即每次重启 consumer后 都会把该topic下的所有消息都消费一次
	partitionConsumer, err := consumer.ConsumePartition(topic, 0, sarama.OffsetOldest)
	if err != nil {
		log.Fatal("ConsumePartition err:", err)
	}
	defer partitionConsumer.Close()
	//会一直阻塞在这里
	for message := range partitionConsumer.Messages() {
		log.Printf("[Consumer] partitionid:%d; offset:%d, value:%s\n", message.Partition, message.Offset, string(message.Value))
	}
}

```

### 多分区消费

```go
func Partitions(topic string) {
	config := sarama.NewConfig()
	host := "x.x.x.x:xx"
	consumer, err := sarama.NewConsumer([]string{host}, config)
	if err != nil {
		log.Fatal("NewConsumer err: err")
	}
	defer consumer.Close()
	//先查询 topic有多少分区
	partitions, err := consumer.Partitions(topic)
	if err != nil {
		log.Fatal("Partitions err:", err)
	}
	var wg sync.WaitGroup
	wg.Add(len(partitions))
	//然后每个分区开一个goroutine来消费
	for _, partitionId := range partitions {
		go consumeByPartition(consumer, topic, partitionId, &wg)
	}
	wg.Wait()
}
func consumeByPartition(consumer sarama.Consumer, topic string, partitionId int32, wg *sync.WaitGroup) {
	defer wg.Done()
	partitionConsumer, err := consumer.ConsumePartition(topic, partitionId, sarama.OffsetOldest)
	if err != nil {
		log.Fatal("ConsumePartition err:", err)
	}
	defer partitionConsumer.Close()
	for message := range partitionConsumer.Messages() {
		log.Printf("[Consumer] partitionid:%d; offset:%d, value:%s\n", message.Partition, message.Offset, string(message.Value))
	}
}
```



反复运行上面的Demo会发现，每次都会从第1条消息开始消费，一直到消费完 全部消息。
这不是妥妥的重复消费吗？
Kafka和其他MQ最大的区别在于Kafka中的消息再消费后不会被删除，而是会一直保留，直到过期。
为了防止每次重启消费者都从第1条消息开始消费，我们需要再消费消息后将offset提交给Kafka。这样重启后就可以接着上次的Offset继续消费了

在独立消费者中没有实现提交Offset的功能，所以需要借助OffsetManager来完成

### 使用OffsetManager的StandaloneConsumer

```go
func OffsetManager(topic string) {
   config := sarama.NewConfig()
   host := "x.x.x.x:xx"
   //配置开启自动提交offset， 这样samara库会自动把最新的offset提交给kafka
   config.Consumer.Offsets.AutoCommit.Enable = true              //开启自动commit offset
   config.Consumer.Offsets.AutoCommit.Interval = 1 * time.Second //自动 commit时间间隔
   client, err := sarama.NewClient([]string{host}, config)
   if err != nil {
      log.Fatal("NewClient err:", err)
   }
   defer client.Close()

   consumer, err := sarama.NewConsumerFromClient(client)
   if err != nil {
      log.Println("NewConsumeFromClient err:", err)
   }
   defer consumer.Close()

   // 先查询该 topic 有多少分区
   partitions, err := consumer.Partitions(topic)
   if err != nil {
      log.Fatal("Partitions err: ", err)
   }

   var wg sync.WaitGroup
   wg.Add(len(partitions))
   for _, partitionId := range partitions {
      go ConsumeByOffsetManager(client, partitionId, &wg, consumer)
   }
   wg.Wait()
}

func ConsumeByOffsetManager(client sarama.Client, i int32, wg *sync.WaitGroup, consumer sarama.Consumer) {
   defer wg.Done()
   //offsetManager用于管理每个ConsumerGroup的offset
   //根据groupId来区分不同的consume, 注意：每次提交的offset信息也是和groupId关联的
   offsetManager, err := sarama.NewOffsetManagerFromClient("myGroupId", client) //偏移管理器
   if err != nil {
      log.Println("NewOffsetManagerFromClient err:", err)
   }
   defer offsetManager.Close()
   partitionOffsetManager, err := offsetManager.ManagePartition("topic_name", i) //对应分区的偏移量管理管理器
   if err != nil {
      log.Println("ManagerPartition err:", err)
   }
   defer partitionOffsetManager.Close()
   //defer在程序结束后再commit一次，防止自动提交间隔之间的信息被丢掉
   defer offsetManager.Commit()

   //根据kafka中记录的上次消费的offset开始+1的位置接着消费
   nextOffset, _ := partitionOffsetManager.NextOffset()
   fmt.Println("nextOffset:", nextOffset)

   pc, err := consumer.ConsumePartition("topic_name", i, nextOffset)
   if err != nil {
      log.Println("ConsumePartition err:", err)
   }
   defer pc.Close()

   for message := range pc.Messages() {
      value := string(message.Value)
      log.Printf("[Consumer] partitionid:%d; offset:%d, value:%s\n", message.Partition, message.Offset, value)
      //每次消费后都更新一次offset，这里更新的只是程序内存中的值，需要commit后才能提交到Kafka
      partitionOffsetManager.MarkOffset(message.Offset+1, "modified metadata")
   }
}
```

1）创建偏移量管理器

```
offsetManager, _ := sarama.NewOffsetManagerFromClient("myGroupID", client) 
```

2）创建对应分区的偏移量管理器

> Kafka 中每个分区的偏移量是单独管理的

```
partitionOffsetManager, _ := offsetManager.ManagePartition(topic, kafka.DefaultPartition) 
```

3）记录偏移量

> 这里记录的是下一条要取的消息，而不是取的最后一条消息，所以需要 +1

```
partitionOffsetManager.MarkOffset(message.Offset+1, "modified metadata") 
```

4）提交偏移量

> sarama 中默认会自动提交偏移量，但还是建议用 defer 在程序退出的时候手动提交一次。

```
defer offsetManager.Commit()
```

更多请看 ：https://www.lixueduan.com/posts/kafka/05-quick-start/#3-consumer-api
