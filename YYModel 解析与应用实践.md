> 在看过 ibireme 的一篇关于JSON 模型转换库的文章后，了解到 YYModel 与其他类库相比在性能上有较大的优势。而网易有钱 App 运行过程中需要大量的 JSON 模型转换，此外数据库查询也依赖于此。于是想从该方面入手进行项目性能优化。本文记录自己学习 YYModel 源码的过程与在项目中进行实践的成果。

##### 基础理论

Model 转换的算法流程：首先，算法利用 Runtime 的特性，在程序运行时获取对象的信息（属性、方法等）；然后根据映射关系将 JSON 中的值赋值给 Model 对应的属性。

![屏幕快照 2018-01-09 18.53.16](http://ww4.sinaimg.cn/large/0060lm7Tly1fnar83kcvcj30dw03ajrv.jpg)

如下代码所示，通过 Runtime 获取到对象类的属性列表（ @property 申明的属性）。代码运行的结果：输出 XXTest 中名称为 name 的 property 所具有的属性，打印结果为 ：T@"NSString",C,N,V_name。在解释字符串含义之前，先介绍下类型编码（[Type Encodings](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html)）。作为对 Runtime 的补充，编译器将每个方法的返回值和参数类型编码为一个字符串，并将其与方法的 selector 关联在一起。Runtime 内部利用类型编码帮助加快消息分发。打印的字符串返回的结果为 property 各个属性的类型编码。

查看 property 类型编码表即可以对字符串进行解读：总是以 T 开始，@ 表示 property 是一个对象，NSString 为类型的名称，C 为属性为 copy，N 代表非原子性访问（nonatomic）,V_name 为属性的名称。通过该特性即可获取到所有的属性，Runtime 为动态的将 JSON 转换为 Model 提供了技术支持。

```objective-c
@interface XXTest ()
@property (nonatomic, copy) NSString *name;
@end
@implementation XXTest
+ (void)printAllPropertiesAndVaules
{
    // Runtime 返回的对象类的属性列表(@property申明的属性)
    objc_property_t *properties = class_copyPropertyList([self class], &outCount);
    for (i = 0; i < outCount; i++) {
        objc_property_t property = properties[i];
        const char *propertyName = property_getName(property);
        const char *propertyAttribute = property_getAttributes(property);
                NSLog(@"property name is %@, attributes is %@", @(propertyName), @(propertyAttribute));
    }
}
```

##### YYModel 源码简析

通过 YYModel 整体结构图对其做整体的认识。

![](http://upload-images.jianshu.io/upload_images/1618300-75c9bb5981fbe16d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)


1. 首先，根据前面列的 Model 解析步骤来看一下 YYModel 解析的过程。YYModel 将 Model 所属的类、属性等信息封装成元对象  `_YYModelMeta`。元对象会会利用这些数据进行数据转换。元对象的构造函数的伪代码如下：

```objective-c
- (instancetype)initWithClass:(Class)cls
{
    // 生成对象描述信息
    YYClassInfo *classInfo = [YYClassInfo classInfoWithClass:cls];

    // 生成 model 的属性数组
    NSMutableDictionary *allPropertyMetas = [NSMutableDictionary new];
    // 从子类遍历到基类获取所有的属性 _YYModelPropertyMeta
    while (classInfo && classInfo.superCls != nil) {
        for (YYClassPropertyInfo *propertyInfo in curClassInfo.propertyInfos.allValues) {
            _YYModelPropertyMeta *meta = [_YYModelPropertyMeta metaWithClassInfo:classInfo
                                                                    propertyInfo:propertyInfo
                                                                     generic:genericMapper[propertyInfo.name]];
            allPropertyMetas[meta->_name] = meta;
        }
        curClassInfo = curClassInfo.superClassInfo;
    }
   // 设置 json 与 model 的映射关系
  if ([cls respondsToSelector:@selector(JSONKeyPathsByModelPropertyKey)]) {
        // 映射关系字典
        NSDictionary *customMapper = [(id <YYModel>) cls JSONKeyPathsByModelPropertyKey];
        [customMapper enumerateKeysAndObjectsUsingBlock:^(NSString *propertyName, NSString *mappedToKey, BOOL *stop) {
            _YYModelPropertyMeta *propertyMeta = allPropertyMetas[propertyName];
           // 保存 JSON 到 Model 的映射关系
           propertyMeta->_mappedToKey = mappedToKey;
            if ([mappedToKey isKindOfClass:[NSString class]]) {
                propertyMeta->_next = mapper[mappedToKey] ? : nil;
                mapper[mappedToKey] = propertyMeta;
        }];
    }
}
```

2. `YYClassInfo`是对成员变量、方法、成员属性以及类这四类信息的封装。

   YYModel 使用遍历属性的方式来进行模型转换，所以其中 Property 信息具有比较重要的作用。通过源码学习下 YYClassPropertyInfo 生成的过程。通过 Runtime 获取到 Model 的属性列表后，进行遍历操作组成` YYClassPropertyInfo`对象数组保存在 `YYClassInfo`对象中。` YYClassPropertyInfo`的构造函数的伪代码如下：

```objective-c
// YYClassPropertyInfo 构造函数
- (instancetype)initWithProperty:(objc_property_t)property {
    // 获得属性的名称
    const char *name = property_getName(property);
    _name = [NSString stringWithUTF8String:name];
    objc_property_attribute_t *attrs = property_copyAttributeList(property, &attrCount);
    for (unsigned int i = 0; i < attrCount; i++) {
       // 设置属性的类型
       _type = YYEncodingGetType(attrs[i].value);
    }
    // 设置存取方法
    if (_name.length) {
        if (!_getter) {
            _getter = NSSelectorFromString(_name);
        }
        if (!_setter) {
            _setter = NSSelectorFromString([NSString stringWithFormat:@"set%@%@:", [_name substringToIndex:1].uppercaseString, [_name substringFromIndex:1]]);
        }
    }
}
```

3. _YYModelPropertyMeta 是对 YYClassPropertyInfo 的进一步封装，保存了 YYClassPropertyInfo 的一些信息，例如 name、getter&setter。此外，存储了 JSON 到 Model 的映射关系。
4. 在获取到 `_YYModelMeta`对象后，遍历属性数组（allPropertyMetas），通过 YYClassPropertyInfo 中的 _mappedToKey 将 JSON 中对应的 value 赋值给属性。

```objective-c
 // 对 dic 中的所有健值执行 ModelSetWithDictionaryFunction 方法
CFDictionaryApplyFunction((CFDictionaryRef) dic, ModelSetWithDictionaryFunction, &context);
 // ModelSetWithDictionaryFunction 将 JSON 的健值赋值给对应的属性
static void ModelSetWithDictionaryFunction(const void *_key, const void *_value, void *_context) {
    ModelSetContext *context = _context;
    __unsafe_unretained _YYModelMeta *meta = (__bridge _YYModelMeta *) (context->modelMeta);
    __unsafe_unretained _YYModelPropertyMeta *propertyMeta = [meta->_mapper objectForKey:(__bridge id) (_key)];;
    __unsafe_unretained id model = (__bridge id) (context->model);
    while (propertyMeta) {
        if (propertyMeta->_setter) {
            ModelSetValueForProperty(model, (__bridge __unsafe_unretained id) _value, propertyMeta);
        }
        propertyMeta = propertyMeta->_next;
    };
}
// 以下为属性赋值的代码片段，为属性为字符串的字段赋值
static void ModelSetValueForProperty(__unsafe_unretained id model,
        __unsafe_unretained id value,
        __unsafe_unretained _YYModelPropertyMeta *meta) {
            switch (meta->_nsType) {
                case YYEncodingTypeNSString:
                case YYEncodingTypeNSMutableString: {
                    if ([value isKindOfClass:[NSString class]]) {
                        if (meta->_nsType == YYEncodingTypeNSString) {
                            ((void (*)(id, SEL, id)) (void *) objc_msgSend)((id) model, meta->_setter, value);
                        } else {
                            ((void (*)(id, SEL, id)) (void *) objc_msgSend)((id) model, meta->_setter, ((NSString *) value).mutableCopy);
                        }
                    }
                }
            }
}
```

**性能比较**

ibireme 在《iOS JSON 模型转换库评测》中对主流转换库的性能做了比较（[性能比较图](https://blog.ibireme.com/wp-content/uploads/2015/10/yymodel_benchmark_1.png)）。其中 Mantle 性能最差。由于项目中一直使用 Mantle 作为转换库，所以对于两者性能的差异的主要原因做一个分析与比较。Mantle 解析流程与 YYModel 相似，都是想通过 Runtime 获取到 Model 的属性列表，然后根据映射关系将 JSON 的值赋值给属性。对比两者解析的流程，一个很大的差异在于为属性赋值。Mantle 采用了 KVC（Key-Value Coding）赋值，而 YYModel 直接调用了 setter&getter 方法。做了简单的测试，对一个对象，分别复制 10000次，比较两者的耗时如下（单位纳秒）。

![屏幕快照 2018-01-09 22.59.49](http://ww4.sinaimg.cn/large/0060lm7Tly1fnar8jp7yaj30b4074mxi.jpg)

##### 应用项目

由于前面项目已经跟 Mantle 深度结合，在实践过程也遇到了一些的问题，下面记录下在这中间踩过的坑。

1. Model 与 FMDB 的转换

项目中通过 MTLFMDBAdapter 这个开源适配进行 Model 转换的，最初的想到的方案是仿写适配器将 YYModel 应用到项目中。先通过伪代码介绍一下 MTLFMDBAdapter 的工作流程。

```
- (id)initWithFMResultSet:(FMResultSet *)resultSet modelClass:(Class)modelClass error:(NSError **)error {
    // 获得属性列表
   NSSet *propertyKeys = [self.modelClass propertyKeys];
   NSArray *Keys = [[propertyKeys allObjects] sortedArrayUsingSelector:@selector(compare:)];
   //  将 FMDB 查询的结果根据 FMDBColumnForPropertyKey 映射关系生成新的 dictionary
   for (NSString *propertyKey in Keys) {
      NSString *columnName = [self FMDBColumnForPropertyKey:propertyKey];
      if (columnName == nil) continue;
        
      id value = [resultSet dataForColumn:columnName];;
      dictionaryValue[propertyKey] = value;  
   }
   // 调用 Mantle 生成新的 Model
   _model = [self.modelClass modelWithDictionary:dictionaryValue error:error];
   return self;
}
```

这部分运算主要包括两个部分。根据 DB 映射关系生成新的 Dictionary，然后将 Dictionary 转换成 Model。在这里做一些性能统计，在调用 Mantle 生成新的 Model 代码后面增加利用 YYModel 生成 Model 的方法。在一次 App 的运行过程中（631次查询）分别统计了这三部分运算的耗时，如图所示。

![](https://s1.ax1x.com/2018/01/10/pQDLXn.png)

其中可以发现，这个过程当中 YYModel 解析的速度大概是 Mantle 的两倍，主要的耗时都是在生成映射关系上。生成新的 Dictionary 的耗时还是让人震惊的，因此想对于适配器再进行优化，从而更好的提高应用的速度。由于 YYModel 已经预设了 JSON 到 Model 的解析，所以如果准备改造下 YYModel 的源码将 DB 的映射关系也保存到 YYModel 中，这样可以不用再生成新的 Dictionary。在项目中应用改造后的 YYModel 与 Mantle 进行性能比较，在一次 App 的运行过程中（631次查询） 如图所示。由此可见改造后的能够有效的降低数据层查询时 Model 转换的时间。

![](https://s1.ax2x.com/2018/01/10/jHQRa.png)

2. Model 拷贝导致的问题

但是在与 FMDB 结合使用的时候，FMDB 就出现了 assert 错误中断了程序的运行（alert 信息：inDatabase: was called reentrantly on the same queue, which would lead to a deadlock）。这个中断是在 queue 里面的 block执行过程中，又调用了 indatabase 方法，就会检查是不是同一个 queue，如果是同一个 queue 会死锁。查看了崩溃的堆栈发现，是由于数据库缓存的时候执行拷贝返回新的对象导致的问题。对比下两者的拷贝方法，就能够找到这个问题的原因。

（1）先看一下 MTLModel 中对于拷贝的处理。在基类 MTLModel 中实现了NSCopying 协议，会利用 Model 的 dictionaryValue 构造出一个全新的对象，如果属性为 MTLModel 的子类则会同样进行拷贝。从而达到深拷贝的作用。

```objective-c
- (instancetype)copyWithZone:(NSZone *)zone {
   return [[self.class allocWithZone:zone] initWithDictionary:self.dictionaryValue error:NULL];
}
// MTLModel 构造函数
- (instancetype)initWithDictionary:(NSDictionary *)dictionary error:(NSError **)error {
    NSSet *propertyKeys = self.class.propertyKeys;
	for (NSString *key in dictionary) {
        // 为属性赋值
		__autoreleasing id value = [dictionary objectForKey:key];
        BOOL success = MTLValidateAndSetValue(self, key, value, YES, error);
	}
	return self;
}
static BOOL MTLValidateAndSetValue(id obj, NSString *key, id value, BOOL forceUpdate, NSError **error) {
            // 如果是 MTLModel 的子类，支持深拷贝
            if ([keyClass isSubclassOfClass:MTLModel.class]) {
              deepCopy = YES;
            }
            if (deepCopy) {
                id convertValue = [MTLJSONAdapter modelOfClass:keyClass fromJSONDictionary:value error:error];
                [obj setValue:convertValue forKey:key];
            } else if(hasProperty) {
                [obj setValue:validatedValue forKey:key];
            }
		}
		return YES;
	}
}

```

（2）YYModel 是提供了一个 Category 的方法 [self.class yy_modelCopy]。该方法同 YYModel 转换的过程，遍历属性，通过 getter 方法获取到属性的值，然后通过 setter 方法赋值给新的对象。

```objective-c
- (id)yy_modelCopy
{
  	// 创建新的对象
    NSObject *one = [self.class new];
    for (_YYModelPropertyMeta *propertyMeta in modelMeta->_allPropertyMetas) {
     	// 示例，属性类型
            switch (propertyMeta->_type & YYEncodingTypeMask) {
                case YYEncodingTypeObject:
                case YYEncodingTypeClass:
                    id value = ((id (*)(id, SEL)) (void *) objc_msgSend)((id) self, propertyMeta->_getter);
                    ((void (*)(id, SEL, id)) (void *) objc_msgSend)((id) one, propertyMeta->_setter, value);
            }     
        }
    }
    return one;
}

```

由于项目中有些 Molde 的属性会进行数据库的操作，例如下面代码所示。

```
- (XXTest *)test
{
    if (_test) {
        return _test;
    }
    if (_testId > 0) {
        _test = (XXTest *)[XXService getModelWithId:_testId];
        return _test;
    }
    return nil;
}
```

在 Mantle 中，拷贝是将当前对象转换成字典后生成新的对象。而 YYModel 则会调用 getter 方法，因此存在一次数据库操作中发起了一个新的数据库操作，从而引起了FMDB 就出现了 assert 错误中断了程序的运行。对于这个问题的解决办法，最好的方式是进行 Model 的简化，从中拆分出 VO（Value Object） 与 BO（Business Object）。将业务抽离到新的 Model 中可以使代码结构更加清晰，并且能够直接的解决前面 copy 导致的问题。

3. 指定数组类型转换的对象

model 中属性类型为数组类型的转换时，例如属性声明为 `@property NSArray *shadows;`，这时转换过程 YYModel 与  Mantle 都需要指定对应的转换类型。这部分需要一一检查每个 model，更换为 yymodel 的实现方式。如果有遗漏将会导致崩溃的出现。

##### YYModel 实践的总结

在本次实践的过程当中深入的学习了 iOS Runtime 的相关知识。对于 Model 转换的实现有了更加深入的了解。但是在这个过程中也出现了很多问题，例如遗漏为数组转换实现函数，导致项目在 QA 阶段产出了一些崩溃与表现异常。在这种大换血的工作当中，唯有细心谨慎一一检查才能避免事故的产生。