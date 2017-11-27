---
layout: post
title: 聊聊 UIAppearance
date: 2017-10-17
<!--tags: UIAppearance,UIAppearanceContainer-->
---

## UIAppearance 的应用
1. 设置某类的全部实例外观

	```
	@implementation ViewController
	
	- (void)viewDidLoad {
		[super viewDidLoad];
		
		//注册外观信息到外观代理中
		[UISwitch appearance].tintColor = [UIColor redColor];//运行时特性
		
		UISwitch *switch1 = [[UISwitch alloc] initWithFrame:CGRectMake(100, 100, 50, 30)];
		UISwitch *switch2 = [[UISwitch alloc] initWithFrame:CGRectMake(100, 200, 50, 30)];
		[self.view addSubview:switch1];
		[self.view addSubview:switch2];
	}
	
	@end
	```

	属性tintColor经过UISwitch类的**appearance proxy**管理后，所有新创建的UISwitch实例都被设置成了红色。
	
	**属性tintcolor：类似这种添加到外观代理中的信息统称为外观信息**

2. 设置部分容器内的类的实例外观

	```
	#import "ViewController.h"
	#import "MYContainerView.h"
	
	@interface ViewController ()
	@property (nonatomic, strong)MYContainerView *pContainerView;
	@end
	
	@implementation ViewController
	
	- (void)viewDidLoad {
		[super viewDidLoad];
		
		//注册外观信息到外观代理中
		[UISwitch appearance].tintColor = [UIColor redColor];//运行时特性
		// 同时注册部分类的外观信息
		[UISwitch appearanceWhenContainedInInstancesOfClasses:@[[MYContainerView class]]].tintColor = [UIColor blueColor];
		
		UISwitch *switch1 = [[UISwitch alloc] initWithFrame:CGRectMake(100, 100, 50, 30)];
		UISwitch *switch2 = [[UISwitch alloc] initWithFrame:CGRectMake(100, 200, 50, 30)];
		
		_pContainerView = [[MYContainerView alloc] initWithFrame:CGRectMake(200, 100, 100, 200)];
		
		[self.view addSubview:switch1];
		[self.view addSubview:switch2];
		[self.view addSubview:_pContainerView];
	}
	
	@end
	
	```
	
	容器类：
	
	```
	#import "MYContainerView.h"
	@implementation MYContainerView
	
	- (instancetype)initWithFrame:(CGRect)frame {
		self = [super initWithFrame:frame];
		if (self) {
			self.backgroundColor = [UIColor purpleColor];
        	UISwitch *switch3 = [[UISwitch alloc] initWithFrame:CGRectMake(10, 0, 50, 30)];
        	
        	[self addSubview:switch3];
        }
       return self;
    }
    
    @end
    
	```
	图示：
	![容器外观](https://github.com/MayerFan/AllDemos/blob/master/Demos/UIAppearance_Demo/Images/appearance_01.png)
	
3. 是不是所有的属性和方法都能被**appearance proxy**管理？

	答案是否定的。对于objective-c而言，只有带有**UI_APPEARANCE_SELECOR**宏的属性才能被外观代理所管理。
	
	对于swift而言，无论是计算属性还是方法(存储属性不可以)，只要添加dynamic修饰，就可以被外观代理所管理。

## 自定义外观协议
通过自定义协议的实现大致了解系统UIAppearance协议的实现原理。

外观协议MYAppearance.h:

```
@protocol MYAppearance <NSObject>
+ (instancetype)appearance;
@end

@interface MYAppearance : NSObject

+ (id)appearanceForClass:(Class)aClass;
/**
 Applies the appearance to the object.
 implementation: send message retained in appearance proxy to target
 */
- (void)applyInvocationTo:(id)target;

@end
```

MYAppearance.m:

```
#import "MYAppearance.h"

/*
    1. 外观代理MYAppearance实例中暂存着对应类实例需要设置的外观信息
    2. 某个时间段类实例invoke外观代理中暂存的外观信息来更新外观
 */

static NSMutableDictionary *instanceOfClassesDictionary = nil;

@interface MYAppearance ()
@property (strong, nonatomic) Class mainClass;
@property (strong, nonatomic) NSMutableArray *invocations;
@end

@implementation MYAppearance

- (id)initWithClass:(Class)thisClass
{
    _mainClass = thisClass;
    _invocations = [NSMutableArray array];
    
    return self;
}
+ (id)appearanceForClass:(Class)aClass
{
    static dispatch_once_t onceToken;
    
    dispatch_once(&onceToken, ^{
        instanceOfClassesDictionary = [[NSMutableDictionary alloc] init];
    });
    
    if (![instanceOfClassesDictionary objectForKey:NSStringFromClass(aClass)])
    {
        id appearance = [[self alloc] initWithClass:aClass];
        [instanceOfClassesDictionary setObject:appearance forKey:NSStringFromClass(aClass)];
        return appearance;
    }
    else {
        return [instanceOfClassesDictionary objectForKey:NSStringFromClass(aClass)];
    }
}

// MYAppearance实例收到任何消息后的触发
// 默认是 [invocation setTarget:target];
// [invocation invoke]; 直接执行方法。但是当前重写后保留，给真正的调用者MYCustomObj(对当前工程而言)
- (void)forwardInvocation:(NSInvocation *)anInvocation;
{
    [anInvocation setTarget:nil];
    [anInvocation retainArguments];
    
    [self.invocations addObject:anInvocation];
}
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector
{
    return [self.mainClass instanceMethodSignatureForSelector:aSelector];
}

- (void)applyInvocationTo:(id)target
{
    for (NSInvocation *invocation in self.invocations) {
        //target执行当前invocation中的所有消息
        [invocation setTarget:target];
        [invocation invoke];
    }
}

@end

```

定义一个MYCustomObj类遵守**MYAppearance**协议:

```
#import <Foundation/Foundation.h>
#import <CoreGraphics/CoreGraphics.h>
#import "MYAppearance.h"

@interface MYCustomObj : NSObject<MYAppearance>

@property (nonatomic,assign)CGFloat customFloat;

@end
```
MYCustomObj.m:

```
#import "MYCustomObj.h"

@implementation MYCustomObj

+ (instancetype)appearance {
    return [MYAppearance appearanceForClass:[self class]];
}

@end
```

测试MYCustomObj实例的外观信息:

```
#import "ViewController.h"
#import "MYCustomObj.h"

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // 注册外观信息
    [MYCustomObj appearance].customFloat = 1000;
    
    MYCustomObj *test1 = [MYCustomObj new];
    MYCustomObj *test2 = [MYCustomObj new];
    
    // 执行外观信息
    //    [[MYCustomObj appearance] applyInvocationTo:test1];// 提示MYCustomObj中没有声明此方法
    id appearance = [MYCustomObj appearance];//利用运行时特性
    [appearance applyInvocationTo: test1];
    [appearance applyInvocationTo: test2];
    
    NSLog(@"test1 is %f\n", test1.customFloat);
    NSLog(@"test2 is %f\n", test2.customFloat);
    
    // 系统的这个执行过程发生在某个时间段
}
```

控制台信息:

```
2017-11-27 15:50:58.343683+0800 UIAppearance[36143:42437627] test1 is 1000.000000
2017-11-27 15:50:58.343920+0800 UIAppearance[36143:42437627] test2 is 1000.000000
```

**系统UIAppearance的 applyInvocationTo（也就是invoke）行为发生在视图将要显示之后和视图将要布局之前的时间段**

## 原理分析
到底UIAppearance背后的实现原理是什么？为什么会这么神奇？

因为没有源码，无法直接看到系统内部的实现逻辑。通过应用UIAppearance和自已实现自定义外观协议，以及debuger和日志测试，大致有个逻辑了解。

1. **UIAppearance**什么阶段invoke外观信息？（也就是说什么时候类实例会去调用存储在**appearance proxy**中的属性）
	
	验证代码：
	
	```
	#import "ViewController.h"
	
	@interface ViewController ()
	@property (nonatomic, strong)UISwitch *pTempSwitch;
	@end

	@implementation ViewController
	
	- (void)viewDidLoad {
    	[super viewDidLoad];
    
    	UISwitch * appear = [UISwitch appearance];
   	appear.tintColor = [UIColor redColor];
    
    	_pTempSwitch = [[UISwitch alloc] initWithFrame:CGRectMake(100, 100, 50, 30)];
    	UISwitch *switch2 = [[UISwitch alloc] initWithFrame:CGRectMake(100, 200, 50, 30)];
    
    	NSLog(@"switch's 当前 tintColor is %@", _pTempSwitch.tintColor);
    
    	[self.view addSubview:_pTempSwitch];
    	[self.view addSubview:switch2];
    
    	NSLog(@"加到视图上之后 tintColor is %@",_pTempSwitch.tintColor);
	}
	- (void)viewWillAppear:(BOOL)animated {
    	[super viewWillAppear:animated];
   	NSLog(@"视图将要显示后 tintColor is %@", _pTempSwitch.tintColor);
}
	- (void)viewDidAppear:(BOOL)animated {
    	[super viewDidAppear:animated];
   	NSLog(@"视图完全显示后 tintColor is %@", _pTempSwitch.tintColor);
}
	- (void)viewWillLayoutSubviews {
    	[super viewWillLayoutSubviews];
    	NSLog(@"视图将要布局 tintColor is %@", _pTempSwitch.tintColor);
}
	```
	控制台输出信息：
	
	```
2017-11-27 13:53:13.662091+0800 UIAppearance[35076:42179727] switch's 当前 tintColor is (null)
2017-11-27 13:53:13.668349+0800 UIAppearance[35076:42179727] 加到视图上之后 tintColor is (null)
2017-11-27 13:53:13.668742+0800 UIAppearance[35076:42179727] 视图将要显示后 tintColor is (null)
2017-11-27 13:53:14.170869+0800 UIAppearance[35076:42179727] 视图将要布局 tintColor is UIExtendedSRGBColorSpace 1 0 0 1
2017-11-27 13:53:14.171519+0800 UIAppearance[35076:42179727] 视图将要布局 tintColor is UIExtendedSRGBColorSpace 1 0 0 1
2017-11-27 13:53:14.209010+0800 UIAppearance[35076:42179727] 视图完全显示后 tintColor is UIExtendedSRGBColorSpace 1 0 0 1
	```
	**可知：invoke外观信息的行为发生在视图将要显示之后和视图将要布局之前的时间段**。因此只要在viewWillLayoutSubviews之前注册外观信息，外观信息就可以被invoke

2. <span id="q&a_3">在每个类实例的生命周期中只有一次机会invoke外观信息。当改变外观信息后，类实例的外观只能在另一个生命周期中更新

	验证代码:
	
	```
	@implementation ViewController
	
	- (void)viewDidLoad {
		[super viewDidLoad];
		
		//注册外观信息到外观代理中
		[UISwitch appearance].tintColor = [UIColor redColor];//运行时特性
		
		UISwitch *switch1 = [[UISwitch alloc] initWithFrame:CGRectMake(100, 100, 50, 30)];
		UISwitch *switch2 = [[UISwitch alloc] initWithFrame:CGRectMake(100, 200, 50, 30)];
		[self.view addSubview:switch1];
		[self.view addSubview:switch2];
		
		UIButton *btn = [UIButton new];
		btn.frame = CGRectMake(100, 50, 150, 30);
		btn.backgroundColor = [UIColor purpleColor];
		[btn setTitle:@"更新外观信息" forState:UIControlStateNormal];
		[btn addTarget:self action:@selector(btnOnClick) forControlEvents:UIControlEventTouchUpInside];
	}
	- (void)btnOnClick {
		[UISwitch appearance].tintColor = [UIColor blueColor]; //点击后完全无效果
	}
	
	@end
	
	```
	图示效果无变化：
	![效果无变化](https://github.com/MayerFan/AllDemos/blob/master/Demos/UIAppearance_Demo/Images/appearance_02.png)
	
	动态添加一个新的switch实例:
	
	```
	- (void)btnOnClick {
		[UISwitch appearance].tintColor = [UIColor blueColor]; 
		UISwitch *switch3 = [[UISwitch alloc] initWithFrame:CGRectMake(100, 300, 50, 30)];
		[self.view addSubview:switch3];// 新添加的switch变化
	}
	```
	图示效果：
	![外观变化](https://github.com/MayerFan/AllDemos/blob/master/Demos/UIAppearance_Demo/Images/appearance_03.png)
	
	**可见：切换主题后，要想立即更新当前页面外观，需要重新开启新的生命周期（可以remvoeView后再addSubviews）**

## Q&A
1.	Q: 既然是处理视图外观，为什么要定义成一个协议，而不是在UIView中实现呢？

	A: 因为像UIBarItem and UIBarButtonItem 这样的不是继承与UIveiw的对象他们也要处理自己身上的视图外观。
2.	Q: UIAppearance有什么用？

	A: 1. 统一视图外观代码简洁；2. 主题切换
3. 	Q: 为什么类实例的外观信息在生命周期中只可以更新一次？

	A: [点击此处](#q&a_3)
4. 	Q: 为什么swift中需要添加**dynamic**才能被*appearance proxy*管理呢？

	A: 因为dynamic告诉编译器，被dynamic声明的方法和属性需要到运行时的时候才能确定，编译期请编译器大哥通融通融。
5. 	Q: 所有类的外观代理是否是同一个？

	A: 不是，每个类会生成一个自己的外观代理 
 


[Demo地址](https://github.com/MayerFan/AllDemos)
---
参考资料：

[官方文档UIAppearance](https://developer.apple.com/documentation/uikit/uiappearance)

[官方文档UIAppearancecontainer](https://developer.apple.com/documentation/uikit/uiappearancecontainer)

[MZAppearance源码](https://github.com/m1entus/MZAppearance)

[自定义系统控件的外观：UIApearance](http://www.cocoachina.com/ios/20150723/12671.html)

[UIAppearance](http://nshipster.com/uiappearance/)

[**UIAppearance for Custom Views**](http://petersteinberger.com/blog/2013/uiappearance-for-custom-views/)

[stackoverflow](https://stackoverflow.com/questions/15732885/uiappearance-proxy-for-custom-objects)




