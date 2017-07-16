Ouranos 前端可视化系统
===

## 0. 目标

- 对简单需求能让产品、运营、设计能够直接参与到前端UI层面，减少总体开发量
- 沉淀出基础UI组件，对于复杂需求前端能够快速响应并使得总体UI风格统一

## 1. 调研

现有的前端可视化系统中，比较强大且灵活性突出的以XCode的Interface Builder和Visual Stdio的WPF为首，这两套系统针对开发者，减少了开发者大量的UI层面的代码。尽管Interface Builder和WPF主要以开发为主，但是我们可以比较好的借鉴两者的思想用于Ouranos中，达到可视化系统灵活性和功能强大的目标，以下主要以Interface Builder来作为主要介绍。

XCode的Interface Builder其实现思想十分简单，主要依靠两个思想达到灵活性：

1. 提供最基础的UI组件
2. 依靠XML作为Interface Builder的中间UI表述，便于分离可视化工具和代码生成的逻辑

依靠这两个思想，iOS对于UI层面的编写既可以进行使用纯代码方式，也可以进行拖拽并绑定后完成，其中可视化拖拽的生成代码的步骤总结为：

1. Interface Builder中进行可视化操作
2. 生成XML表述拖拽组件
3. 根据XML使用Code Genrator进行代码生成

这样的方式的好处也十分明显，依托XML这个中间UI表述，其生成的形式可以不局限于某种语言以及UI框架，基础组件可以很容易的被替换，其实现是比较优雅的。

## 2. 技术选型

- UI中间表述语言：JSX
- UI库：React
- 数据框架：Redux
- Schema：React.PropsType

使用XML作为中间UI表述是比较优雅的方式，很好的提供了组件层级这一要求，通过遍历XML进行组件代码的生成是比较容易的，但由于现在Web Component的实现在现有的UI框架中比较常见，因此我们将XML->Web Component的表述也是可以的，其中JSX与XML是十分亲和的，同时JSX可以很容易的被现有的JSX Compiler编译成React代码，因此我们很自然的就选择了JSX来代替XML。

UI库的选择个人喜好选择了React（你也可以选择Vue）。

传统的可视化系统中基本上很少有做组件间事件关联的，其原因在于通知模型是事件分发。要让可视化系统达到灵活，那么组件间事件关联是必不可少的。React中对于父子组件通知/广播这一需求除了使用事件模型外，单一数据流prop传递重渲染子组件的方式更为简单可靠。因此我们需要Redux这一框架提供单向数据流的能力，让通知从事件分发转移到模型角度，减少组件开发中关于组件事件通知的复杂度（你也可以选择Vuex）。

Schema在可视化平台中是必要的，能够比较清晰的描述整个组件所依赖的数据，React自身可以使用React.defaultProps进行描述。

接下来会从以下几个方面简要介绍Ouranos其中的核心思想：

0. 目录结构
1. 混合基础组件的Ouranos组件
2. 中间UI表述语言JSX的decode和encode
3. 组件拖拽和显示层级
4. 使用Redux达到组件关联通知及广播
5. 数据请求的集成
6. 愉快的进行SSR

但针对Ouranos，我们可以总结一句话为主旨：通过中间UI语言的转化和组件模型来减少整个可视化系统的复杂度，View is just a result。

## 3. 目标结构

整体的目录结构是比较简单的，遵循以往组件化的方式进行目录结构组织即可
```shell
└─component
    └─Slider
        └─images
        └─Slider.css
        └─Slider.js
        └─Slider.schema.js
```

## 4. 混合基础组件的Ouranos组件

Ouranos组件是基于基础组件的，例如：
```javascript
class Slider extends React.component{
    render(){
        return (
            <Ant-Slider></Ant-Slider>
        )
    }
}
```
但是为了让可视化系统能够获取到这个组件的相关信息，例如可以被设置的数据，可以被关联的方法等等，因此我们需要让Ouranos组件继承一个Ouranos的基础组件。其次也要提供schema信息，方便Redux的Store完成初始化。

Ouranos的基础组件我们假设为BaseComponent，其必要的属性和方法如下：
```javascript
class BaseComponent extends React.component{
    constructor(props){
        super(props)
    }

    // 用于生成JSX的中间UI表述
    genJSXStr(){
        // sth to code...
    }
}
```
对于一个Ouranos组件，需要给外部系统提供三个信息：
- styles: 可被设置的组件样式
- props: 可被设置的额外属性
- letouts: 可被关联的方法
因此我们的一个Ouranos组件在编写时大致如下代码：
```javascript
import BaseComponent from '../BaseComponent'
import defaultProps from 'Slider.schema'

@register
class Slider extends BaseComponent{
    static defaultProps = defaultProps
    static defaultStyles = BaseComponent.defaultStyles

    constructor(props){
        super(props);
    }
    
    @letout
    show(){
        // sth to do...
    }

    @letout
    hide(){
        // sth to do...
    }
}

```
defaultProps承担了暴露可被设置的额外属性和默认组件默认属性的职责，defaultStyles代表此组件可被设置的组件样式，我们使用@letout提供组件的可被关联的方法。最后使用@register将组件暴露在全局空间供可视化系统调用。

@letout的实现大致为:
```javascript
function letout(base, key ,desc){
    base.defaultLetouts = base.defaultLetouts || {};
    base.defaultLetouts[key] = key;
    return desc;
}
```
当我们进行关联组件方法的时候，实际上是对函数的name这一属性进行触发，然后由Redux的机制进行函数触发，[7.使用Redux达到组件关联通知及广播]有详细介绍。

@register的实现大致为：
```javascript
function register(target){
    let components = window.__OURANOS__COMPONENTS__  || {};
    let fnName = target.toString().match(/function\s+(.+?)\s*\(/)[1];
    
    window.__OURANOS__COMPONENTS__ = Object.assign(components, {
        [fnName]: target
    });
    
    return target;
}
```
至此，就完成了Ouranos组件关键信息的暴露，以及组件在可视化系统的注册工作。


## 5. 中间UI表述语言JSX的decode和encode
在不考虑拖拽的情况下，我们来考虑中间UI表述的问题。JSX作为中间UI表述的好处在于我们可以很容易的表述UI的各方面信息，并且很容易和React进行集成，但与此同时也出现了一个问题，JSX是不容易被修改的，例如，可视化系统进行初始化后我们新增一个组件后进行相关属性的修改，那么这个组件在JSX这个数据结构中是相当麻烦的，因此我们需要有一个与之等价的表述，能够很方便的进行修改，并且能够与JSX进行很好的转换，在这里我们选择JSON。比如存在一个JSX：
```xml
<Container>
    <Slider></Slider>
</Container>
```
通过这个JSX我们可以转换成等价的JSON：
```javascript
[{
    component: Container,
    instance: null,
    props: {},
    styles: {},
    letouts: {},
    parent: null,
    children: [1]
},{
    component: Slider,
    instance: null,
    props: {},
    styles: {},
    letouts: {},
    parent: 0,
    children: []
}]
```
当我们需要在Container组件进行增加一个Button组件时，我们很容易可以完成这个操作：
```javascript
[{
    component: Container,
    //...
    children: [1,3]
},{
    component: Slider,
    //...
},{
    component: Button,
    instance: null,
    props: {},
    styles: {},
    letouts: {},
    parent: 0,
    children: []
}]
```
因为有了parent和children字段，因此对于其他组件的删改也都能很好的支持。同时JSON的格式也很好的存储在NOSQL数据库中，非递归嵌套形式也更容易被diff和查询。
假设我们现在从数据库中获取到了上面的JSON格式的表述，系统在初始化时需要将上面的数据转换成JSX然后进行React的初始化是十分容易的：
```xml
<Container index={0}>
    <Slider index={1}></Slider>
    <Button index={2}></Button>
</Container>
```
这里我们给每个组件传递了index表述这个组件在JSON中的具体位置，方便React的instance进行引用（比如组件删除时需要更新JSON，然后进行整体的重渲染），同时也需要让JSON的instance字段引用对应React的instance，其目的是便于对React的DomElement做相关设置（比如样式的设置无需走Redux的流程），因此对于BaseComponent我们需要做一点修改:
```javascript
class BaseComponent extends React.component{
    constructor(props){
        this.index = props.index;
        window.__OURNAOS__JSON__[props.index].instance = this;
        //...
    }
}
```

## 6. 组件拖拽和组合嵌套
对于可视化系统来说，拖拽和组件嵌套组合是不可少的。对于拖拽来说，现有Github中有比较成熟的实现，比如[react-dnd](http://react-dnd.github.io/react-dnd)，但是其实现并不是Dom的移动，而且子组件的create和delete（可以查看对应[tutorial](http://react-dnd.github.io/react-dnd/docs-tutorial.html)）。由于我们的Container并不可知下面的子组件有哪些，并且不能每产生一个组件就维护Container显示与否的逻辑，因此使用react-dnd是不可行的。其原因其实在于React的Virtual Dom Diff的实现，对于拖拽和组件嵌套组合是不易于实现的，因为可能会涉及到组件的跨层级，但Virtual Dom Diff为了性能，本身就只是同一层级Virtual Dom的比较。但好在Virtual Dom是完全隔离于Dom的，因此我们可以将其视为透明层，而只考虑React中DomElement的情况。

对于DomElement的拖拽和组合嵌套等有很成熟的方式，在这里不赘述，我们需要在BaseComponent中增加一个方法，用于实现Dom的拖拽:
```javascript
class BaseComponent extends React.Component{
    //...
    componentDidComponent(){
        super.componentDidComponent();
        let domEle = ReactDOM.findDOMNode(this);
        // domElem drag and drop event
    }

    componentWillUnmount(){
        // destory your drag and drop event
    }
}
```
但这里需要注意一点，当涉及到跨层级时，比如Slider从Container1移动到Container2时，我们只改变了Dom，但是并没有改变Virtual Dom，因此在drop的时候我们需要修改依赖的JSON，然后重新生成新的JSX然后再进行一次初始化的重渲染，以保证Virtual Dom也得到了变化。

## 7. 使用Redux达到组件关联通知及广播
通过前面的描述，我们已经可以很好的对的styles和props进行设置了，对于styles的设置十分简单，由于我们的组件和JSON是互相引用的，因此整个过程可以被描述为：
1. 用户点击对应组件
2. 获取组件prototype上暴露的可设置styles信息
3. 渲染styles设置面板
4. 获取现有JSON中设置了的styles属性进行相关填充
5. 用户对styles进行设置后确认
6. 设置JSON数据的styles，然后通过instance获取到其DomElem，进行样式的设置

而对于props的相关设置也比较简单，首先我们先介绍props在Redux.Store上的存储方式。Store上存储了所有已经实例化组件的相关props，其结构可以被描述为：
```javascript
{
    'Container_0':{
        // container props
    },
    'Slider_1':{
        // slider props
    },
    'Button_2':{
        // button props
    }
}
```
store每个item的key值实际上是componentName+object_id，当涉及到props的修改时，我们直接使用store.dispatch对应组件的props即可，其更新过程可以被描述为：
1. 用户点击对应组件
2. 获取组件prototype上暴露的可设置的props信息
3. 渲染props设置面板
4. 获取现有JSON中设置了的props属性进行填充
5. 用户对props进行后确认
6. 更新JSON数据的props，然后使用store.dispatch进行组件的渲染

现在我们最后剩下的是处理letouts，实际上letouts的处理也是通过模型数据的变化进行的，比如Button的click操作关联了Slider的show方法，那么我要做的就是修改store上的Slider_1的show变量，然后当Slider的componentWillReceiveProps方法判断nextProp的show变化后，触发对应的show自身的show方法即可，我们修改后的BaseComponent如下：
```javascript
class BaseComponent extends React.Component{
    componentWillReceiveProps(nextProps){
        const letouts = this.constructor.prototype.defaultLetouts;
        for(let key in letouts){
            if(this.props[key] !== nextProps[key]){
                this[key].call(this);
            }
        }
    }
}
```
而store.dispatch的对应letout值只需要不断递增即可，此种实现比较简单，易于实现。当需要注意的是，当绑定了对应的letout时，JSON里面的letouts数据也需要进行更新。

## 8. 数据请求的集成
由于使用了Redux，因此对于数据请求的集成式十分简单的，我们只需要按照约定在componentDidMount中进行数据请求，然后store.dispatch相关的数据即可完成props的更新，其思想于设置props如出一辙。

## 9. 愉快的进行SSR
当整个页面完成时，我们需要对整个页面进行发布，在发布的整个过程中，我们只需要关注JSON即可，例如，当我们有这样一个页面：
```javascript
[{
    component: Container,
    instance: null,
    props: {},
    styles: {
        x: 100,
        y: 100
    },
    letouts: {},
    parent: null,
    children: [1,2]
},{
    component: Slider,
    instance: null,
    props: {
        image: 'images/1.png'
    },
    styles: {
        x: 100,
        y: 100,
        background: 'red'
    },
    letouts: {},
    parent: 0,
    children: []
},{
    component: Button,
    instance: null,
    props: {},
    styles: {},
    letouts: {
        'show': 'Slider_1'
    },
    parent: 0,
    children: []
}]
```
当我们转化为JSX时，实际上是类似这样的形式：
```xml
<Container index={0} styles={{x:100,y:100}} letouts={}>
    <Slider index={1} styles={{x:100,y:100}} letouts={}></Slider>
    <Button index={2} styles={} letouts={{show:'Slider_1'}}></Button>
</Container>
```
由于props需要在Redux初始化时给store进行处理，因此我们也同时生成store的初始化结构：
```javascript
{
    'Container_1':{},
    'Slider_1':{
        show: 0,
        images: 'images/1.png'
    },
    'Button':{}
}
```
然后通过后端的node compiler将这些数据进行template的替换，插入到预先设置的模板位置即可（开发预览可以在前端eval进行）。

当进行SSR的时候，也十分简单，因为Redux很容易进行SSR，并且我们有对应的JSX代码以及Store初始化数据，并且对应的数据请求都在每个组件的componentDidMount中（SSR不会调用此方法），因此我们可以很容易的进行SSR渲染，对于特殊的逻辑，再由前端触发componentDidMount的数据请求进行处理。






