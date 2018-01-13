#### **Mobx——React应用状态管理库**

​	React是自带View与Controller的库，那么在具体的项目开发中通常还需要一套机制去管理组件与组件之间，组件与数据模块之间的通信。从而一些基于Flux架构思想的框架被广泛应用于各个产品中，例如Redux、Reflux等。但是，在项目具体开发中由此产生了一些问题。例如设计阶段没有考虑周全，开发时就要不停的在action，container，reducer之间修改，增加了编码过程的复杂度。其实Redux等框架适合于大型项目，只有当React不能解决问题的时候，再考虑引入Redux框架。后来在阮一峰的文章中了解到Mobx状态管理库，它与React结合能够很好的管理应用的状态、组件与数据模块之间的通信。下面就介绍下这个库。

##### Mobx简介

* Mobx是一个简单的、可扩展的状态管理库。
* 在React中，把组件看成是一个状态机，通过用户的交互更新组件的State，然后根据新的state重新渲染用户的界面。React引入Virtual DOM机制优化渲染UI过程中复杂繁琐的DOM操作。而Mobx提供一个机制来存储与管理应用状态。它通过虚拟依赖状态图来管理应用状态与React组件之间的同步。
##### Mobx核心概念与API
* ![](http://i1.piimg.com/567571/8b1b02a13eea005d.png)

* **State**：被观察的状态（Observable state）

  * 用来描述应用的状态，用户交互可以更新state，从而更新与渲染UI。state可以是对象、数组等。在mobx中。利用`observable`修饰类中的某个属性为状态，例如标题状态：

    ```
    class Store {
    	    @observable title = '';
    }
    ```

* **Actions**：引起状态变化的动作

  * Mobx将会确保由Actions导致状态的变化被derivations和reactions自动的处理。Mobx建议将那些会改变状态的函数标记为`action`，例如设置标题:

    ```javascript
     @action setTitle(title) {
            this.title = title;
     }
    ```

* **Derivations**: 派生

  * 派生指的是自动的根据state 计算后返回的结果。比如未完成的任务的数量，或者是根据多个网络请求得到的页面的整体数据。Mobx利用`computed`修饰get函数表示为从state中的派生。例如，为标题添加修饰返回新的标题：

    ```java
    @computed get getTitle() {
        return '修饰后的标题'+this.title ;
    }
    ```

* **Reaction**：附加动作

  * Reaction是由Actions导致状态变化而同时出现的副作用。与Derivations相似，但是reaction是没有返回值。可以用于打印日志、网络请求等。例如：

    ```
    autorun(() =>{
        console.warn(store.title);
    })
    ```

  在了解Mobx的核心概念后，下面用简单的demo演示mobx的使用。

##### 在项目中安装mobx

1. 通过npm安装需要的依赖：mobx与 mobx-react

   ```
   npm i mobx mobx-react --save
   ```

2. 让工程支持ES7的decorator特性。 decorator 是一种语法糖，它可以用更优雅的方式实现同样的函数变换。

   首先，安装 babel 插件

   ```
   npm i babel-plugin-transform-decorators-legacy babel-preset-react-native-stage-0 --save-dev
   ```

   然后，在 .babelrc 文件配置 babel 插件

   ```
   {
    'presets': ['react-native'],
    'plugins': ['transform-decorators-legacy']
   }
   ```

##### Mobx的使用

* 在配置好项目好后，利用简单的计数器应用展示Mobx的状态管理功能与特性。

* ![Demo](http://i1.piimg.com/567571/2e76221bb5d71248.png)

* 首先，创建一个Store文件存储应用的状态。在应用的状态是数字，加与减的用户交互动作将会引起状态的变化。代码如下：

  ```javascript
  //Store.js
  import {observable, computed, action, autorun} from 'mobx';

  class Store {
  	//State
      @observable numbs = 0;
  	//Action
      @action plus() {
          this.numbs = this.numbs + 1;
      }

      @action minus() {
          this.numbs = this.numbs - 1;
      }
  	//Derivation
      @computed get getNumbs() {
          return '计数：' + this.numbs;
      }
  }
  const store = new Store();
  export default store;
  //Reaction
  autorun(() => {
      console.warn(store.numbs);
  })
  ```

* 然后，写一个组件来展示计数器的功能。利用Mobx观察状态的变化来更新与渲染UI。当用户点击按钮，触发状态的变化，会通知组件进行更新得到新的计数结果。

  ```javascript
  import {observer} from 'mobx-react/native'
  import store from './Store';

  //组件需要被observer修饰，才能响应Store中状态的变化
  @observer
  export default class extends Component {

      render() {
          return (
              <View style={{flex: 1, alignItems: 'center', justifyContent: 'center'}}>
                  <Text>{store.getNumbs}</Text>
                  <Text>{store.getStoreCount}</Text>
                  <View style={{flexDirection: 'row'}}>
                      <Button title="减" onPress={() => store.minus()}/>
                      <Button title="加" onPress={() => store.plus()}/>
                  </View>
              </View>
          );
      }
  }
  ```


##### 写在最后

​	在刚接触到Mobx的时候，它带来响应函数式编程的便利与状态管理的新的思想让我迫不及待的在项目中进行应用。它也很好的证明，能够优化项目的结构，提高编程的效率与应用的性能。推荐大家在开发中可以尝试下Mobx。