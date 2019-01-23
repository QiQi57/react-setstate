# setState 
 使用setState来修改react状态。如果setState传入的是对象，修改了状态之后不会立即更新状态，而是把状态放到一个队列中，如果多个setState都修改了状态，会对这些修改操作进行合并（shallowly merged），然后批量更新状态。如果setState传入的是函数，会作为一个独立事务执行（不会放入队列中），并把结果合并（shallowly merged）成新的状态返回。
 
 备注：setState 会re-render组件，除非shouldComponentUpdate() 返回false.
    状态merge的时候是shallowly merged。
    默认情况下react只在event handlers中才会进行batch update.根据官方文档／开发者介绍，react 17.x发布之后，会进行统一，都会进行batch update。
## 1.使用setState，传入对象
引用一个例子，
```
class Sample extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      count: 0
    };
  }
  componentDidMount() {
      var that=this;
    this.setState({
      count: this.state.count + 1
    });
    console.log(this.state.count);    // 打印 0
    this.setState({
      count: this.state.count + 2
    });
    console.log(this.state.count);    // 打印 0
    setTimeout(function(){
        that.setState({
       count: that.state.count + 1
     });
     console.log(that.state.count);   // 打印 3
     that.setState({
       count: that.state.count + 1
     });
     console.log(that.state.count);   // 打印 4
    }, 0);
    // setTimeout(function(){
    //     that.setState({
    //    count: that.state.count + 1
    //  });
    //  console.log(that.state.count);   // 打印
    // }, 0);
  }
  render() {
    return (
      <h1>{this.state.count}</h1>
    )
  }
}
```

## 2.setState 传入函数
上面例子的结果，0 0 是因为setState的时候react 会把多个操作放到一个队列缓存，通过react transaction 来实现，执行批量更新。3是因为settimeout作为一个单独事务来执行，并且开始的两个setState 合并操作，只有最后一个setState起作用了。
如何才能不把两次操作合并呢？可以setState传入函数来解决，最终count的值为3
```
class RootFun extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      count: 0
    };
  }
  componentDidMount() {
    let that = this;
    this.setState(function(state, props) {
      return {
        count: state.count + 1
      }
    });
    this.setState(function(state, props) {
      return {
        count: state.count + 2
      }
    });
  }
  render() {
    return (
      <h1>{this.state.count}</h1>   //页面中将打印出3
    )
  }
}

```

## 3.setState 第2个参数（用法扩展）
componentDidUpdate 和 setState callback (setState(updater, callback))都可以取得最新状态，推荐使用componentDidUpdate。setState callback 会在setState完成并且会在组件的re-rendered之后执行。
```
class Sample extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      count: 0
    };
  }
  componentDidMount() {
      var that=this;
    this.setState({
      count: this.state.count + 1
    },function(pre){
        console.log(this.state.count);    // 打印 1
    });
  }
  render() {
    return (
      <h1>{this.state.count}</h1>
    )
  }
}
```

## 4.setState 相关事件
setState -> shouldComponentUpdate ->componentWillUpdate ->render ->componentDidUpdate

## 5.setState batch update 批量更新实现原理
react setState 通过react transaction 来实现批量更新操作。下面是transaction 执行过程。transaction 创建的时候需要传入getTransactionWrappers，并且传入需要是个数组集合，集合中每个对象都需要实现initialize和close方法。当transaction实例开始perform(anyMethod)时候，会initializeAll(0) 把所有wrapper中initialize都执行，然后执行anyMethod，最后会调用closeAll(0),把所有wrapper中的close方法都执行。
下面看一下transaction的执行示意图（把源码中的图稍微修改一下）：
```
 * <pre>
 *                       wrappers (injected at creation time)
 *                                      +        +
 *                                      |        |
 *                    +-----------------|--------|--------------+
 *                    |                 v        |              |
 *                    |      +---------------+   |              |
 *                    |   +--|    wrapper1   |---|----+         |
 *                    |   |  +---------------+   v    |         |
 *                    |   |          +-------------+  |         |
 *                    |   |     +----|   wrapper2  |--------+   |
 *                    |   |     |    +-------------+  |     |   |
 *                    |   |     |                     |     |   |
 *                    |                               v     v   | wrapper
 *                    | +---+ +---+   +---------+   +---+ +---+ | invariants
 * perform(anyMethod) | |   | |   |   |         |   |   | |   | | maintained
 * +----------------->|-|---|-|---|-->|anyMethod|---|---|-|---|-|-------->
 *                    | |   ^ |   ^   |         ^   |   ^ |   ^ |
 *                    | |   | |   |   |         |   |   | |   | |
 *                    | v   | v   |   v         |   v   | v   | |
 *                    |  ---   ---     ---------     ---   --- |
 *                    |  initialize                    close    |
 *                    +-----------------------------------------+
 * </pre>
```

先看一下transaction perform 函数的实现源码。当调用transaction的perform方法时，会先执行this.initializeAll(0)，然后执行perform传入的方法，最后执行this.closeAll(0)。transaction的实现就是这么简单。问题的关键就在于getTransactionWrappers 函数，initialize 和close方法都是通过这个函数，在transaction初始化的时候传入的。
```
 perform: function(method, scope, a, b, c, d, e, f) {
    var errorThrown;
    var ret;
    try {
      this._isInTransaction = true;
      errorThrown = true;
      this.initializeAll(0); //把初始化时 传入的wrappers 数组中的所有initialize都执行
      ret = method.call(scope, a, b, c, d, e, f);//perform 调用时传入需要执行的方法 
      errorThrown = false;
    } finally {
      try {
        if (errorThrown) {
          try {
            this.closeAll(0);//把初始化时 传入的wrappers 数组中的所有close都执行(initialize或perform传入的方法异常时会执行)
          } catch (err) {
          }
        } else {
          this.closeAll(0);//把初始化时 传入的wrappers 数组中的所有close都执行
        }
      } finally {
        this._isInTransaction = false;
      }
    }
    return ret;
  },
```
下面看一下transaction initializeAll 和 函数的实现源码：
```
 initializeAll: function(startIndex) {
    var transactionWrappers = this.transactionWrappers;
    for (var i = startIndex; i < transactionWrappers.length; i++) {
      var wrapper = transactionWrappers[i];
      try {
        this.wrapperInitData[i] = Transaction.OBSERVED_ERROR;
        this.wrapperInitData[i] = wrapper.initialize ?
          wrapper.initialize.call(this) :
          null;// 执行所有的 initialize 方法
      } finally {
        if (this.wrapperInitData[i] === Transaction.OBSERVED_ERROR) {
          try {
            this.initializeAll(i + 1);//initialize 异常时，执行下一个 initialize 方法
          } catch (err) {
          }
        }
      }
    }
  },
  closeAll: function(startIndex) {
    var transactionWrappers = this.transactionWrappers;
    for (var i = startIndex; i < transactionWrappers.length; i++) {
      var wrapper = transactionWrappers[i];
      var initData = this.wrapperInitData[i];
      var errorThrown;
      try {
        errorThrown = true;
        if (initData !== Transaction.OBSERVED_ERROR && wrapper.close) {
          wrapper.close.call(this, initData);// 执行所有的 close 方法
        }
        errorThrown = false;
      } finally {
        if (errorThrown) {
          try {
            this.closeAll(i + 1);//close 异常时，执行下一个 close 方法
          } catch (e) {
          }
        }
      }
    }
    this.wrapperInitData.length = 0;
  }
```


transaction 文档https://github.com/facebook/react/blob/401e6f10587b09d4e725763984957cf309dfdc30/src/shared/utils/Transaction.js

setState如何使用transaction 完成batch update?

看一下源码，首先定义需要执行的操作 FLUSH_BATCHED_UPDATES，RESET_BATCHED_UPDATES，通过getTransactionWrappers 接口传入transaction 中。当调用 ReactDefaultBatchingStrategy.batchedUpdates ，如果当前处于isBatchingUpdates=false，会通过transation执行callback，并且在close函数中会先调用flushBatchedUpdates，然后重置状态isBatchingUpdates 的值 ；如果isBatchingUpdates=true，会直接调用callback.

```
var RESET_BATCHED_UPDATES = {
  initialize: emptyFunction,
  close: function() {
    ReactDefaultBatchingStrategy.isBatchingUpdates = false;
  }
};

var FLUSH_BATCHED_UPDATES = {
  initialize: emptyFunction,
  close: ReactUpdates.flushBatchedUpdates.bind(ReactUpdates)
};

var TRANSACTION_WRAPPERS = [FLUSH_BATCHED_UPDATES, RESET_BATCHED_UPDATES];

function ReactDefaultBatchingStrategyTransaction() {
  this.reinitializeTransaction();
}

assign(
  ReactDefaultBatchingStrategyTransaction.prototype,
  Transaction.Mixin,
  {
    getTransactionWrappers: function() {
      return TRANSACTION_WRAPPERS;
    }
  }
);

var transaction = new ReactDefaultBatchingStrategyTransaction();

var ReactDefaultBatchingStrategy = {
  isBatchingUpdates: false,
  batchedUpdates: function(callback, a, b, c, d) {
    var alreadyBatchingUpdates = ReactDefaultBatchingStrategy.isBatchingUpdates;

    ReactDefaultBatchingStrategy.isBatchingUpdates = true;

    // The code is written this way to avoid extra allocations
    if (alreadyBatchingUpdates) {
      callback(a, b, c, d);
    } else {
      transaction.perform(callback, null, a, b, c, d);
    }
  }
};
```



## 6.setState 合并操作实现原理

通过上面事务我们看到 flushBatchedUpdates 执行批量更新。在flushBatchedUpdates 后续操作中，_processPendingState 会先对修改的state进行合并(shallowly merged)。所以，如果setState传入的是函数，不会对修改操作进行合并，而是会立即执行修改，把最终状态合并到最新状态。这就解释了上面例子中，如果需要进行连续修改状态，可以通过setState传入函数来实现。

```
  _processPendingState: function(props, context) {
    var inst = this._instance;
    var queue = this._pendingStateQueue;
    var replace = this._pendingReplaceState;
    this._pendingReplaceState = false;
    this._pendingStateQueue = null;

    if (!queue) {
      return inst.state;
    }

    if (replace && queue.length === 1) {
      return queue[0];
    }

    var nextState = assign({}, replace ? queue[0] : inst.state);
    for (var i = replace ? 1 : 0; i < queue.length; i++) {
      var partial = queue[i];
      assign(
        nextState,
        typeof partial === 'function' ?
          partial.call(inst, nextState, props, context) :
          partial
      );//合并修改的状态
    }

    return nextState;
  },
```










