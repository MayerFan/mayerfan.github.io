---
layout: post
title: WeX5(cordova)调用苹果内购
date: 2017-11-26
---

说是WeX5调用苹果内购，是不严谨的。确切的说是WeX5利用cordova的内购插件(cordova-plugin-inapppurchase)和内购通讯，而插件本质上是利用的苹果原生api（StoreKit）访问内购。

WeX5调用苹果内购原理图示：
![01](/images/2017-12-26/01.png)

这个思路就是苹果官方指南中描述：App Store内购买流程必须经由Xcode中的**StoreKit framework**访问(服务器端二次验证收据属于内购买结束验证)。

苹果官方图示：
![02](/images/2017-12-26/02.png)

## WeX5开发
WeX5的具体开发请参考官方教程

## iTunesconnect给app配置内购信息
app内具体配置内购信息流程网上教程遍布。具体请Google。

## WeX5打包教程
主要介绍WeX5打包教程。期间会遇见过多问题

[WeX5打包教程文档参考](http://docs.wex5.com/wex5-platform-to-app-process/)

**打包平台： Mac OS + Xcode9.1 环境 本地打包**

1. 下载mac环境 WeX5平台版本


2. 获取.p12证书和.mobileprovision的App描述文件. 此证书和描述文件关联已配置支持内购信息的应用。

3. 下载内购插件 [cordova-plugin-inapppurchase](https://github.com/AlexDisler/cordova-plugin-inapppurchase)

4. 添加cordova内购插件到WeX5中

	添加至目录：/model/Native/plugins/
	
5. 添加WeX5 UI资源demo 

	5.1 添加demo至目录：model/UI2/
	
	5.2 [一个简单的demo](#resourse)

6. 打开WeX5版本，双击“启动WeX5开发工具.bat”打开studio开发工具

7. 在模型资源下找到Native目录，右键菜单可新建–创建本地APP。

	![网图](http://doc.wex5.com/wp-content/uploads/2015/03/212.jpg)

8. 选择应用模式。默认或一般选择为模式1. 多种模式区别请参考[WeX5模式选择](http://docs.wex5.com/choose-app-packing-mode/)
	
	![应用模式](http://doc.wex5.com/wp-content/uploads/2015/03/43.jpg)
	
9. 打开“设置服务地址和选择UI资源”界面. 如果不从网络加载资源（即本地资源）ajax服务地址和web地址可以不填。此时ajax服务地址默认为http://localhost.

	![服务地址](/images/2017-12-26/03.png)
	
10. 选择UI资源。比如我的当前资源为myDemo。双击list.w指定首页路径。勾选ui资源

	![UI资源](/images/2017-12-26/04.png)

11. 配置应用信息. 

	版本号每次打正式发布包时需写新的序号，一个正式APP包对应一个版本号，以便用户在移动终端上安装时能检查到已安装应用进行更新。

	应用包名输入苹果APP证书生成时对应的Bundle ID

	![应用信息](http://doc.wex5.com/wp-content/uploads/2015/03/73.jpg)

12. 配置开发者信息和证书。

	打iOS的APP包需要根据使用的是iOS的开发证书还是发布证书进行选择。输入iOS证书密码（是P12文件的密码），然后选择对应的P12文件和APP验证文件，是开发证书则选择ios.developer.mobileprovision和ios.developer.p12，是发布证书则选择ios.distribution.mobileprovision和ios.distribution.p12。（证书文件名称没有要求，平台会自动将文件名称修改为标准的并拷贝至生成APP的文件夹下）

	![证书](http://doc.wex5.com/wp-content/uploads/2015/03/82.jpg)
	
13. 设置屏幕选项。 如果不填。默认为WeX5的图

	![屏幕](http://doc.wex5.com/wp-content/uploads/2015/03/92.jpg)
	
14. 选择打包的本地插件. 默认为自动选择. 搜索inapppurchase插件，手动添加

	![插件](/images/2017-12-26/05.png)
	
15. 打包本地应用。

	本地应用包含UI资源：**模式1该项必选**；模式2和模式3建议选择该项，可以第一次打开时不下载资源提升速度.
	
	重新编译使用到的UI资源：建议默认都重新编译。如未选择打包资源，则该选项默认为不选且是灰色的，不会重新编译资源。更换打包模式时，必须重新编译。
	
	发布模式：使用iOS的发布证书（distribution）打包时必须选择该模式。使用iOS的开发证书（developer）打包时，该项必须**不选择**。

	![打包](/images/2017-12-26/06.png)
	
16. **我的WeX5版本支持Xcode7版本打包**。对于最新的Xcode9.1环境。Cordova利用Xcode9编译签名的时候报签名错误

	```
	Code Signing Error: Signing for "test" requires a development team. Select a development team in the project editor.
    Code Signing Error: Code signing is required for product type 'Application' in SDK 'iOS 11.1'
	```
17. 换一种打包思路。

	既然本质上利用Xcode命令编译完成（会生成Xcode目录），只是签名失败。因此可以利用Xcode工具打开编译完成的目录。
	
	进入目录：/model/Native/test/build/src/platforms/ios/
	
	会看到编译生成的Xcode工程
	
	![xcode](/images/2017-12-26/07.png)
	
18. 打开工程。利用Xcode配置完证书打包。这种方式是不是so easy

19. 手机安装ipa。

	![1](/images/2017-12-26/08.png)
	
	![2](/images/2017-12-26/09.png)
	
	![3](/images/2017-12-26/10.png)
	

## js调用源码
1. 查询列表

	```
	var product_ids = ['id1', 'id2', 'id3']
	inAppPurchase
	      .getProducts(product_ids)
	      .then(function (products) {
	    	for(var i =0;i<products.length;i++){
	    	  that.products.push(products[i]);
	        };
	      })
	      .catch(function (err) {
	    	  console.log(err);
	      });
	```
2. 购买

	```
	inAppPurchase
		  .buy(productid)
		  .then(function (data) {
		    return inAppPurchase.consume(data.type, data.receipt, data.signature);
		  })
		  .then(function () {
		    console.log('consume done!');
		  })
		  .catch(function (err) {
		    console.log(err);
		  });
	```
3. 重置

	```
	inAppPurchase
	    .restorePurchases()
	    .then(function (purchases) {
	      alert('重置成功')
	      console.log(JSON.stringify(purchases));
	  })
	  .catch(function (err) {
	    console.log(err);
	  });
	```




<span id="resourse">
[myDemo资源](https://github.com/MayerFan/callAppleInAppPurchaseWithWeX5){:target="_blank"}

---
参考资料：

[cordova-plugin-inapppurchase](https://github.com/AlexDisler/cordova-plugin-inapppurchase)

[WeX5文档](http://docs.wex5.com/)

[苹果内购流程](http://blog.csdn.net/yupu56/article/details/46907609)

[苹果官方内购指南](https://developer.apple.com/library/content/documentation/NetworkingInternet/Conceptual/StoreKitGuide/Introduction.html#//apple_ref/doc/uid/TP40008267-CH1-SW1)
