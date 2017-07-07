在[上一篇](./1-intro.md)中对实现视图层进行了一些思考和实践，总结出应该将View与Model拆分开来。

上篇中将拆分View与Model的目的简单的总结为 **解藕** 与 **组合**,
本篇将分别深入讨论View与Model的拆分目的与实现思路。

# View
View要做到解藕，首先要与Model隔离开来。因为Model可能是会变化的，
一旦Model发生变化，View相关代码就需要跟着变动，此为影响解藕成功与否原因之一。
其二，View如果与Model绑定，那么一个View就只能展示与该Model相关的视图。

假设现有一个需求，渲染用户列表、渲染订单列表。
```TypeScript
// 用户数据结构
interface UserDesc {
    id: String,
    name: String
}
// 订单数据结构
interface OrderDesc {
    id: String,
    title: String
}
```
代码大致如下:
```TypeScript
type userList = Array<UserDesc>
type orderList = Array<OrderDesc>
// 渲染用户列表
<ul>
{
    userList.map(mod => (<li>{mod.name}</li>))
}
</ul>
// 渲染订单列表
<ul>
{
    userList.map(mod => (<li>{mod.title}</li>))
}
</ul>
```
因为两个Model定义不同，则不得不书写重复的代码。
此时的渲染过程为`Model -> View`

解决这个问题也很简单，既然渲染结构相同，只是数据不同，那么将两份数据映射成
相同的结构即可，只需要加一个中间数据层，我们可以考虑一下，如何设计一下新的View。

此时需要我们跳出View与Model关联的思维方式，View有其自己的固定格式数据。
无需理会外面的数据是什么样子，如果要使用该View，就需要提供这样的数据格式。

人们将渲染View需要的数据称之为ViewModel，也就是上文中的中间数据层，View则可以如下设计：

```TypeScript
interface ViewListDesc = {
    text: string
}
type ViewModel = Array<ViewListDesc>
function renderListView(vm: ViewModel):string{
    return (
        <ul>
        {
            vm.map(v => (<li>{v.text}</li>)))
        }
        </ul>
    )
}
```
渲染如下：
```TypeScript
const user_vm = userList.map(m => ({text: m.name}))
const order_vm = orderList.map(m => ({text: m.title}))
renderListView(user_vm)
renderListView(order_vm)
```
此时渲染过程为`Model -> ViewModel -> View`，View与Model完全独立，
如果要使用该View，必须将数据转为ViewModel。

# Model

Model功能主要包含数据获取与存储，以及一些简单的处理。
一般而言，Model保持其纯粹是一个比较好的实践，因为可以提高其复用的可能性。

实现Model分离之前，需要考虑接口的统一管理。
将一个Model相关的接口使用单例设计模式来处理，这样可以解决两个较为头疼的问题：
＋ 同一Model多处定义
＋ 同一接口返回值使用不同的方式处理

我个人认为这是一个前端工程规划是否合理的重要考核指标之一。

此处需要做两件事情：
＋ 统一接口返回内容的结构
＋ 封装接口类

## 返回值结构定义
接口返回内容需要包含如下信息：
```TypeScript
interface ResponseStrct<T> {
  success: boolean
  code: number
  message: string
  result: <T>
  type: string | number
}
```
`code`字段一般要求与`http`状态保持一致，这有助于后期日志处理。
`type`字段标识`result`数据类型，根据业务需要，可以扩展其他字段。

前端可以依据该接口做统一的预处理，比如检测`success`值显示`message`。

## 统一接口定义
接口包含一个url，以及一些调用方式，诸如post get put。由于服务器选型方案不同，可能需要解析url(Restfull或graphQ)，还可能需要对返回值进行简单的处理，比如返回json。
```TypeScript
interface Resource<T> {
  new(url:string)
  get(params:any):Resource<T>
  post(params:any, body:any):Resource<T>
  json():Promise<ResponseStrct<T>>
  text():Promise<ResponseStrct<T>>
}
```

接口还有一个比较重要的情况需要考虑：输入验证。
每个接口都需要对输入进行验证，包括数据类型、结构、有效性等方面，数据库字段有还包含长度、取值范围或enum值验证。
验证的位置包括用户输入、客户端验证、服务端验证等，而这些验证逻辑必须保持一致。
所以验证代码最好使用单独的模块实现。

基本的验证类型包含type、enum、required、in、oneof、bound等。
React使用的PropTypes接口设计得比较好，可以衍生出如下结构以适应接口参数验证。
```TypeScript
inerface ChekResult {
  message: string | null
  success: boolean
}
interface Checker {
  (props: any, name: string): CheckResult
  isRequired(props: any, name: string): CheckResult
}
interface DefineTypes {
  [key:string]: Checker
}
declare function defineTypes(types: DefineTypes):(props:any) => CheckResult
interface PropTypes {
  string: Checker
  number: Checker
  array: Checker
} 
```
使用如下
```TypeScript
const checker = defineTypes({
  str: PropTypes.string,
  num: PropTypes.number.isRequired
})
const result: CheckResult = checker({
  str:'s', 
  num: 1
})
```
`checker`定义后，可多处使用。

TODO