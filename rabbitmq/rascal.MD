# 连接断开自动重试

`rascal` 内部通过 [generic-pool](https://github.com/coopernurse/node-pool#readme) 维护连接池，并使用 `poolQueue` （使用了 [async queue](https://caolan.github.io/async/v3/docs.html#queue) ）维护需要的发送回调 (err, channel)，其内部调用 `channel` 的 `publish` 方法与 `rabbitMQ` 通信发送数据。
使用自动重连的机制就是当由于网络，`rabbitMQ` 宕机等原因导致连接断开会被 `handleConnectionError` 捕获，代码如下

```js
// 绑定错误回调
function attachErrorHandlers(connection, config) {
    connection.removeAllListeners('error');
    var errorHandler = _.once(handleConnectionError.bind(null, connection, config));
    connection.once('error', errorHandler);
    connection.once('close', errorHandler);
  }
// 捕获连接错误
 function handleConnectionError(borked, config, err) {
    debug('Handling connection error: %s initially from connection: %s, %s', err.message, borked._rascal_id, connectionConfig.loggableUrl);
    self.emit('disconnect');
    pauseChannelAllocation();
    connection = undefined;
    self.emit('error', err);
    console.log('retry is ', connectionConfig.retry)
    connectionConfig.retry && self.init(function(err) {
      if (!err) return;
      var delay = timer.next();
      debug('Will attempt reconnection in in %dms', delay);
      return setTimeout(handleConnectionError.bind(null, borked, config, err), delay).unref();
    });
  }
```

此函数会当默认配置重试的时候，在一定延时后递归调用自身，实现断开重连。

# 消息发送预存队列

在尝试连接的这段时间内如何保证发送队列不会丢失数据？ rascal 使用了 queue 来保存所有的发送调用并依次调用，当连接断开的时候队列的调用会被暂停，因为在连接断开的错误捕获函数 `handleConnectionError` 内部调用了 `pauseChannelAllocation` 函数，如下

```js
  function pauseChannelAllocation() {
    channelCreator.pause();
    regularChannelPool && regularChannelPool.pause();
    confirmChannelPool && confirmChannelPool.pause();
  }
```

而 `confirmChannelPool` 是什么呢？可以理解成下面这样，

```js
confirmChannelPool = {
      stats: stats,
      borrow: borrow,
      return: release,
      drain: drain,
      pause: poolQueue.pause.bind(poolQueue),
      resume: poolQueue.resume.bind(poolQueue),
    };
```

从上面源码可以看出来调用了 `poolQueue` 的 `pause` 方法，该方法会使队列暂停工作（此时队列可能由于会有新加入的回调造成堆积），而连接恢复的时间会调用对应的 resume 方法来使队列继续工作，那么也就可以保证当集群恢复的时候不会丢失发送数据，因为发送队列在断开的时候进行已经暂停了。

队列核心代码如下：

```js
// 定义队列
poolQueue = async.queue(function(__, cb) {
      pool.acquire().then(function(channel) {
        setImmediate(function() {
          cb(null, channel);
        });
      }).catch(cb);
    }, 1);
// 推送队列回调函数
poolQueue.push(null, function cb (err, channel) {
            // 这个next 是下面的borrowChannelFn 的参数（回调函数)
        if (err) return next(err);
        debug('Borrowed %s channel: %s. %o', mode, channel._rascal_id, stats());
        next(null, channel); 
）});
// 实际调用的回调
borrowChannelFn(function(err, channel) {
      if (err) return emitter.emit('error', err, messageId);
      var errorHandler = _.once(handleChannelError.bind(null, channel, messageId, emitter, config));
      var returnHandler = emitter.emit.bind(emitter, 'return');
      addListeners(channel, errorHandler, returnHandler);
      try {
        publishFn(channel, buffer, publishConfig, function(err, ok) {
          ok ? returnChannel(channel, errorHandler, returnHandler)
             : deferReturnChannel(channel, errorHandler, returnHandler);
          if (err) return emitter.emit('error', err, messageId);
          emitter.emit('success', messageId);
        });
      } catch(err) {
        returnChannel(channel, errorHandler, returnHandler);
      };
});
```
# BUG

测试发现当 `pool.acquire()` 拿到已经关闭的连接时（因为获取的同时连接被关闭），会正常走 next ，这时候会导致消息丢失，因为这个队列回调已经被调用了，而后续的 `error` 没有触发 `error` 事件，相关已经提了 MR

https://github.com/guidesmiths/rascal/pull/81

# 生产者顺序发送

使用 `poolQueue` 维护待发布消息的发布方法回调，依次进行调用，进而实现顺序发送。

# 生产者确认模式

`publish` 方法内部声明了 `EventEmitter` 实例，通过事件回调的形式处理 `rabbitMQ` 的生产者确定模式。

```js
  function _publish(buffer, publishConfig, next) {
    var emitter = new EventEmitter(); // 声明 emitter
    var messageId = publishConfig.options.messageId;
    borrowChannelFn(function(err, channel) {
      if (err) return emitter.emit('error', err, messageId);
      var errorHandler = _.once(handleChannelError.bind(null, channel, messageId, emitter, config));
      var returnHandler = emitter.emit.bind(emitter, 'return');
      addListeners(channel, errorHandler, returnHandler);
      try {
        publishFn(channel, buffer, publishConfig, function(err, ok) {
          ok ? returnChannel(channel, errorHandler, returnHandler)
             : deferReturnChannel(channel, errorHandler, returnHandler);
          if (err) return emitter.emit('error', err, messageId);
          emitter.emit('success', messageId);
        });
      } catch(err) {
        returnChannel(channel, errorHandler, returnHandler);
        return emitter.emit('error', err, messageId);
      };
    });
    next(null, emitter); // 这里返回了这个 emitter
  }
```

使用的时候需要监听这几个事件，然后对应不同的事件需要自己决定选择重发或者别的方案。

```js
broker.publish("p1", "some message", (err, publication) => {
  if (err) throw err; // publication didn't exist
  publication.on("success", (messageId) => {
     console.log("Message id was: ", messageId)
  }).on("error", (err, messageId) => {
     console.error("Error was: ", err.message)
  }).on("return", (message) => {
     console.warn("Message was returned: ", message.properties.messageId)
  })
})
```

# 生产者自动重试

根据上面的确认机制， racal 本身并没有默认的确认模式下的消息失败处理机制，只是给了相关的事件回调，需要开发者自身处理。

# 消费者模型

调用 subscribe 返回的 SubscriberSession 是 EventEmitter 对象，所以可以通过监听 message 事件来收到通知，可以分析  subscribeLater  函数来找到触发事件的地方。

```js
  this.subscribe = function(overrides, next) {
    var session = new SubscriberSession(sequentialChannelOperations, config);
    subscribeLater(session, _.defaultsDeep(overrides, config));
    return next(null, session);
  };
```

subscribeNow 调用 channel.consume 和 rabbitMQ 建立 socket 连接，并将 _onMessage  绑定到回调函数。

```js
  function subscribeNow(session, config, next) {
    sequentialChannelOperations.push(function(done) {
      if (session.isCancelled()) {
        debug('Subscription to queue: %s has been cancelled', config.queue);
        return done();
      }
      debug('Subscribing to queue: %s', config.queue);
      vhost.getChannel(function(err, channel) {
        if (err) return done(err);
        if (config.prefetch) channel.prefetch(config.prefetch);
        var removeErrorHandlers = attachErrorHandlers(channel, session, config);
        var onMessage = _onMessage.bind(null, session, config, removeErrorHandlers);
        channel.consume(config.source, onMessage, config.options, function(err, response) {
          if (err) {
            debug('Error subscribing to %s using channel: %s. %s', config.source, channel._rascal_id, err.message);
            removeErrorHandlers();
            return session.isCancelled() ? done() : done(err);
          }
          session._open(channel, response.consumerTag, function(err) {
            if (err) return done(err);
            timer.reset();
            done();
          });
        });
      });
    }, next);
  }
```

`onMessage` 方法内部调用 `emit('message')` 触发事件。

```js
  function _onMessage(session, config, removeErrorHandlers, message) {
    if (!message) return handleConsumerCancel(session, config, removeErrorHandlers);
    debug('Received message: %s from queue: %s', message.properties.messageId, config.queue);
    decorateWithRoutingHeaders(message);
    if (immediateNack(message)) return ackOrNack(session, message, true);
    decorateWithRedeliveries(message, function(err) {
      if (err) return handleRedeliveriesError(err, session, message);
      if (redeliveriesExceeded(message)) return handleRedeliveriesExceeded(session, message);
      getContent(message, config, function(err, content) {
        err ? handleContentError(session, message, err)
            : session.emit('message', message, content, ackOrNack.bind(null, session, message));
      });
    });
  }
```

# 消费者重连机制

当 `channel` 异常断开当时候，会被 `handleChannelError` 捕获并尝试重新调用 `subscribeNow` 来建立连接，如果失败会根据默认的重试配置在一定间隔后递归调用 `handleChannelError` 来无限尝试重连。

```js
  function handleChannelError(session, config, removeErrorHandlers, attempts, err) {
    debug('Handling channel error: %s from %s using channel: %s', err.message, config.name, session._getRascalChannelId());
    if (removeErrorHandlers) removeErrorHandlers();
    session.emit('error', err);
    config.retry && subscribeNow(session, config, function(err) {
      if (!err) return;
      var delay = timer.next();
      debug('Will attempt resubscription(%d) to %s in %dms', attempts + 1, config.name, delay);
      return setTimeout(handleChannelError.bind(null, session, config, null, attempts + 1, err), delay).unref();
    });
  }
```

# 消息 ACK

肯定确认，用于消息正常处理或者因为幂等和其他原因忽略此消息的情况。

```js
aclOrNack() 
```

否定确认，消息会被删除。如果配置了死信队列，会被保存到死信队列。

```js
ackorNack(err) 
```

否定确认，要求在 defer 延迟后重发消息，消息会被重新放到队列前面。

```js
ackorNack(err, { strategy: "nack", requeue: true, defer: 5000 })
```

否定确认，要求在 defer 延迟后重发消息，消息会被放到队列后面，原理是复制此条消息然后重新发送，使用确认模式，发送完成后 ack 源消息。如果配置了 attempts ，则达到 attempts 次数后，消息会 nack , 等同 ackorNack(err)

```js
ackOrNack(err, { strategy: 'republish', defer: 1000, attempts: 10 })
```

否定确认，attempts 次数内尝试转发到指定的 publication，如果将该 publication 路由到延迟队列，延迟队列再配置死信 exchange，在消息超时后通过死信 exchange 路由重新到队列，功能和 republish 差不多，但是超时的状态和消息保存在额外队列中，不会因为应用重启等原因导致重启计时器。

```js
ackOrNack(err, { strategy: 'forward', publication: 'some_exchange' , attempts: 10 })
```