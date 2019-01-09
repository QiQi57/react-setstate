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
    console.log(this.state.count);    // 打印
    this.setState({
      count: this.state.count + 1
    });
    console.log(this.state.count);    // 打印
    setTimeout(function(){
        that.setState({
       count: that.state.count + 1
     });
     console.log(that.state.count);   // 打印
     that.setState({
       count: that.state.count + 1
     });
     console.log(that.state.count);   // 打印
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


## 2.setState 问题

## 3.setState 解决办法

## 4.setState 相关事件


## 5.setState 实现原理



