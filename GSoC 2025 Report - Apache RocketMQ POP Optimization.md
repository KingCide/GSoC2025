## Summary
- [https://github.com/apache/rocketmq/issues/9632](https://github.com/apache/rocketmq/issues/9632)
- [https://github.com/apache/rocketmq/issues/9633](https://github.com/apache/rocketmq/issues/9633)
- [https://github.com/apache/rocketmq/pulls](https://github.com/apache/rocketmq/pulls)

## Problem Description
| Consumer Group | Topic | Retry Topic |
| :--- | :--- | :--- |
| `MyConsumerGroup` | `Order_Topic` | `%RETRY%MyConsumerGroup_Order_Topic` |
| `MyConsumerGroup_Order` | `Topic` | `%RETRY%MyConsumerGroup_Order_Topic` |

### Problem 1: Pop long-polling is not awakened for V1 retry messages, causing significant consumption delay
In Pop consumption mode, when a message is not acknowledged (ACKed) within its invisible time, it is moved to a retry topic for later consumption. The system is expected to wake up any waiting long-polling requests for the original topic so the message can be re-consumed promptly.

Currently, this wake-up mechanism works correctly for V2 retry topics (formatted as `%RETRY%group+topic`) because the original topic and group can be reliably parsed from the name.

However, for V1 retry topics (formatted as `%RETRY%group_topic`), the Broker fails to parse the original topic name from the retry topic. As a result, the `notifyMessageArrivingWithRetryTopic` method cannot identify and awaken the correct long-polling request. This forces the consumer's long-polling request to wait until it times out (controlled by `BROKER_SUSPEND_MAX_TIME_MILLIS`, typically 15 seconds), introducing a significant delay in message retries.

### Problem 2: V1 retry topic name collisions can cause cross-topic consumption and message mix-up
Under the V1 retry topic naming convention (`%RETRY%group_topic`), different (topic, consumerGroup) pairs may map to the same retry topic name. When this happens, messages retried from one topic/group can be consumed by another topic/group that shares the same V1 retry topic name, causing cross-topic consumption and potential data leakage.

**Root cause**: The V1 retry topic naming scheme (`%RETRY%group_topic`) loses delimiter information and is not a one-to-one mapping. Different combinations can collide, for example:

- Combination 1: (Topic: `Order_Topic`, Consumer Group: `MyConsumerGroup`)
- Combination 2: (Topic: `Topic`, Consumer Group: `MyConsumerGroup_Order`)
Both of these combinations generate the same V1 retry topic name.

Since the Broker routes retried messages based solely on the retry topic name, this collision causes message streams from different sources to be merged.

### The V2 implementation causes many unnecessary wake-ups
Previously, when a message arrived on a V2 retry topic, it would wake up connections for all consumer groups subscribing to the original topic. In reality, only one specific group needed to be awakened. This resulted in many empty pulls and wasted resources.

## Solutions
### Final Solution
```java
public void notifyMessageArrivingWithRetryTopic(final String topic, final int queueId, long offset,
                                                Long tagsCode, long msgStoreTime, byte[] filterBitMap, Map<String, String> properties) {
    String prefix = MixAll.RETRY_GROUP_TOPIC_PREFIX;
    if (topic.startsWith(prefix)) {
        // Get the original topic name from properties
        String originTopic = properties.get(MessageConst.PROPERTY_ORIGIN_TOPIC);
        // Based on the original topic and retry topic, derive the corresponding cid (consumer group ID)
        // (This could also be cross-verified with topicCidMap)
        String suffix = "_" + originTopic; // Using '+' instead of '_' would also work here
        String cid = topic.substring(prefix.length(), topic.length() - suffix.length());
        POP_LOGGER.info("Processing retry topic: {}, originTopic: {}, properties: {}",
                        topic, originTopic, properties); // You can see the logs by running: grep "Processing retry topic" ~/logs/rocketmqlogs/pop.log
        POP_LOGGER.info("Extracted cid: {} from retry topic: {}", cid, topic);
        // Then, call notifyMessageArriving with the cid
        long interval = brokerController.getBrokerConfig().getPopLongPollingForceNotifyInterval();
        boolean force = interval > 0L && offset % interval == 0L;
        if (queueId >= 0) {
            notifyMessageArriving(originTopic, -1, cid, force, tagsCode, msgStoreTime, filterBitMap, properties);
        }
        notifyMessageArriving(originTopic, queueId, cid, force, tagsCode, msgStoreTime, filterBitMap, properties);
    } else {
        // For normal (non-retry) messages, the previous logic remains unchanged
        notifyMessageArriving(topic, queueId, offset, tagsCode, msgStoreTime, filterBitMap, properties);
    }
}
```

1.  In `common/message/MessageConst.java`, add a new field `PROPERTY_ORIGIN_TOPIC` to store the original topic name for messages in the retry queue.
2.  During the Pop checkpoint (CK) processing, extract the original topic name from the CK and add it to the `PROPERTY_ORIGIN_TOPIC` field.
3.  In `notifyMessageArrivingWithRetryTopic`, parse the `cid` based on the extracted original topic name. **This allows for a targeted wake-up of a specific consumer group.**
4.  **This solves both the issue of V1 retry topics failing to wake up polls correctly and the issue of V2 versions causing many unnecessary empty wake-ups** (which previously woke up all CIDs under a topic based on `topicCidMap`).

### Solution 2
1.  In `notifyMessageArrivingWithRetryTopic`, check if popKV is enabled.
2.  Get the `cid` field from the popKV record.
3.  Use the `KeyBuilder.parseNormalTopic(topic, cid)` method to get the original topic name.
4.  Call `notifyMessageArriving` with the correct information.

### Solution 3
```java
public void notifyMessageArrivingWithRetryTopic(final String topic, final int queueId, long offset,
                                                Long tagsCode, long msgStoreTime, byte[] filterBitMap, Map<String, String> properties) {
    String notifyTopic;
    if (KeyBuilder.isPopRetryTopicV2(topic)) {
        notifyTopic = KeyBuilder.parseNormalTopic(topic);
    } else {
        notifyTopic = findTopicForV1RetryTopic(topic);
    }
    notifyMessageArriving(notifyTopic, queueId, offset, tagsCode, msgStoreTime, filterBitMap, properties);
}

/**
     * Find the correct topic name for V1 retry topic by checking topicCidMap
     * @param retryTopic V1 retry topic name
     * @return the original topic name, or the retryTopic itself if ambiguous
     */
private String findTopicForV1RetryTopic(String retryTopic) {
    // Check if the potential group exists in topicCidMap
    boolean hasDuplicatedTopic = false;
    String originalTopic = null;
    for (String topic : topicCidMap.keySet()) {
        ConcurrentHashMap<String, Byte> cids = topicCidMap.get(topic);
        if (cids != null) {
            for (String cid : cids.keySet()) {
                // Check if this cid could be the correct consumer group
                String expectedRetryTopic = KeyBuilder.buildPopRetryTopicV1(topic, cid);
                if (expectedRetryTopic.equals(retryTopic)) {
                    if (originalTopic == null) {
                        originalTopic = topic;
                    } else {
                        hasDuplicatedTopic = true;
                        break;
                    }
                }
            }
        }
    }
    if (hasDuplicatedTopic) {
        return retryTopic;
    } else {
        return originalTopic;
    }
}
```

1.  `topicCidMap` is a core data structure in `PopLongPollingService` that stores the mapping of consumer groups (CIDs) for each topic. It is a two-level map populated during the polling method.
2.  Leveraging this data structure, I designed a reverse lookup mechanism to avoid the ambiguity of string parsing:
    1.  Iterate through all topic-consumerGroup combinations in `topicCidMap`.
    2.  For each combination, reconstruct the V1 retry topic name and match it against the input retry topic.
3.  If multiple original topics can generate the same retry topic, it is marked as `hasDuplicatedTopic`.
    1.  In case of ambiguity, the original retry topic name is returned to avoid incorrect message routing. This prevents this code change from causing notification errors (though if a collision exists, the error will still occur due to the fundamental naming issue).

#### Phenomenon of the Solution
```bash
Received message #33 = MessageViewImpl{messageId=01EA2C235B8585B70D08C1575B00000015, topic=Topic, ...}
🔄 RETRY MESSAGE DETECTED!
   MessageId: 01EA2C235B8585B70D08C1575B00000015
   Delivery Attempt: 2
Received message #34 = MessageViewImpl{messageId=01EA2C235B8585B70D08C1575B0000000B, topic=Topic, ...}
🔄 RETRY MESSAGE DETECTED!
   MessageId: 01EA2C235B8585B70D08C1575B0000000B
   Delivery Attempt: 2
   Time since first failure: 3072 ms (3.072 seconds)
   Time since first failure: 3074 ms (3.074 seconds)
...
----------------------------------------
✅ Message #33 - SUCCESS
   Retried at: Thu Aug 28 10:22:30 CST 2025
----------------------------------------
✅ Message #34 - SUCCESS
   Retried at: Thu Aug 28 10:22:30 CST 2025
----------------------------------------
```
The wake-up is almost immediate, which meets expectations.

## Experimental Results
### Observation 1: Before the change, using remoting protocol PushConsumer, sending 32 messages at once
```plain
[NORMAL] Topic: TopicA, QueueId: 4, ReconsumeTimes: 0, Content: Hello world
    ❌ Do not ACK this message, let it go to the retry queue
[NORMAL] Topic: TopicA, QueueId: 3, ReconsumeTimes: 0, Content: Hello world
    ❌ Do not ACK this message, let it go to the retry queue
[RETRY] Topic: TopicA, QueueId: 0, ReconsumeTimes: 1, Content: Hello world
    ⚡ 1st retry message delay: 14955 ms (15.0 seconds)
[RETRY] Topic: TopicA, QueueId: 0, ReconsumeTimes: 1, Content: Hello world
    ⚡ 2nd retry message delay: 14955 ms (15.0 seconds)
```
1.  We choose not to ACK the first two messages.
2.  The producer sends no further messages.
3.  **After 15s, the messages from the retry queue are received** and can be re-consumed.
4.  **Experimental timeline:**
    1.  **Time message was first NACKed: 2025-08-22 16:05:21,414**
    2.  **Time consumer pulled message from retry queue: 2025-08-22 16:05:36,385**
    3.  **`mqadmin topicStatus` shows retry queue last update time: 2025-08-22 16:05:33,497**
5.  **Inference:** The constant `BROKER_SUSPEND_MAX_TIME_MILLIS = 1000 * 15` is taking effect. When the V1 retry topic fails to wake up the long poll, the Pop request waits until this 15-second timeout, then returns and re-initiates the request, at which point it can pull the messages from the retry queue.

### Observation 2: Before the change, using remoting protocol PushConsumer, with continuous message sending
```plain
[NORMAL] Topic: TopicTestForNormal, QueueId: 3, ReconsumeTimes: 0, Content: Hello world
    ❌ Do not ACK this message, let it go to the retry queue
...
[RETRY] Topic: TopicTestForNormal, QueueId: 0, ReconsumeTimes: 1, Content: Hello world
    ⚡ 1st retry message delay: 8067 ms (8.1 seconds)
[RETRY] Topic: TopicTestForNormal, QueueId: 0, ReconsumeTimes: 1, Content: Hello world
    ⚡ 2nd retry message delay: 9075 ms (9.1 seconds)
...
```
1.  We choose not to ACK the first two messages.
2.  The producer sends one message per second; the invisible time is set to 5 seconds.
3.  **After about 8 seconds, the first retry message is received.**
4.  **Inference:** With continuous message sending, the arrival of new messages triggers the long-polling request to return, which allows it to receive messages from the retry topic in a more timely manner after the invisible time expires.

### Observation 3: Before the change, using gRPC protocol PushConsumer, sending 32 messages at once
```bash
Time since first failure: 19973 ms (19.973 seconds)
First failed at: Wed Aug 27 20:14:24 CST 2025
     Retried at: Wed Aug 27 20:14:44 CST 2025
```
**The interval from NACK to the second pull is 20s.**
```bash
sh mqadmin topicstatus -n 127.0.0.1:9876 -t %RETRY%ConsumerGroupPush_Topic -c DefaultCluster
#Last Updated
2025-08-27 20:14:27,841
```
**However, the message entered the retry queue after only 3s.**

### Observation 4: After the change, using gRPC protocol PushConsumer, with continuous message sending
```bash
✅ Message #33 - SUCCESS
   First failed at: 2025-08-29 11:20:36,645
   Retried at: 2025-08-29 11:20:39,724
```
The message entered the retry queue at 11:20:39,716 and was pulled at 11:20:39,724. The retry delay is only about 3s (due to the message's invisible time).
```bash
bin % sh mqadmin topicstatus -n 127.0.0.1:9876 -t %RETRY%ConsumerGroup2_Topic
#Broker Name                      #QID  #Min Offset           #Max Offset             #Last Updated
broker-a                          0     0                     20                      2025-08-29 11:20:39,716
```
This proves that after the change, Pop consumption can be awakened immediately.

The pop log below shows that the `originTopic` property I added is correctly parsed and used to wake up the corresponding topic+cid pop long-polling request.
```bash
2025-08-29 11:20:39 INFO ReputMessageService - Processing retry topic: %RETRY%ConsumerGroup2_Topic, originTopic: Topic, properties: {ORIGIN_TOPIC=Topic, ...}
2025-08-29 11:20:39 INFO ReputMessageService - Processing retry topic: %RETRY%ConsumerGroup2_Topic, originTopic: Topic, properties: {ORIGIN_TOPIC=Topic, ...}
```

### Observation 5: After the change, using gRPC protocol SimpleConsumer, sending 32 messages at once
Setting invisible time to 10s.

Time of first message pull: `2025-08-29 15:56:39,876`
Time of second pull (retry message): `2025-08-29 15:56:52,958`
```bash
Received message #15 = MessageViewImpl{messageId=..., deliveryAttempt=2, ...}
🔄 RETRY MESSAGE DETECTED!
   MessageId: 010EC51B9D3D4AD91E08C2F73700000004
   Delivery Attempt: 2
   Time since first failure: 13082 ms (13.082 seconds)
   First failed at: 2025-08-29 15:56:39,876
   Retried at: 2025-08-29 15:56:52,958
```
The last message entered the `%RETRY%ConsumerGroup3_Topic` retry queue at `15:56:52,944`.
```bash
bin % sh mqadmin topicstatus -n 127.0.0.1:9876 -t %RETRY%ConsumerGroup3_Topic
#Broker Name                      #QID  #Min Offset           #Max Offset             #Last Updated
broker-a                          0     0                     20                      2025-08-29 15:56:52,944
```
Pop log:
```bash
2025-08-29 15:56:52 INFO ReputMessageService - Processing retry topic: %RETRY%ConsumerGroup3_Topic, originTopic: Topic, properties: {ORIGIN_TOPIC=Topic, ...}
2025-08-29 15:56:52 INFO ReputMessageService - Processing retry topic: %RETRY%ConsumerGroup3_Topic, originTopic: Topic, properties: {ORIGIN_TOPIC=Topic, ...}
```
This demonstrates that our consumption delay has been reduced from several seconds to a millisecond-level wake-up.

## Other Problem Exploration
### Exploring the use of `attemptId` to reduce blocking in Pop orderly consumption
The difficulty here is not in finding a solution; reusing `attemptId` can easily implement re-entry. The core problem is defining which scenarios actually require fixing the blocking behavior. In many cases, I found that blocking is the intended and reasonable solution.

After in-depth code reading, I discovered that `attemptId` is only used in the gRPC PushConsumer, and only reused after a network timeout. For the remoting protocol, `attemptId` is always passed as `null`, so the server treats every request as unique.

Therefore, to further utilize `attemptId` to address this problem, the possible options are:
1.  Categorize various blocking scenarios, distinguishing between those that should be re-entrant and those that should not.
2.  Expose this concept to the user, allowing SimpleConsumer users to control this logic themselves.

### Unreliable ACKs might cause blocking in Pop orderly consumption
In an orderly message scenario, the PushConsumer ensures, to some extent, that messages are consumed sequentially. Barring network issues, ACKs should be received in order. (However, the network is unreliable, and it's possible that the failure to deliver some ACKs could lead to consumption blockage).

For this problem: A cumulative acknowledgement approach (where a later ACK confirms all previous messages) could be a solution. However, this works well for single-queue sequential consumption but could cause issues if parallel consumption capabilities are introduced. **Moreover, the current strategy is not a major issue; if an ACK is not received, blocking is the expected behavior.**