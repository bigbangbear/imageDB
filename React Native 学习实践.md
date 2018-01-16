##### 前言

在 2016、2017年 前端技术快速发展，在客户端开发上也出现了 React-Native、Weex 等混合开发的技术，市场上一度叫嚣原生开发即将落寞。作为一个客户端开发得去了解该项技术，扩展自己对于行业技术发展的了解。因此自己在学习 React-Native 的过程当中尝试搭建了一个小项目，在这记录下这段时间所学习的内容，以及项目的搭建过程。

##### 基础知识

首先React-Native 开发是利用 JavaScript 语言开发的，所以 基本 JavaScript 和 DOM 知识必须了解。然后必须学习的基础知识首先是 React。推荐大家学习下阮一峰写的文章，学习后对于 React 将会有基础的认识。尤其是 ES6，对于熟悉与了解 Java 语言等面向对象语言同学，学习 ES6 会有熟悉的感觉，将会事半功倍。

[React 入门实例教程](http://www.ruanyifeng.com/blog/2015/03/react.html)

[ECMAScript 6 入门](http://es6.ruanyifeng.com/)

##### 环境搭建

安装官网提供的[步骤](https://reactnative.cn/docs/0.51/getting-started.html#content)一步步进行，在这里安利一波使用 WebStorm 进行开发（代码格式化、提示、集成了 React-Native 可以直接进行 debug 等优点），不过是收费。

##### 框架搭建

在程序开发的过程当中选择适合的开发框架有效的组织代码和安排内部逻辑，才能更好的开发与维护应用。例如，Android 中常用的 MVP、iOS 的 MVVM 等。在 React 开发中常用的开发框架为 Flux 与 Redux。推荐两篇文章了解下《[Flux 架构入门教程](http://www.ruanyifeng.com/blog/2016/01/flux.html)》与《[Redux 入门教程](http://www.ruanyifeng.com/blog/2016/09/redux_tutorial_part_one_basic_usages.html)》。在学习过程中了解到另一个框架 Mobx，这是一个简单的、可扩展的状态管理。特别构建适合中小型应用，推荐学习《[Mobx——React应用状态管理库](https://github.com/yuhuibin123/imageDB/blob/master/Mobx%E2%80%94%E2%80%94React%E5%BA%94%E7%94%A8%E7%8A%B6%E6%80%81%E7%AE%A1%E7%90%86%E5%BA%93.md)》做简单的了解。

首先贴一下整体项目的结构图。

![](https://github.com/yuhuibin123/imageDB/blob/master/image_article_react_native_1.png?raw=true)

其中，src 为根目录，dao 存放为数据相关模块；image 存放图像；module 为应用业务模块；network 网络请求模块；util 存放常用的工具类。

以登录功能介绍下，对于业务模块的开发架构设计。平时在开发当中，iOS 一般基于MVC (Model-View-View Controller) 的模式进行开发，MVC 架构本身并不复杂，但是 View Controller 负责胶水代码和业务逻辑的部分，而且很多 View 相关的部分也在这里，慢慢的在开发过程当中 View Controller 就会一天天的膨胀，到后面特别难以维护。结合 Mobx，就可以实现基于状态的单项数据流的函数式开发。为了更好的优化代码结构，将 UI 相关的部分独立在 Screen 中，将业务逻辑、数据获取等操作放置到 Store 中。通过 Action 函数进来两者之间的通信。

![](https://github.com/yuhuibin123/imageDB/blob/master/image_article_react_native_3.png?raw=true)

LoginScreen 为登录界面，主要进行页面的绘制以及其他 View 相关的用户交互处理相关的工作。代码模版如下：

```javascript
import {observer} from 'mobx-react/native'
import store from 'src/module/login/LoginStore';
@observer
export default class Template extends Component {
    constructor(props) {
        super(props);
    }
    render() {
        return (
            <View style={styles.container}>
           		 <TextInput style={styles.textInput}
                               placeholder="请输入密码"
                               onChangeText={(text) => {
          					      // 通过 Action 更新密码
                                   store.updatePassword(text);
                               }}
                               value={store.password}
                               secureTextEntry={true}
                    />
            </View>
        );
    }
}
```

每个界面都会利用 mobx 来管理状态工具，例如登录界面的 LoginStore。 Store 中的 observable 变量为抽象的状态，用户交互可以更新状态，mobx 会根据状态的不同自动的更新与渲染 UI。在这规定把所有操作状态的操作都封装在 Store 中。

```javascript
import {observable, computed, action} from "mobx";
class LoginStore {
    @observable password = '';
    constructor() {
    }
    @action
    updatePassword(password) {
       this.password = password;
    }
}
const store = new LoginStore();
export default store;
```

##### 项目组建

一个应用的开发需要许多不用的组件，例如导航栏组件、图像选择组件、以及各种 View 的组件。这里介绍下本次开发过程使用到的库。在平时开发中如果需要其他库支持，推荐到 [awesome-react-native](https://github.com/jondot/awesome-react-native) 上寻找，可以避免重复造轮子以及学习别人的实现。

1. 导航栏

   React-Native 提供了导航栏的组件，例如 NavigatorIOS\ToolbarAndroid ，但是没有提供一套可以在同时两端运行的组件。因此，对比了现在开源的导航栏组件库。选择了[react-native-navigation](https://github.com/wix/react-native-navigation) 作为基本的组件。选择该库的原因：（1）完全由原生代码实现了 （2）使用文档详情 （3）接口设计简单。

   在根目录下的 app.js 文件中，进行模块页面注册，导航栏启动等工作。具体的配置需要查看开发文档。

   ```javascript
   // 页面注册
   export function registerScreens() {
     Navigation.registerComponent(loginScreen, () => LoginScreen);
   }
   registerScreens()
   // 启动单页面的导航的应用
   Navigation.startSingleScreenApp({
       screen: {
           screen: loginScreen, 
           title: '登录', 
           navigatorStyle: {}, 
           navigatorButtons: {} 
       },
   });
   // 启动基于 Tab 的导航应用
   Navigation.startTabBasedApp({
           tabs: createTabs(),
           appStyle: {
               tabBarSelectedButtonColor: '#000',
               tabFontFamily: 'BioRhyme-Bold',
               forceTitlesDisplay: true
           }
       });
   ```

2. UI 组件库

在界面开发的过程当中需要运用不同的组件，例如文本，按钮，搜索框等多种组件。但是 React-Native 只提供了基本的组件，在使用过程当中不是特别友好。例如以按钮为例，只能通过自定义  Text 组件与 TouchableHighlight 结合使用才能构造出一个按钮：

```javascript
<TouchableHighlight style={styles.btn} onPress={this._onPress}>
                    <Text style={styles.btnText}>
                        我是按钮
                    </Text>
</TouchableHighlight>
```

在这推荐 8000+ stars 的 UI 组件库--[react-native-elements](https://github.com/react-native-training/react-native-elements)，该库封装了常用的组件例如按钮、头像、复选框。利用组件库后代码可以优化成如下

```
<Button buttonStyle={styles.btn} title="登录" onPress={this._onPress}/>
```

3. 其他常用组件库

图像选择组件：react-native-image-picker

时间选择组件：react-native-datetime

toast 提醒组件：react-native-simple-toast

iPhone X适配：react-native-iphone-x-helper

在开发的过程当中，需要对于常用的控件、组件进行封装，优化代码结构、提高开发效率。在使用开源组件的过程当中也需要主动学习才能更好的进步。

数据库：react-native-realm

4. 网络请求封装

客户端应用免不了需要通过网络请求操作与服务器交互，根据以往的经验，需要验证的接口都需要带上 Token 等验证信息，以及对于接口异常的处理。在这里也总结下自己在网络请求上进行的一些简单封装来满足开发的需要。Token：现在 App 大多数都是通过 token 机制来验证用户的安全性。在登录成功后会分配唯一的 token 给客户端。React Native 中网络请求的方法 fetch 默认会自动持久化并携带 cookie。因此想通过把 Token 存储在 cookie 中，当需要的时候取出 token 进行网络请求。这部分已经有了管理 cookies 的开源库 [react-native-cookies](https://github.com/joeferraro/react-native-cookies)，通过该库将获取到的 token 保存到 cookie 持久化文件当中。每次打开应用的时候取出 token 会赋值给 Global.token ，这样直接就可以在请求中使用。

```
export default class Network extends React.Component {
    static post(url, params, callback) {
        //fetch请求
        fetch(url, {
            method: 'POST',
            headers: {
                'token': Global.token
            },
            body: JSON.stringify(params)
        })
            .then((response) => response.json())
            .then((responseJSON) => {
                callback(responseJSON)
            }).done();
    }
}
```

##### Native 与 原生 通信

在学习 React-Native 的过程当中，推荐大家了解下它的通信机制。因为在平时开发中与 H5 进行交互的时候，都是通过 JSBridge 来进行通信的，例如 iOS 是利用 JavaScriptCore 提供的特性来进行 JS 与 OC 的通信。而 React-Native 则是自己实现了一套机制进行通信。可以阅读下 bang 写《[React Native通信机制详解](http://blog.cnbang.net/tech/2698/)》文章了解原理。在这里介绍下，两者之间如何进行通信，以 iOS Native 与 RN 通信为例。

1. 在 iOS Native 中创建通信管理文件 RNMessageManager，该类继承 RCTEventEmitter（RCTEventEmitter 是在 0.27 版本之后出现的消息发送类）。头文件如下：

```objective-c
#import <React/RCTBridgeModule.h>
#import <React/RCTEventEmitter.h>
@interface RNMessageManager : RCTEventEmitter<RCTBridgeModule>
+ (void)emitEventWithPayload:(NSDictionary *)payload;
@end
```

2. 实现消息生成类。包括申明所有的消息发送方法与实现消息发送方法

```objective-c
// 声明消息发送方法
- (NSArray<NSString *> *)supportedEvents {
  return @[@"sendMessage"];
}
// 实现消息发送方法
-(void)sendMessage:(NSDictionary*) payload  
{  
  [self sendEventWithName:@"sendMessage"  
                     body:payload];  
}  
```

3. JS 端监听消息

```
import {
    NativeEventEmitter, NativeModules, NativeAppEventEmitter,
} from 'react-native';
const {RNMessageManager} = NativeModules;

export default class Template extends Component {
  componentWillMount() {  
         this.messageManager = new NativeEventEmitter(RNMessageManager);
         this.messageManager.addListener(
            'sendMessage',
            (payload) => {
            	// 收到的消息
            }
         );
    }  
    componentWillUnmount() {  
      this.messageManager && this.messageManager.remove();  
      this.messageManager = null;  
    } 
}
```

##### 应用发布

在完成应用开发后，需要对应用进行打包、测试、签名以及发布。其中 Android 可以利用开发工具 Android studio 进行签名、生成安装包，就可以进行自主的应用发包或者找应用市场进行发布。相比之下 iOS 应用的上架则麻烦许多。例如 iOS 上架过程需要使用到各种证书，Provisioning Profile，entitlements，CertificateSigningRequest，p12，AppID等概念，推荐阅读《[iOS App 签名的原理](http://wereadteam.github.io/2017/03/13/Signature/)》详细了解这些概念。

（1）申请苹果开发者账户，只有拥有开发者账号才能进行打包、上传 [参考链接](https://www.jianshu.com/p/9b994a019ee6)。

（2）iOS开发证书配置和安装 [参考链接](http://blog.csdn.net/xiaojiuxing_csdn/article/details/50963430)

（3）自动化发布。为了方便测试，提供开发效率，通过脚本工具完成自动化打包上传到 fir 平台上的工作。

（4）应用上架。

##### 实践

为了检验本次的学习成果，将以前开发的 Android 应用利用 React Native 重写了一遍后成功上架到 AppStore。在这个过程当中，不单单学习到许多的 React-Native 开发知识，更重要的是深化了对 iOS 开发的了解，从申请证书 、打包管理、上架、审核步骤等都有了认识。让自己在这个过程当中有了许多的进步，对于平时的 iOS 开发工作有了新的认识。此外利用 React-Native 制作简单的应用还是具有天然优势，开发速度快，跨平台，而且支持热更新，期待它能够有更好的发展。
