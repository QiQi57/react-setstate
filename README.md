# setState 
 使用setState来修改react状态。setState不是同步更新状态，多个setState会进行合并操作，放到一个队列中，最终批量更新状态。
## 1.如何使用setState
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

## 2.setState 问题
上面例子的结果，0 0 是因为setState的时候react 会把多个操作放到一个队列缓存，通过react transaction 来实现，执行批量更新。3是因为settimeout作为一个单独事务来执行，并且开始的两个setState 合并操作，只有最后一个setState起作用了。
如何才能不把两次操作合并呢？可以setState传入函数来解决，最终count的值为3
```
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
```

## 3.setState 用法扩展
setState第2个参数，可以取得state最新状态值
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

## 5.setState batch update实现原理
react setState 通过react transaction 来实现批量更新操作。下面是transaction 执行过程。transaction 创建的时候需要传入getTransactionWrapper，并且传入需要是个数组集合，集合中每个对象都需要实现initialize和close方法。当transaction实例开始perform(anyMethod)时候，会initializeAll(0) 把所有wrapper中initialize都执行，然后执行anyMethod，最后会调用closeAll(0),把所有wrapper中的close方法都执行。
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
 *                    |   v     v                     v     v   | wrapper
 *                    | +---+ +---+   +---------+   +---+ +---+ | invariants
 * perform(anyMethod) | |   | |   |   |         |   |   | |   | | maintained
 * +----------------->|-|---|-|---|-->|anyMethod|---|---|-|---|-|-------->
 *                    | |   | |   |   |         |   |   | |   | |
 *                    | |   | |   |   |         |   |   | |   | |
 *                    | |   | |   |   |         |   |   | |   | |
 *                    | +---+ +---+   +---------+   +---+ +---+ |
 *                    |  initialize                    close    |
 *                    +-----------------------------------------+
 * </pre>
```

















