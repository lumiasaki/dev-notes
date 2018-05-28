在我们开发的 React/React Native 应用达到一定规模时，简单地使用`setState()`来进行数据更新会使我们的逻辑变得越来越复杂，出了渲染上的错误想要找到错误根源变得越来越难，因为这个原因，许多开发者在使用 React/React Native 时，都会搭配着 React 全家桶中的数据流框架 Redux 来辅助应用的开发，即使 Redux 因其繁琐、死板的使用方式以及对初学者来说十分陡峭的学习曲线而饱受争议，但依然在 React 开发中占有重要的地位。

没用过 Redux 的人，想学习这个框架的话首先想到的就是去网上搜相关的文档，同样，Redux 官方提供了以完成一个`To-Do`应用的示例来介绍 Redux 使用方法的 Guide ，然而我们照着做一遍之后好像也知道了一点 Redux 工作方式的基本原则和概念，但为了一个简单的应用，我们不仅写了一堆 action，还要写一堆 reducer，简直付出了成倍的不必要劳动，好像我们不这么做，一个`To-Do`也完全能很快做出来。那既然 Redux 如此繁琐与啰嗦，还有一大堆不知所云的组件，为何还是迅速占领了极大的市场呢？这篇文章就来聊一聊 Redux 到底是怎样的一个框架。

## 一个🌰

我们来做一个小应用，其中某一个页面的代码片段如下：

```js
constructor(props) {
    super(props);
    this.state = {
        title: 'this is title',
        date: '2018-01-01'
    }
}
...
render() {
    return (
        <View>
            <Text>{this.state.title}</Text>
            <Text>{this.state.date}</Text>
        </View>
    )
}
```

假设在不考虑使用任何额外的数据流框架下，我想更新 title 的值，应该怎么做呢？很简单：

```js
this.setState({
    ...this.state,
    title: 'this is new title',
});
```

调用了上面这段代码后，`setState()`就会引起一次 render，title 得到了更新。需要说明的是，随着代码的复杂度上升，也许我们还得对 title 做非常复杂的逻辑变换之后，才能得到最终用于渲染的 title，将其作为传入`setState()`中的值。

注意到`...this.state`这一句，其目的是保留原来 state 中已有的值，而只修改 title，如果忘记这么做的话，会导致新的 state 中就只有 title 这一个值了，其他的都会消失。而当我们的代码规模达到庞大或非常庞大时，到处充满着`setState()`这样的代码，我们必须小心翼翼地操作着 state 中的内容，但也难免会有哪位同学一时疏忽，导致 state 中不想被改动到的值被修改了，而因为可能是任何一个`setState()`引起的，势必排查 bug 又是一个漫长而辛苦的过程。

如果我们想要更改 state 变得更加可控一点，会怎么做呢？

一个简单而又符合程序员抽象思维的方式很容易想到，比如下面这样做：

```js
setTitle(title) {
    // 一些可能存在的处理 title 的逻辑
    ...

    this.setState({
        ...this.state,
        title
    });
}

setDate(date) {
    // 一些可能存在的处理 date 的逻辑
    ...

    this.setState({
        ...this.state,
        date
    });
}
```

我们把修改域缩小，独立出两个函数，分别为`setTitle()`和`setDate()`，它们只负责修改指定的值，而内部定义了一个容易测试的、不容易出错的`setState()`，把对传入数据的额外处理逻辑聚合在该函数下，这样外部使用者由直接通过调用`setState()`变为了调用这些方法来实现渲染的更新。这样看起来就解决了上面遇到的那个问题。

而这样的操作，其实从某种程度来说，我觉得已经比较类似 Redux 的工作流程了。

## 聊聊 Redux 到底是什么

Redux 是什么呢？它当然是一个数据流框架，但我更觉得它是一种思想。由于我本人是做 iOS 客户端开发的，第一次接触到 Redux 这个框架，并且第一次更理解其内涵时，带来的感受是震撼的，相比于 iOS 开发中，我们可以在代码的任何位置去更新View，Redux 使用了一种非常简单、单向、可控、纯函数的方式，将渲染过程中的复杂度降低，当然在这个过程中我们可能失去了一些自由度，但其能让代码更可控和更健壮我认为是值得的。

下面，我们一点一点剥开 Redux的外壳，看看它里面到底是什么。

### 组成部分

首先，通过官方文档我们也可以很快的知道一个概念，那就是 Redux 是由三个主要组成部分组成的。

* Store
* Action
* Reducer


拿最初我们的那个例子来做对比，在这里：

Store 类似于例子中顶层 Component 的 state，作用是存放用于渲染的数据。  
Action 类似于例子中，调用修改 title 或 date 的函数。  
Reducer 类似于例子中，修改 title 或 date 函数中做了逻辑最后的那个`setState()`。

当然上面的比较只是类比！

### 工作原理

一步一步，透过最简版代码了解原理。

1. Store


首先明确一点，Store 是一个对象，这个对象里至少含有了三个函数：

```js
{
    getState: () => {
        // 用于获取当前的 state
        ...
    },
    dispatch: (action) => {
        // 用于分发传入的 action，action 本质上来说是一个对象
        ...
    },
    subscribe: (listener) => {
        // 用于订阅，在 state 更新之后能够刷新，例如可以订阅 render()
        ...
    }
}
```

而我们使用 Redux 时，要生成一个 Store，会调用 `createStore(reducer)`这个函数，这里同样，我们通过最简单的代码来理解一下它大概是个什么样的过程：

```js
function createStore(reducer) {
    let state;
    let listeners = [];

    let getState = () => {
        return state;
    }
    let dispatch = (action) => {
        let newState = reducer(state, action);
        state = newState

        // 调用所有的 listener
        listeners.forEach(li => {
           li();
        });
    }
    let subscribe = (listener) => {
        listeners.push(listener);
    }

    return { getState, dispatch, subscribe }
}
```

调用该方法就返回了一个 Store，这个对象里包含了上面我们说到的三个函数。

所以通过代码，我们知道了 Store 是一个中枢大脑，它提供了数据的存放（state），事件的分发（dispatch）以及订阅者的注册和调用（subscribe 和 dispatch 中的`forEach()`），可以说是 Redux 的核心。

2. Action


Action 更简单了，其本质就是一个 Plain Object，特别的，按照 Redux 的规则，这个对象必须带有一个名为`type`的 key，这一点是强制的，用以辨别该 Action 是做什么的。

下面是一个 Action 的例子：

```js
{
    type: 'SET_TITLE',
    title
}
```

使用时可以像如下方式调用：

```js
store.dispatch(action);
```

这样简单的方式来调用，`dispatch`即上面提到的 Store 提供的三大函数之一，它大概做了什么在之前的伪代码中也有描述。

由于 Action 只是一个对象，所以它其实可以放在代码的任何一个地方，例如我们可以放在顶层 Component 中，但这样会限制一个 Action 和一个 Component 强绑定在一起，重用会比较差，所以 Redux 的官方文档上也是建议我们将 Action 单独写在一个 JS 文件中，这里就要提到一个`actionCreator`的概念了，什么是`actionCreator`？

本质来说，`actionCreator`就是一个方法，方法返回一个 Plain Object，也就是 Action，所以`actionCreator`也就是如名字所说，是一个 Action 的生成器，这个方法可以单独放在一个新的 JS 文件中，这样这个方法也就得到了重用。

但是每次都这样手动地调用`dispatch`未免有些麻烦，所以这里有了`bindActionCreator`这样一个东西，它的目的就是简单地将`actionCreator`与`dispatch`绑定在一起，当我们调用`actionCreator`时，不需要再手动使用`dispatch`，例如如下这样：

```js
store.dispatch(actionCreator())
```

而只需要直接调用`actionCreator()`即可，是不是很神奇？？我们从代码的角度来看看 Redux 是怎样实现这个机制的。

```js
function bindActionCreators(actionCreators, dispatch) {
    let actions = {};

    Object.keys(actionCreators).forEach(creatorName => {
        actions[creatorName] = (...args) => dispatch(actionCreators[creatorName](...args)); 
    });

    return actions;
}
```

上面的伪代码只保留了核心流程，去除了异步 Action 的操作，便于理解。

其中可以看到，第一个参数`actionCreators`是我们传入的一个包含了所有我们希望进行**dispatch绑定**的 actionCreator 的对象，使用es6的语法即为import * as actionCreators from 'xxx'；第二个参数`dispatch`即为`store.dispatch`。

可以注意到，关键的一步在于，该方法会返回一个对象，这个对象包含`creatorName`为 key，一个函数为 value 。该函数帮我们 dispatch 了 action ，所以只要调用 bindActionCreators 返回的 actions 里的某一个方法，就可以自动的 dispatch 了，是不是非常简单？

当然 Redux 之所以强大，异步 Action 功不可没，在下一篇文章中，会再对异步 Action 以及 Middleware 进行介绍。

3. Reducer


前面的 Action 是一个对象，而这里的 Reducer 的本质就是一个函数了，这个函数接收上一次的 state 和 传入的 action，做一些逻辑处理之后返回新的 state。

一个标准的reducer是什么样的？

```js
function todoApp(state, action) {
    switch (action.type) {
        case SET_VISIBILITY_FILTER:
        return Object.assign({}, state, {
            visibilityFilter: action.filter
        });
        default:
        return state;
    }
}
```

这段代码是各种 Redux 教程文档中反复出现的一个标准 reducer 的代码，第一个参数是上一次的 state ，第二个参数是 dispatch 过来的 action（如果你还记得dispatch方法做了什么），可以看到 reducer 所做的事情无非就是接收 action，处理，塞给下一个 state。有一点我们需要注意，reducer 中必须拷贝一个 state，在拷贝的 state 进行修改，再返回新的 state，而不能直接修改原state。

当然我们不可能把一个应用的所有修改 state 的逻辑都放在一个 reducer 方法中，我们可以创建多个 reducer ，每一个 reducer 负责一部分具体的任务。所以这里就出现了需要将所有 reducer 合并成一个 reducer 的需求，因为我们最终 createStore 的时候只能接收一个 reducer。`combineReducer`就是 Redux 提供的完成该工作的函数。

一个简单的`combineReducer`是什么样的呢？还是像上面一样，我们用最简单的代码来看一看：

```js
function combineReducers(reducers) {
    return (state, action) => {
        let newState = {};

        Object.keys(reducers).forEach(reducerName => {
            newState[reducerName] = reducers[reducerName](state, action);
        });

        return newState;
    } 
}
```

代码中可以看到，该函数返回的依然是一个 (state, action) => newState这样的一个函数，这个函数签名正好对应的是 reducer 的函数签名，所以这样就很巧妙地将一堆 reducers 合并成了一个 reducer。与之前 reducer 稍不同的是，为了区别每一个 reducer 的内容，新的 state 会用 reducers 中的 reducer 的名字作为key，将一个特定的 reducer 处理后的 state 挂载在这个 reducer 下。

Redux 中很强调**纯函数 (Pure Function)**，目的是尽量少地产生 Side Effect，函数式编程又是一个更大的话题了，有兴趣的同学可以去看看 FRP 相关的文章或书籍。

具体到 Redux 中来说，官方文档中有如下的一句：
> Since changes always flow through reducers and reduces shouldn't mutate state...


那为什么必须是 Immutable 的呢？我自己的浅显理解是，在了解了 combineReducer 之后可以注意到，一个 action 被 dispatch 之后，如果 reducer 是合并过的，那么这个 action 将在**每一个 reducer 里都跑一遍**，又因为 reducer 的模型是纯函数的，函数式编程一个输入，一个输出，没有副作用，所以 reducer 返回一个新值可以说是函数式的一种体现，如果不是 Immutable 的话，在一次 dispatch 中某一个 reducer 中对 state 进行了一些原址修改，则会造成你根本无法追踪到底是哪里出现了问题。

以上也就介绍了 Redux 最重要的几个部分。

## 聊聊 Redux 之外非必须的部分

### redux-react

好像经过上面的介绍，我们已经能够很清楚 Redux 做了什么，但是想要更方便地使用 Redux，我们还可以利用上几个 Redux 之外非必须的东西。

首先我们来看看`redux-react`。它是什么呢？

我的理解是，其是一套针对 react 的自动绑定工具，但并不是必须的，如果不使用的话，你要使用 Redux 的话就需要将 Store 传给每一个子控件（因为我们可以知道，Redux 中几乎所有核心元素都需要用上 Store ），例如下面这样：

```js
render() {
    return (
        <View>
            <Text>{store.getState().title}</Text>
        </View>
    )
}
```

如果所有的控件都没有被封装的话，这样用也没什么问题，但是一旦出现了封装，你就需要一层一层的将 Store 传入，Store 也需要 subscribe 非常多 Component 的 render。

那`redux-react`为我们提供了什么便利呢？

首先它提供了一个`<Provider/>`组件，什么是`<Provider/>`？简单来说，`<Provider/>`就是一个接收 Store，然后将 Store 注入到它的子组件的组件，注入的方式是利用了 react 中的`getChildContext`方法，在上层组件中声明`getChildContext`，返回一个对象，这样在子组件中就可以通过`this.context.store`将 Store 取出。

还是通过简单代码来理解：

```js
class Provider extends Component {
    getChildContext() {
        return {
            store: this.props.store
        }
    }

    render() {
        return (
            this.props.childen
        )
    }
}
```

真实的`<Provider/>`是比较复杂的，这里只是最简单的通过代码表述`<Provider/>`做了什么。

但思考一下，我们其实要完成应用开发，得到 Store 只是手段而不是目的，我们只是要 Store 里的`getState`和`dispatch`而已，更进一步说，子组件没有必要拿到 Store，因为它并没有用到整个 State 树上所有数据的必要性，它只需要拿到和它渲染相关的数据就可以了。

这里就引出了`connect`，`connect`又是什么呢？

本质上来说，`connect`是一个把`state`和`dispatch`映射到 Component 的 props 上的一个柯里化函数，其首先接收两个函数类型的参数，分别是`mapStateToProps`和`mapDispatchToProps`，再接收一个组件实例作为参数，最终返回一个Component。

`mapStateToProps`这个函数负责了处理由 state 到 props 的映射关系，state 即 Store 中得到的 state，而 props，则是相对于通过`connect`出来的 Compoent，这样下来，这个 Component 就只会从 state 中取出自己需要用到的数据作为 props，有效地控制了复杂度，提高了健壮性。另外 Component 的子组件也不再需要关注 Store 这个概念了，所有需要的数据都由父组件负责传入。

`mapDispatchToProps`这个函数又是做什么的呢？我们回想一下，其实在使用过程中，我们需要的也并不是`dispatch`这个函数，虽然我们许多地方都需要用到他（例如 `dispatch` 一个 Action），我们需要的只是执行`dispatch`这个动作，结合之前提过的`bindActionCreators()`函数，这时我们可以直接将绑定好的`actionCreators`（即`bindActionCreators()`返回的对象）映射到 Component 的 props中，这样子组件只需要调用绑定好的actionCreator就可以自动dispatch了，例如下面这样：

```js
this.props.actions.setTitle(aTitle);
```

## 结语

通过本文，基本上将 Redux 的所有组件都探究了一遍，揭开面纱，用最简单的伪代码来帮助理解，希望能够将大家所认为的 Redux 的陡峭学习曲线拉直一些。因为作者本人是做 iOS 开发的，因为工作原因较早地接触到了 React Native，也接触到了 Redux，可以说 Redux 这样一种思想对我的启发还挺大的，通过限定一个 Workflow，减少 Side Effect，将原本随着代码规模增大而会变得很复杂很容易出错的渲染流程统一了起来，使用之后的渲染错误，debug 起来也有了固定套路，因为 reducer 永远都是纯函数的，只需要去 reducer 中打断点即可发现出错的原因，极大的增强了健壮性。这样一种函数式的思想也可以使用在我们日常的其他代码工作中，KISS (Keep it Simple and Stupid) 的原则值得我们更多的思考。
