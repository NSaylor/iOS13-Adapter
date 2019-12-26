# iOS13-Adapter
关于iOS13适配的相关问题

#### 内容声明
 `赤裸裸的使用本文内容去做所谓原创的，麻烦请注明出处。`

#### Xcode11 缺失库文件导入位置变更 
[libstdc-6.0.9 文件下载](https://github.com/MonkeyHZT/libstdc-6.0.9)

Xcode11下 这个目录不存在了
```
/Applications/Xcode-beta.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/Library/CoreSimulator/Profiles/Runtimes/iOS.simruntime/Contents/Resources/RuntimeRoot/usr/lib/
【变更为】
/Applications/Xcode-beta.app/Contents/Developer/Platforms/iPhoneOS.platform/Library/Developer/CoreSimulator/Profiles/Runtimes/iOS.simruntime/Contents/Resources/RuntimeRoot/usr/lib/
```
----以下位置不需要改变
```
/Applications/Xcode-beta.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk/usr/lib/

/Applications/Xcode-beta.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS.sdk/usr/lib/

/Applications/Xcode-beta.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/
```

#### ~~友盟相关~~ `升级最新版本SDK`
##### ~~注册新浪平台 崩溃【验证：仅在模拟器上出现】~~
这个应该是需要微博官方进行适配了，尝试模拟了 `getUniqueStrByUUID` 中的相关写法。
![屏幕快照2019-06-05下午1.47.08.png](https://upload-images.jianshu.io/upload_images/2120742-600e04ba19dee57d?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
* 暂时的解决方案：
```
//修复iOS13下 崩溃问题 验证为：模拟器下出现
#if TARGET_IPHONE_SIMULATOR
/// 交换方法实现
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        if(@available(iOS 13.0, *)){
            Method origin = class_getClassMethod([UIDevice class], NSSelectorFromString(@"getUniqueStrByUUID"));
            //    IMP originImp = method_getImplementation(origin);
            
            Method swizz = class_getClassMethod([self class], @selector(swizz_getUniqueStrByUUID));
            //交换方法实现
            method_exchangeImplementations(origin, swizz);
        }
    });

#pragma mark - 获取唯一标识 新浪
+ (NSString *)swizz_getUniqueStrByUUID{
    CFUUIDRef  uuidObj = CFUUIDCreate(nil);//create a new UUID
    //get the string representation of the UUID
    NSString    *uuidString = (__bridge_transfer NSString *)CFUUIDCreateString(nil, uuidObj);
    CFRelease(uuidObj);
    return uuidString ;
}
#endif
```

##### ~~+[_LSDefaults sharedInstance] 崩溃问题~~
> 针对的友盟版本：
移动统计：6.0.5
消息推送：3.2.4
社会化分享：6.9.6

* 暂时的解决方案：
```
@implementation NSObject (Extend)
+ (void)load{
    
    SEL originalSelector = @selector(doesNotRecognizeSelector:);
    SEL swizzledSelector = @selector(sw_doesNotRecognizeSelector:);
    
    Method originalMethod = class_getClassMethod(self, originalSelector);
    Method swizzledMethod = class_getClassMethod(self, swizzledSelector);
    
    if(class_addMethod(self, originalSelector, method_getImplementation(swizzledMethod), method_getTypeEncoding(swizzledMethod))){
        class_replaceMethod(self, swizzledSelector, method_getImplementation(originalMethod), method_getTypeEncoding(originalMethod));
    }else{
        method_exchangeImplementations(originalMethod, swizzledMethod);
    }
}

+ (void)sw_doesNotRecognizeSelector:(SEL)aSelector{
    //处理 _LSDefaults 崩溃问题
    if([[self description] isEqualToString:@"_LSDefaults"] && (aSelector == @selector(sharedInstance))){
        //冷处理...
        return;
    }
    [self sw_doesNotRecognizeSelector:aSelector];
}

```
#### DeviceToken获取 [友盟公告](https://info.umeng.com/detail?id=174&&cateId=1)

```
#include <arpa/inet.h>

- (void)application:(UIApplication *)application didRegisterForRemoteNotificationsWithDeviceToken:(NSData *)deviceToken
{
    if (![deviceToken isKindOfClass:[NSData class]]) return;
    const unsigned *tokenBytes = (const unsigned *)[deviceToken bytes];
    NSString *hexToken = [NSString stringWithFormat:@"%08x%08x%08x%08x%08x%08x%08x%08x",
                          ntohl(tokenBytes[0]), ntohl(tokenBytes[1]), ntohl(tokenBytes[2]),
                          ntohl(tokenBytes[3]), ntohl(tokenBytes[4]), ntohl(tokenBytes[5]),
                          ntohl(tokenBytes[6]), ntohl(tokenBytes[7])];
    NSLog(@"deviceToken:%@",hexToken);
}
```

#### UITextField 
##### 通过KVC方式修改空白提示语颜色 崩溃
```
[UITextField setValue:[UIColor redColor] forKeyPath:@"_placeholderLabel.textColor”];
```
* 解决方案：
```
attributedPlaceholder
```

##### leftView、rightView 设置异常 【疑似iOS13beta4新出现】
> 在设置`leftView` 左按钮时，如果使用的是`UIImageView`即会出现图片无法按照意图显示的问题。

>[@騲尼罵人獣狂](https://www.jianshu.com/u/19da237c8c77) 反馈`UITextField`的`rightView`的子视图如果使用`约束布局`， 会导致`rightView`覆盖整个`UITextField`。 

```
// Confuse in beta4 iOS13
UIImageView *imageIcon = [[UIImageView alloc]initWithFrame:CGRectMake(0, 0, 34, 30)];
//search_icon  15*15
imageIcon.image = [UIImage imageNamed:@"search_icon"];
imageIcon.contentMode = UIViewContentModeCenter;
UITextField *txtSearch = [[UITextField alloc] init];
txtSearch.leftView = imageIcon;
```
* 解决方案：`UIImageVIew 包一层UIView再设置给leftView 、设置leftView或rightView不要使用约束布局` 

#### UISearchBar 
> 验证来源：[@騲尼罵人獣狂](https://www.jianshu.com/u/19da237c8c77)
`searchTextField`属性对外暴露了，不用再通过KVC获取了。
```
    UISearchBar *searchBar = [[UISearchBar alloc]init];
    searchBar.searchTextField
```
#### 控制器相关
##### 模态跳转样式变更
过场动画上下文机制有调整，默认调整为了卡片样式。
```
/*
 Defines the presentation style that will be used for this view controller when it is presented modally. Set this property on the view controller to be presented, not the presenter.
 If this property has been set to UIModalPresentationAutomatic, reading it will always return a concrete presentation style. By default UIViewController resolves UIModalPresentationAutomatic to UIModalPresentationPageSheet, but other system-provided view controllers may resolve UIModalPresentationAutomatic to other concrete presentation styles.
 Defaults to UIModalPresentationAutomatic on iOS starting in iOS 13.0, and UIModalPresentationFullScreen on previous versions. Defaults to UIModalPresentationFullScreen on all other platforms.
 */
@property(nonatomic,assign) UIModalPresentationStyle modalPresentationStyle API_AVAILABLE(ios(3.2));
[nav setModalPresentationStyle:UIModalPresentationFullScreen];
```
##### 模态横屏弹出 
> 如果希望模态视图是以横屏状态弹出，需要注意到其会受到跳转样式的影响。
`默认的卡片样式，仍然会以竖屏弹出。`

[参考链接](https://www.jianshu.com/p/cb364ffa30a0)
```
- (BOOL)shouldAutorotate {
    return NO;
}
- (UIInterfaceOrientationMask)supportedInterfaceOrientations {
       return UIInterfaceOrientationMaskLandscapeRight;
}
- (UIInterfaceOrientation)preferredInterfaceOrientationForPresentation {
        return UIInterfaceOrientationLandscapeRight;
}

```


#### Dark Mode 颜色主题相关

##### UIColor 适配
```
 UIColor *dynamicColor =  [UIColor colorWithDynamicProvider:^UIColor * _Nonnull(UITraitCollection * provider) {
 //使用 provider 判断，有时会出问题
    if(keyWindow.isDark){
        return darkColor;
    }
    return lightColor;
 }];
```
##### YYTextView && YYLabel 适配 `完美解决的前提是 UIColor 必须正确适配`  
[针对YY的改造--链接直达](https://www.jianshu.com/p/4bae9e942630)


> 如果你也在做该适配的话，那么很可能遇到以下的问题。
尝试了很多次，最大的问题竟出在 `UIColor`。
不过出现的问题，疑似和YY内部在Runloop 即将休眠时进行绘制任务具有很大的相关性。
具体原因还不能确定，等以后再深究一下。
###### 系统表现
>这里先看下系统UILabel的暗夜适配找寻一下灵感

![UILabel DarkMode.png](https://upload-images.jianshu.io/upload_images/2120742-1f417d7b9772ba3f?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可以看到，UILabel的绘制是调用 `drawTextInRext`，而翻看YY能看到其使用的    `CTRunDraw()`。由于一开始对UIDynamicProviderColor有误解，也尝试过解析其中打包的颜色，来通过查看CTRun的属性集来判断当前是否正确渲染。

....然而，在YYLabel应用上述方案时可能正常，但YYTeView却出现了其它的问题。
....排查中发现，某些时候`UITraitCollection.currentTraitCollection`解析出的颜色，和对应的状态不符。
....最终发现，`colorWithDynamicProvider`中回调的状态可能出现和当前系统状态不一致的情况，也就是说这个回调有点不那么可信了... 误我青春

###### YYLabel 适配

> YYLabel.m 添加如下代码

```
#pragma mark - DarkMode Adapater

#ifdef __IPHONE_13_0
- (void)traitCollectionDidChange:(UITraitCollection *)previousTraitCollection{
    [super traitCollectionDidChange:previousTraitCollection];
    
    if (@available(iOS 13.0, *)) {
        if([UITraitCollection.currentTraitCollection hasDifferentColorAppearanceComparedToTraitCollection:previousTraitCollection]){
            [self.layer setNeedsDisplay];
        }
    } else {
        // Fallback on earlier versions
    }
}
#endif
```
###### YYTextView 适配

> YYTextView.m 添加如下代码

```
#pragma mark - Dark mode Adapter

#ifdef __IPHONE_13_0
- (void)traitCollectionDidChange:(UITraitCollection *)previousTraitCollection{
    [super traitCollectionDidChange:previousTraitCollection];
    
    if (@available(iOS 13.0, *)) {
        if([UITraitCollection.currentTraitCollection hasDifferentColorAppearanceComparedToTraitCollection:previousTraitCollection]){
            [self _commitUpdate];
        }
    } else {
        // Fallback on earlier versions
    }
}
#endif
```
额外要做的事情
* NSAttributedString+YYText.m  去除部分CGColor 调用
```
- (void)setColor:(UIColor *)color {
    [self setColor:color range:NSMakeRange(0, self.length)];
}

- (void)setStrokeColor:(UIColor *)strokeColor {
    [self setStrokeColor:strokeColor range:NSMakeRange(0, self.length)];
}

- (void)setStrikethroughColor:(UIColor *)strikethroughColor {
    [self setStrikethroughColor:strikethroughColor range:NSMakeRange(0, self.length)];
}

- (void)setUnderlineColor:(UIColor *)underlineColor {
    [self setUnderlineColor:underlineColor range:NSMakeRange(0, self.length)];
}

```
#### UIImageView 
> 主要发现两个问题：
>1.初始化：UIImageView.image 初始化时必须设置，否则不显示 。
>2.暗黑适配：图片经过拉伸处理后，会导致暗黑适配失效。

* [@不羁觅觅羊](https://juejin.im/user/5b513d33e51d4519873f4b9c) 分享的解决方案：
```
#pragma mark - 解决Image拉伸问题
+ (UITraitCollection *)lightTrait API_AVAILABLE(ios(13.0)) {
    static UITraitCollection *trait = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        trait = [UITraitCollection traitCollectionWithTraitsFromCollections:@[
            [UITraitCollection traitCollectionWithDisplayScale:UIScreen.mainScreen.scale],
            [UITraitCollection traitCollectionWithUserInterfaceStyle:UIUserInterfaceStyleLight]
        ]];
    });

    return trait;
}

+ (UITraitCollection *)darkTrait API_AVAILABLE(ios(13.0)) {
    static UITraitCollection *trait = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        trait = [UITraitCollection traitCollectionWithTraitsFromCollections:@[
            [UITraitCollection traitCollectionWithDisplayScale:UIScreen.mainScreen.scale],
            [UITraitCollection traitCollectionWithUserInterfaceStyle:UIUserInterfaceStyleDark]
        ]];
    });

    return trait;
}

+ (void)fixResizableImage API_AVAILABLE(ios(13.0)) {
    
    Class klass = UIImage.class;
    SEL selector = @selector(resizableImageWithCapInsets:resizingMode:);
    Method method = class_getInstanceMethod(klass, selector);
    if (method == NULL) {
        return;
    }
    
    IMP originalImp = class_getMethodImplementation(klass, selector);
    if (!originalImp) {
        return;
    }
    
    IMP dynamicColorCompatibleImp = imp_implementationWithBlock(^UIImage *(UIImage *_self, UIEdgeInsets insets, UIImageResizingMode resizingMode) {
            // 理论上可以判断UIColor 是否是 UIDynamicCatalogColor.class, 如果不是, 直接返回原实现; 但没必要.
            UITraitCollection *lightTrait = [self lightTrait];
            UITraitCollection *darkTrait = [self darkTrait];

            UIImage *resizable = ((UIImage * (*)(UIImage *, SEL, UIEdgeInsets, UIImageResizingMode))
                                      originalImp)(_self, selector, insets, resizingMode);
            UIImage *resizableInLight = [_self.imageAsset imageWithTraitCollection:lightTrait];
            UIImage *resizableInDark = [_self.imageAsset imageWithTraitCollection:darkTrait];
        
            if (resizableInLight) {
                [resizable.imageAsset registerImage:((UIImage * (*)(UIImage *, SEL, UIEdgeInsets, UIImageResizingMode))
                                                         originalImp)(resizableInLight, selector, insets, resizingMode)
                                withTraitCollection:lightTrait];
            }
            if (resizableInDark) {
                [resizable.imageAsset registerImage:((UIImage * (*)(UIImage *, SEL, UIEdgeInsets, UIImageResizingMode))
                                                         originalImp)(resizableInDark, selector, insets, resizingMode)
                                withTraitCollection:darkTrait];
            }
            return resizable;
        });

    class_replaceMethod(klass, selector, dynamicColorCompatibleImp, method_getTypeEncoding(method));
}

```

#### 绘制异常 【暂未确定是否为iOS13新发】
##### UIImageView绘制
> 应用场景：`自定义控件生成图片`，绘制API会受到`UIImage`的拉伸方式`UIImageResizingModeTile`干扰。
> 影响：`所得并不是所见`

```
/// 常见的绘制代码
UIGraphicsBeginImageContextWithOptions(contentSize,YES,[[UIScreen mainScreen] scale]);

问题API：
/// CGContextRef ctx = UIGraphicsGetCurrentContext();
/// [self.viewContent.layer renderInContext:ctx];

建议使用如下API：
[self.viewContent drawViewHierarchyInRect:CGRectMake(0, 0, contentSize.width, contentSize.height) afterScreenUpdates:YES];

//生成图片
UIImage *image = UIGraphicsGetImageFromCurrentImageContext();
UIGraphicsEndImageContext();
```
##### 截长图
> 有朋友反馈 iOS13上截长图，只能截取到当前屏幕。
> 原因：约束和绝对布局混用导致，截图代码和布局代码的`UI设定环境`调整为一致就好。

* 随处可见的，截长图代码，原理就是设置`UIView.size`为实际内容`size`。
```
    UIImage* shotImage = nil;
    UITableView *scrollView = self.tableBase;
    //临时数据
    CGPoint fltOriginOffset = scrollView.contentOffset;
    CGRect rectOrigin = scrollView.frame;

    scrollView.contentOffset = CGPointZero;
    scrollView.frame = CGRectMake(0, 0, scrollView.contentSize.width, scrollView.contentSize.height);

    UIGraphicsBeginImageContextWithOptions(CGSizeMake(scrollView.contentSize.width, scrollView.contentSize.height), scrollView.opaque, 0.0);
    [scrollView.layer renderInContext:UIGraphicsGetCurrentContext()];
    shotImage = UIGraphicsGetImageFromCurrentImageContext();

    UIGraphicsEndImageContext();
    
    //状态复原
    scrollView.contentOffset = fltOriginOffset;
    scrollView.frame = rectOrigin;
```

#### UIWindow 变更  `iOS 13.3 版本中已由苹果修复`
>在iOS13版本下，App 任意处生成YYTextView均会导致全局的scrollsToTop 回顶功能失效。
虽然最终追查至`YYTextEffectWindow`中，但整个排查过程还是发现了很多新内容的。

这是正常的回顶功能调用逻辑，基于此一条条的覆写了系统相关的私有函数来判别问题出自何处。
```

-[UICollectionView scrollViewShouldScrollToTop:] 
-[UIScrollView _scrollToTopIfPossible:] ()
-[UIScrollView _scrollToTopFromTouchAtScreenLocation:resultHandler:] ()
-[UIWindow _scrollToTopViewsUnderScreenPointIfNecessary:resultHandler:]_block_invoke.796 ()
-[UIWindow _handleScrollToTopAtXPosition:resultHandler:] ()

//此处能看到有个新鲜的 UIStatusBarManager 是iOS13新增的类，可以看到状态栏的点击事件已经被其接管了。
//经过实践，出问题的时候该方法也能被正常调用故此排上以上调用栈方法。
-[UIStatusBarManager _handleScrollToTopAtXPosition:] ()
-[UIStatusBarManager handleTapAction:] ()
```
开始以为是多个UIScrollView共存时`scrollsToTop `的设置问题，还有`UIScrollViewContentInsetAdjustmentNever`的设置问题。结果都不是...
最终沿着UIScrollView 子类一直查找，找到了`YYTextView` 其中用到的 `YYTextEffectWindow`也进入了视野...
```
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        if (![UIApplication isAppExtension]) {
            one = [self new];
            one.frame = (CGRect){.size = kScreenSize};
            one.userInteractionEnabled = NO;
            //此处能看到 窗口等级是高于状态栏的，但多次尝试等级调整均无果。
            one.windowLevel = UIWindowLevelStatusBar + 1;
            
           //元凶在这里
           //所以，即使关闭了用户交互 但是它竟能够阻挡状态栏的事件，但却对常规Window的事件无任何影响...
            if (@available(iOS 13.0, *)) {
                //费解的结果...
                one.hidden = YES;
            }else{
                one.hidden = NO;
            }
            
            // for iOS 9:
            one.opaque = NO;
            one.backgroundColor = [UIColor clearColor];
            one.layer.backgroundColor = [UIColor clearColor].CGColor;
        }
    });
    return one;
```
> 在UIWindow 上使用addSubview添加子视图，需要注意 暗黑模式切换，并不会向子视图下发状态变更。

* 解决方案：
```
获取系统暗黑切换通知（自行实现），然后重写添加到 UIWindow 的子视图 overrideUserInterfaceStyle 为正确的状态。
```

#### UIScrollView 滚动条异常偏移
> 屏幕旋转可能会触发系统对滚动条的自动修正 
`如果没有修改需求，关闭该特性即可`
```
#ifdef __IPHONE_13_0
   if (@available(iOS 13.0, *)) {
       self.automaticallyAdjustsScrollIndicatorInsets = NO;
   }
#endif
```

#### UICollectionView 异常API
> 该API 在iOS13下会强制将cell置中，导致上部留白。
推测，底部应该也会有留白的情况。
```
#pragma mark - 修复iOS13 下滚动异常API
#ifdef __IPHONE_13_0
- (void)scrollToItemAtIndexPath:(NSIndexPath *)indexPath atScrollPosition:(UICollectionViewScrollPosition)scrollPosition animated:(BOOL)animated{
    
    [super scrollToItemAtIndexPath:indexPath atScrollPosition:scrollPosition animated:animated];

    //修复13下 Cell滚动位置异常
   if (@available(iOS 13.0, *)) {
      //顶部
      if(self.contentOffset.y < 0){
          [self setContentOffset:CGPointZero];
          return;
      }
    
      //底部
      if(self.contentOffset.y > self.contentSize.height){
          [self setContentOffset:CGPointMake(0, self.contentSize.height)];
      }
  }
}
#endif
```

#### UITableViewCell 异常 【疑似iOS13beta4新出现】
> [self addSubView: viewObjA] 
>[self.contentView addSubview:viewObjB]
>`上面两种方式，在遇到折叠需求时。设置self.clipsToBounds=YES 可能会有布局异常`
```
//有其他朋友反馈，遇到的情况正好相反。保险的做法 [添加如下两句] 
self.clipsToBounds = YES;  【无效】
//此处很费解
self.layer.masksTobounds = YES;  【有效】
```

####  疑似布局引擎机制有调整 【有待确定】
> 在调试中发现，如果代码流是这样一种状态  可能会造成  【约束失效】
>`如果遇到诡异问题  需要排查下是否存在约束和绝对布局混用的情况`
```
[viewMain addSubView:viewA];
[viewMain addSubView:viewB];
// 更新 viewA 的约束
 [self.imageBackground mas_updateConstraints:^(MASConstraintMaker *make) {
       make.top.mas_equalTo(50);
 }];

 //立即更新约束
 [viewA.superview setNeedsUpdateConstraints];
 [viewA.superview updateConstraintsIfNeeded];
 [viewA.superview layoutIfNeeded];

//更新容器约束
[viewMain mas_updateConstraints:^(MASConstraintMaker *make) {
       make....
 }];

....
其它处理逻辑
....

// 更新 viewA 的约束 【代码不会生效】
 [self.imageBackground mas_updateConstraints:^(MASConstraintMaker *make) {
       make.top.mas_equalTo(100);
 }];

```
#### WKWebView  
##### 暗黑适配
> 要点：
>1. 模式参数通过 UA 传递
>2. 联调中出现加载闪白问题
```
webView.opaque = false;
```
##### WebJavascriptBridge失效
```
待验证
```

#### 手势影响
> iOS13下，如果在`UITextView`上附加如拖动手势，会发现如果触点落在`UITextView`之上极易触发第一响应者。
实际效果，可对比iOS12之前版本的表现。

```
!!!注意这两项即使开启也不会有改善效果!!!
gesture.cancelsTouchesInView = YES;
gesture.delaysTouchesBegan = YES;
```
* 先看一下相关调用栈
![单击UITextView.png](https://upload-images.jianshu.io/upload_images/2120742-a2551ab17feee00c?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![iOS12下拖动UITextView.png](https://upload-images.jianshu.io/upload_images/2120742-54feac8ebc68bbf0?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![iOS13下拖动UITextView.png](https://upload-images.jianshu.io/upload_images/2120742-3fecc250a0379152?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


* 解决方案：为`UITextView`内部手势添加外部依赖
```
///UITextView override
#ifdef __IPHONE_13_0
//用于解决 iOS13下，可拖动UITextView场景中放大镜手势引起的异常激活编辑状态问题。
-(void)addInteraction:(id<UIInteraction>)interaction{
    [super addInteraction:interaction];

    if(!self.otherGestureRecognizer || self.otherGestureRecognizer.count == 0){
        return;
    }

    if([interaction isKindOfClass:NSClassFromString(@"UITextLoupeInteraction")]){
        UIGestureRecognizer *delayLoupeGesture = [self.gestureRecognizers objectWithBlock:^BOOL(UIGestureRecognizer *obj) {
            return [obj isKindOfClass:NSClassFromString(@"UIVariableDelayLoupeGesture")];
        }];
        if(delayLoupeGesture){
            
            [self.otherGestureRecognizer forEach:^(UIGestureRecognizer *obj) {
                [delayLoupeGesture requireGestureRecognizerToFail:obj];
            }];
        }

    }
}
#endif
```
#### iOS13 下文本框 -- `针对YYTextView`
##### 双光标问题
`注意：之前使用 __IPHONE_13_0 宏错误，应使用 @available(iOS 13.0, *)`
```
具体方法： 
_updateSelectionView {
...
...
}
```
针对`YYTextView`所做的过往修改 [链接直达](https://www.jianshu.com/p/4bae9e942630)

>问题产生原因：
>1. 机制更改  
>2. 之前所做的解决搜狗输入法、系统键盘单词中断问题所做的修改存在问题。

* 机制更改
在真机上尝试了很多次，确认如下：
iOS13下，只要遵循了`UITextInput`相关协议，在进行文本选择操作时系统会自动派生出`UITextSelectionView`系列组件，显然和`YY`自有的`YYTextSelectionView`冲突了。（此处隐藏一颗彩蛋）

* 之前对`YYTextView`所做的代码变更，武断的注释掉了如下方法：
```
if (notify) [_inputDelegate selectionWillChange:self];
if (notify) [_inputDelegate selectionDidChange:self];
```
会造成内部对文本选择相关的数据错乱，当然这种影响目前只在iOS13下能够看到表现。

* 解决方案 ：隐藏系统派生出来的`UITextSelectionView`相关组件
```
/// YYTextView.m 

/// 增加标记位
struct {
       .....
       unsigned int trackingDeleteBackward : 1;  ///< track deleteBackward operation
       unsigned int trackingTouchBegan : 1;  /// < track touchesBegan event
} _state;

/// 方法重写
#pragma mark - override
- (void)addSubview:(UIView *)view{
    
    //解决蓝点问题
    Class Cls_selectionGrabberDot = NSClassFromString(@"UISelectionGrabberDot");
    if ([view isKindOfClass:[Cls_selectionGrabberDot class]]) {
        view.backgroundColor = [UIColor clearColor];
        view.tintColor = [UIColor clearColor];
        view.size = CGSizeZero;
    }
    
    //获取UITextSelectionView
    //解决双光标问题
    Class Cls_selectionView = NSClassFromString(@"UITextSelectionView");

    if ([view isKindOfClass:[Cls_selectionView class]]) {
        view.backgroundColor = [UIColor clearColor];
        view.tintColor = [UIColor clearColor];
        view.hidden = YES;
    }
    
    [super addSubview:view];
    
}

/// 方法修改
/// Replace the range with the text, and change the `_selectTextRange`.
/// The caller should make sure the `range` and `text` are valid before call this method.
- (void)_replaceRange:(YYTextRange *)range withText:(NSString *)text notifyToDelegate:(BOOL)notify{
    if (_isExcludeNeed) {
        notify = NO;
    }

    if (NSEqualRanges(range.asRange, _selectedTextRange.asRange)) {
        //这里的代理方法需要注释掉 【废止】
        //if (notify) [_inputDelegate selectionWillChange:self];
        /// iOS13 下，双光标问题 便是由此而生。
        if (_state.trackingDeleteBackward)[_inputDelegate selectionWillChange:self];
        NSRange newRange = NSMakeRange(0, 0);
        newRange.location = _selectedTextRange.start.offset + text.length;
        _selectedTextRange = [YYTextRange rangeWithRange:newRange];
        //if (notify) [_inputDelegate selectionDidChange:self];
        /// iOS13 下，双光标问题 便是由此而生。
        if (_state.trackingDeleteBackward) [_inputDelegate selectionDidChange:self];
        ///恢复标记
        _state.trackingDeleteBackward = NO;
    } else {
    .....
    .....
}

- (void)deleteBackward {
    //标识出删除动作：用于解决双光标相关问题
    _state.trackingDeleteBackward = YES;
    
    [self _updateIfNeeded];
    .....
    .....
}

- (void)_updateSelectionView {
    _selectionView.frame = _containerView.frame;
    _selectionView.caretBlinks = NO;
    _selectionView.caretVisible = NO;
    _selectionView.selectionRects = nil;
  .....
  .....
    
    if (@available(iOS 13.0, *)) {
        if (_state.trackingTouchBegan) [_inputDelegate selectionWillChange:self];
        [[YYTextEffectWindow sharedWindow] showSelectionDot:_selectionView];
        if (_state.trackingTouchBegan) [_inputDelegate selectionDidChange:self];
    }else{
         [[YYTextEffectWindow sharedWindow] showSelectionDot:_selectionView];
    }

    if (containsDot) {
        [self _startSelectionDotFixTimer];
    } else {
        [self _endSelectionDotFixTimer];
  .....
  .....
}
```
#####  iOS13 新增编辑手势 --`暂时禁用` 

>编辑手势  [描述借鉴](https://www.jianshu.com/p/01cda53e2fc8)
复制：三指捏合
剪切：两次三指捏合
粘贴：三指松开
撤销：三指向左划动（或三指双击）
重做：三指向右划动
快捷菜单：三指单击

```
#ifdef __IPHONE_13_0
- (UIEditingInteractionConfiguration)editingInteractionConfiguration{
    return UIEditingInteractionConfigurationNone;
}
#endif
```

#### UITabbar 

> `UITabbar` 层次发生改变，无法通过设置` shadowImage`去掉上面的线。
[@还是老徐ooo](https://www.jianshu.com/u/c4516646f1b8) 反馈

* 解决方案：[stackoverflow](https://stackoverflow.com/questions/58062613/cant-set-tab-bar-shadow-image-in-ios-13) `自行转化为OC哦`
```
if #available(iOS 13, *) {
  let appearance = self.tabBar.standardAppearance.copy()
  appearance.backgroundImage = UIImage()
  appearance.shadowImage = UIImage()
  appearance.shadowColor = .clear
  self.tabBar.standardAppearance = appearance
} else {
  self.tabBar.shadowImage = UIImage()
  self.tabBar.backgroundImage = UIImage()
}
```

###### UITabBar frame 设置失效 
```
待验证
```

####  layoutSubviews 相关
```
//容错性更高 
//具体待补充
```
#### removeFromSuperview
> 调试老项目偶然发现 `SVProgressHUD`在iOS13上会出现提示文案空白的问题，切换到iOS11以及12等版本后发现了该异常。

* 相关代码段 `SVProgressHUD`
``` 
- (void)dismiss {
    NSDictionary *userInfo = [self notificationUserInfo];
    [[NSNotificationCenter defaultCenter] postNotificationName:SVProgressHUDWillDisappearNotification
                                                        object:nil
                                                      userInfo:userInfo];
    
    self.activityCount = 0;
    [UIView animateWithDuration:0.15
                          delay:0
                        options:UIViewAnimationCurveEaseIn | UIViewAnimationOptionAllowUserInteraction
                     animations:^{
                         self.hudView.transform = CGAffineTransformScale(self.hudView.transform, 0.8f, 0.8f);
                         if(self.isClear) // handle iOS 7 UIToolbar not answer well to hierarchy opacity change
                             self.hudView.alpha = 0.0f;
                         else
                             self.alpha = 0.0f;
                     }
                     completion:^(BOOL finished){
                         if(self.alpha == 0.0f || self.hudView.alpha == 0.0f) {
                             self.alpha = 0.0f;
                             self.hudView.alpha = 0.0f;
                             
                             [[NSNotificationCenter defaultCenter] removeObserver:self];
                             [self cancelRingLayerAnimation];
                             //解决提示不显示问题
                             [_stringLabel removeFromSuperview];
                             _stringLabel = nil;

                             [_hudView removeFromSuperview];
                             _hudView = nil;
                             
                             [_overlayView removeFromSuperview];
                             _overlayView = nil;
                             
                             [_indefiniteAnimatedView removeFromSuperview];
                             _indefiniteAnimatedView = nil;
                             
                             UIAccessibilityPostNotification(UIAccessibilityScreenChangedNotification, nil);
                             
                             [[NSNotificationCenter defaultCenter] postNotificationName:SVProgressHUDDidDisappearNotification
                                                                                 object:nil
                                                                               userInfo:userInfo];
                             
                             // Tell the rootViewController to update the StatusBar appearance
#if !defined(SV_APP_EXTENSIONS)
                             UIViewController *rootController = [[UIApplication sharedApplication] keyWindow].rootViewController;
                             if ([rootController respondsToSelector:@selector(setNeedsStatusBarAppearanceUpdate)]) {
                                 [rootController setNeedsStatusBarAppearanceUpdate];
                             }
#endif
                             // uncomment to make sure UIWindow is gone from app.windows
                             //NSLog(@"%@", [UIApplication sharedApplication].windows);
                             //NSLog(@"keyWindow = %@", [UIApplication sharedApplication].keyWindow);
                         }
                     }];
}
......
......

- (UILabel *)stringLabel {
    if (!_stringLabel) {
        _stringLabel = [[UILabel alloc] initWithFrame:CGRectZero];
		_stringLabel.backgroundColor = [UIColor clearColor];
		_stringLabel.adjustsFontSizeToFitWidth = YES;
        _stringLabel.textAlignment = NSTextAlignmentCenter;
		_stringLabel.baselineAdjustment = UIBaselineAdjustmentAlignCenters;
        _stringLabel.numberOfLines = 0;
    }
    
    if(!_stringLabel.superview){
        [self.hudView addSubview:_stringLabel];
    }
    _stringLabel.textColor = SVProgressHUDForegroundColor;
    _stringLabel.font = SVProgressHUDFont;
    
    return _stringLabel;
}
```

> 问题的实质原因在于，`_stringLabel.superview` 也就是 `_hudView`在调用`[_hudView removeFromSuperview]`后：
> iOS12以下  `_stringLabel.superview = nil`
> iOS13         `_stringLabel.superview != nil`

`注意：以后如果使用view.superview 来进行判断的话还是多思考一下吧`

#### 一断毒代码 `注：仅在iOS13以上版本可见`
> *********神奇表现*********
前提：系统旋转开关处于锁定状态
DEV 调试无任何问题 --- `No Problem`
Adhoc 和 Release，只要用户手动操作一次前后台切换下面的方法即死翘翘。 --- `Die`
`此问题为多种综合因素造成，各位并不一定遇到。`
```
#pragma mark - 旋转屏幕
+(void)changeOrientation:(UIInterfaceOrientation)toOrientation{
 
  if ([[UIDevice currentDevice] respondsToSelector:@selector(setOrientation:)]) {
        SEL selector = NSSelectorFromString(@"setOrientation:");
        NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:[UIDeviceinstanceMethodSignatureForSelector:selector]];
        [invocation setSelector:selector];
        [invocation setTarget:[UIDevice currentDevice]];
        int val = UIInterfaceOrientationLandscapeRight;
        [invocation setArgument:&val atIndex:2];
        [invocation invoke];
    }
}
```
* 解决方案： `疑似函数内部栈指针错乱`
```
要解决看起来很简单，只需要在方法函数头部添加几句不会被编译器优化掉的代码即可。

#pragma mark - 旋转屏幕
+(void)changeOrientation:(UIInterfaceOrientation)toOrientation{
  
  [self uploadLog:@"旋转屏幕"];

  if ([[UIDevice currentDevice] respondsToSelector:@selector(setOrientation:)]) {
        SEL selector = NSSelectorFromString(@"setOrientation:");
        NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:[UIDeviceinstanceMethodSignatureForSelector:selector]];
        [invocation setSelector:selector];
        [invocation setTarget:[UIDevice currentDevice]];
        int val = UIInterfaceOrientationLandscapeRight;
        [invocation setArgument:&val atIndex:2];
        [invocation invoke];
    }
}
```

#### 状态栏相关
##### 状态栏的显示与隐藏 
【方案一】
> 虽然下面的方法被列为`DEPRECATED`，在iOS13上仍然奏效。

```
- (void)setStatusBarHidden:(BOOL)hidden withAnimation:(UIStatusBarAnimation)animation API_DEPRECATED("Use -[UIViewController prefersStatusBarHidden]", ios(3.2, 9.0)) API_UNAVAILABLE(tvos);
```
【方案二】
> 奇淫技巧：通过复写私有方法，其实可以做到全局状态栏不显示，但并不推荐。

```
!!!还是不要尝试的好!!!
```

#####  获取状态栏 `注意本质为新创建一个状态栏，和当前系统状态栏可能不一致`
```
        if (@available(iOS 13.0, *)) {
            UIView *_localStatusBar = [[UIApplication sharedApplication].keyWindow.windowScene.statusBarManager performSelector:@selector(createLocalStatusBar)];
            UIView * statusBar = [_localStatusBar performSelector:@selector(statusBar)];
            // 注意此代码不生效 用于绘制
           // [statusBar drawViewHierarchyInRect:statusBar.bounds afterScreenUpdates:NO];
            [statusBar.layer renderInContext:context];

        } else {
            // Fallback on earlier versions
        }
```
#####  横屏时状态栏`显示且在左侧或右侧` 
> 常见于视频播放页需求
原因1：视频播放器所属的 UIViewController 未能正确旋转所致。
原因2：iOS13下API失效
` [[UIApplication sharedApplication] setStatusBarOrientation:UIInterfaceOrientationLandscapeLeft];`
* 解决方案：
```
允许控制器旋转，iOS13横屏时会自动隐藏状态栏。
```
* 状态栏设置背景色
> 由于iOS13下当前状态栏实际上被系统强制接管了，是拿不到的。`🐶 或者是有我没找到的API`
* 解决方案：
```
    UIWindow *keyWindow = [UIApplication sharedApplication].keyWindow;
    UIView *viewStatusColorBlend = [[UIView alloc]initWithFrame:keyWindow.windowScene.statusBarManager.statusBarFrame];
    viewStatusColorBlend.backgroundColor = Color;
    [keyWindow addSubview:viewStatusColorBlend];
```

##### 横屏时显示状态栏

【方案一】
> 针对iOS13 需要明确如下: 
> 1. iOS13 设备支持情况，不同设备有着不同的表现。
> 2. 该方案不能保证一定通过苹果审核，`请慎重对待!!!` `请慎重对待!!!` `请慎重对待!!!`
> 3. 创建了本地状态栏后，是会对系统的状态栏造成不确定影响的，所以下面的方法谨慎的使用了 `addSubView ` 、`removeFromSuperview`。

* 不支持的设备 [图片来源](https://baijiahao.baidu.com/s?id=1634386032243111849&wfr=spider&for=pc)

![iOS13不支持的设备.jpg](https://upload-images.jianshu.io/upload_images/2120742-cf15f9ea6002c024?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* 状态栏在不同设备的表现

A: iPhone X ~ iPhone ... Max （刘海屏）：状态栏显示时，分左、右两段，其中左上角为时间、右上角为🔋等信息，且在`iOS13 下横屏不显示`。

B: iPhone7 ~ iPhone 8P （矩形屏）：状态栏显示时，分左、中、右三段，其中时间居中显示，且在`iOS13 下横屏不显示`。

C: iPAD （糊脸屏）：状态栏显示时，分左、右两段，其中左上角为时间，右上角为🔋等信息，且在`iOS13 （纠正：iPad OS13）下默认显示`。

`总结: A 、B 两种情况需要特殊处理`

* 效果预览

![Simulator Screen Shot - iPhone Xʀ - 2019-09-27 at 16.51.25.png](https://upload-images.jianshu.io/upload_images/2120742-c1ad31084af22b8b?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![Simulator Screen Shot - iPhone 8 - 2019-09-27 at 16.54.08.png](https://upload-images.jianshu.io/upload_images/2120742-47226c2b2809e71e?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* 解决方案： 参考上面的获取状态栏方法
```
/** 状态栏--iOS13 */
@property (nonatomic, strong) UIView  *viewStatusBar API_AVAILABLE(ios(13.0));

- (UIView *)viewStatusBar{
    if (!_viewStatusBar) {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wundeclared-selector"
        _viewStatusBar = [[UIApplication sharedApplication].keyWindow.windowScene.statusBarManager performSelector:@selector(createLocalStatusBar)];
        _viewStatusBar.backgroundColor = [UIColor clearColor];
        _viewStatusBar.hidden = YES;
        _viewStatusBar.tag = 784321;
        _viewStatusBar.overrideUserInterfaceStyle = UIUserInterfaceStyleLight;
        
        //手动设置状态栏为白色样式
        UIView *statusBar = [[_viewStatusBar valueForKey:@"_statusBar"]valueForKey:@"_statusBar"];
        [statusBar performSelector:@selector(setForegroundColor:) withObject:[UIColor whiteColor]];
#pragma clang diagnostic pop
    }
    return _viewStatusBar;
}

#param mark - 横屏方法
- (void)setOrientationLandscapeConstraint {
 //状态栏处理
    if (@available(iOS 13.0, *)) {
        if(![self viewWithTag:784321] && !DP_IS_IPAD){
            self.viewStatusBar.hidden = NO;
            [self addSubview:self.viewStatusBar];
            //我这里是为了避开 其它控件，具体的按各自的需求来
            CGFloat fltLeftMargin = DP_IS_IPHONEX ? (-24) : 0;
            CGFloat fltTopMargin = DP_IS_IPHONEX ? (-12) : 0;
            UIView *leftView = DP_IS_IPHONEX ? self.lblTitle : self;
            
            [self.viewStatusBar mas_makeConstraints:^(MASConstraintMaker *make) {
                 make.left.mas_equalTo(leftView.mas_left).offset(fltLeftMargin);
                 make.top.mas_equalTo(fltTopMargin);
                 make.right.mas_equalTo(0);
            }];
        }
    }
}

#param mark - 竖屏方法
- (void)setOrientationPortraitConstraint { 
    //状态栏处理
    if (@available(iOS 13.0, *)) {
        if (!DP_IS_IPAD) {
            self.viewStatusBar.hidden = YES;
            if ([self viewWithTag:784321]) {
                [self.viewStatusBar removeFromSuperview];
            }
        }
    }
}

```

* 本来希望是达到和爱奇艺一样的效果，但上面的方式如果顺利过审的话已经满足需求了。
`当然 如果哪位大佬知道怎么实现的，还望不吝赐教。`

![爱奇艺 XS Max 截取](https://upload-images.jianshu.io/upload_images/2120742-a2182909dd3dbc78?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

【方案二】`重点：显示系统状态栏`
> 该方案使用原生状态栏，而非通过私有API创建。
> 额外注意的是：虽然能够正常显示，但对状态栏`Frame`修改的尝试均失败。

```
@implementation UIStatusBarManager (Extend)
/// 更改默认配置
- (CGFloat)defaultStatusBarHeightInOrientation:(UIInterfaceOrientation)orientation {
    if (orientation ==  UIInterfaceOrientationLandscapeLeft || orientation == UIInterfaceOrientationLandscapeRight ) {
    /// 主要就是这里    
    return 20;
    }
    /// 此处仅做示例，实际使用需要针对不同设备进行适配。
    return 44;
}

@end
```
~~【方案三】~~
`印证辅助：Alook浏览器内置的视频播放器和爱奇艺的效果基本一致，作者明确了横屏状态栏是自实现，而非系统。`

> 分析爱奇艺的实现，个人鄙见：`主要讲实现方案`
>1. 视频最终页构成：仅支持竖屏的控制器 + 自定义承载视频播放器的`UIWindow`
>2. 横竖屏逻辑：通过弹出一个模态控制器，来控制真是的旋转方向，但其内容一定是透明的。（具体大概意思就是  视频最终页（竖屏） ---> 透明的模态控制器（横屏））
>3. 视频播放器UIWindow转场效果

* `注意：该API已经失效，这也是很多人一直在找横竖屏替代方案的原因。`
```
 [[UIApplication sharedApplication] setStatusBarOrientation:currentOrientation animated:NO];
```

* `调试时发现的一个私有方法：猜测是用来代替上面方法的，但实际调用无效果。`
```
setStatusBarOrientation:fromOrientation:windowScene:animationParameters:updateBlock:
```
【方案三】`使用 iPAPatch 分析爱奇艺实现而来`
> 横屏时的电量与时间显示，`CustomUIView`没有疑问 。
> 视频横竖屏切换，使用的也是新建 `CustomUIWindow`，只不过其`rootViewController`是`Push`而非`模态`。
> 大体思路： ~~`CustomUIWindow` 接替原`UIWindow`成为`keyWindow`~~，转而使用其`rootViewController`控制当前屏幕的实际方向。

* 验证代码：`实际效果 等同于  [[UIApplication sharedApplication] setStatusBarOrientation:currentOrientation animated:NO];`

![扫描 2019年12月18日 下午7.21.png](https://upload-images.jianshu.io/upload_images/2120742-fbebff1ea7a414f8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


```
//切换横竖屏方法 示例
+(void)changeOrientation:(UIInterfaceOrientation)toOrientation{
    UIApplication * app = [UIApplication sharedApplication];
    UIWindow * appWindow = app.delegate.window;
    //大小随意
    CGRect rect = CGRectMake(0, 0, 30, 30);
    //这里其实可以 将toOrientation 传递给 TempViewController 用于

    if (!_customWindow) {
        _customWindow = [[UIWindow alloc] initWithFrame:rect];
        _customWindow.backgroundColor = [UIColor cyanColor];
        TempViewController * viewctrl = [[TempViewController alloc] init];
         _customWindow.rootViewController = viewctrl;
    }
   //其实并不一定要接替keyWindow，只需保证多Window中仅有一个能够支持旋转即可。
   //[_customWindow makeKeyAndVisible];
}
```

```
#import "TempViewController.h"

@interface TempViewController ()

@end

@implementation TempViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view.
    
    self.view.backgroundColor = [UIColor greenColor];
    
    UIView *viewRed = [[UIView alloc]initWithFrame:CGRectMake(0, 0, 10, 10)];
    viewRed.backgroundColor = [UIColor redColor];
    [self.view addSubview:viewRed];
    
}

- (BOOL)shouldAutorotate{
    return YES;
}

- (UIInterfaceOrientationMask)supportedInterfaceOrientations{
    return UIInterfaceOrientationMaskLandscape;
}


@end
```
###### 小结：关于横竖屏以及状态栏的问题，总结至此。

####  代码规范问题相关  `NSDate` 
###### 命名
>  [@frankfan1990](https://www.jianshu.com/u/c73dff241440) 反馈应用切换到后台时Crash，全局断点捕获不到并且直接断进了main函数。

问题原因：
 自行实现的`NSDate`的`category`，里面有一个方法叫 `+(NSInterger)now `的方法。该方法在iOS13 中已有实现，具体崩溃原因待验证。

解决方案： `改写系统提供类的方法或者新增方法，注意避免冲突。最好将命名带有前后缀，对于属性也一样适用。`

###### 奇淫巧技`请注意`
> 三目运算符
>` result = x?x:y;  ==> result = x?:y;` 
```
可能是由于Xcode11 编译器或者语法器的调整，上面的技巧可能不会得到你想要的结果了。
```

#### VOIP 相关
> [@2742810c4ba1](https://www.jianshu.com/u/2742810c4ba1) 反馈到后台收不到推送，并且有报错闪退。

解决方案： [@叫点什么好呢](https://www.jianshu.com/u/96344462e2fa) 提到push 推送的`data数据结构`发生变更，需要更新SDK。

#### 3D Touch  &&  Haptic Touch 
> 支持判断

```
注意：使用该判断，并且直接return的话  可能你同时也就错过了 Haptic Touch。
traitCollection.forceTouchCapability == UIForceTouchCapabilityUnavailable

可能是由于苹果的疏忽，Haptic Touch的能力并未和3D Touch统一。
我能想到的唯一判断方式，是通过系统版本。
//是否支持触摸力度检测
if(self.window.traitCollection.forceTouchCapability == UIForceTouchCapabilityUnavailable){
    //不支持触摸力度检测时，检测是否支持 haptic touch。
    //haptic touch，判断依据 仅iOS 13.0 以上版本。
    if(@available(iOS 13.0, *)) {
     
     }else{
            return;
     }
} 
```


 #### 补充
##### 状态栏调试补充
> 在调试过程中，挖了一些相关数据。
下面会贴出来，目前对于状态栏问题的解决并不是很满意。
所以希望有兴趣的朋友，能够一起找寻下最佳方案。

###### 配置及结构
* 为什么iOS13 横屏会不显示状态栏？
![默认配置.png](https://upload-images.jianshu.io/upload_images/2120742-e681a7f3fe35f1ce?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* 状态栏层级结构
![状态栏层级结构.png](https://upload-images.jianshu.io/upload_images/2120742-6ab49928229e1c93?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###### 私有方法挖掘
* UIStatusBarManager
```
.cxx_destruct
setWindowScene:
_settingsDiffActionsForScene:
initWithScene:
_scene
_setScene:
windowScene
isStatusBarHidden
defaultStatusBarHeightInOrientation:
statusBarStyle
statusBarHeight
updateStatusBarAppearance
updateLocalStatusBars
statusBarHidden
statusBarAlpha
setupForSingleLocalStatusBar
statusBarFrameForStatusBarHeight:
updateStatusBarAppearanceWithAnimationParameters:
_updateStatusBarAppearanceWithClientSettings:transitionContext:animationParameters:
_updateVisibilityForWindow:targetOrientation:animationParameters:
_updateStyleForWindow:animationParameters:
_updateAlpha
_visibilityChangedWithOriginalOrientation:targetOrientation:animationParameters:
activateLocalStatusBar:
_updateLocalStatusBar:
statusBarFrame
_handleScrollToTopAtXPosition:
_adjustedLocationForXPosition:
updateStatusBarAppearanceWithClientSettings:transitionContext:
deactivateLocalStatusBar:
createLocalStatusBar
handleTapAction:
localStatusBars
setLocalStatusBars:
statusBarPartStyles
isInStatusBarFadeAnimation
debugMenuHandler
setDebugMenuHandler:
```
* UIStatusBar_Modern 
```
.cxx_destruct
 layoutSubviews
 intrinsicContentSize
 setMode:
 setOrientation:
 currentStyle
 forceUpdate:
 statusBar
 setStatusBar:
 forceUpdateData:
 setEnabledPartIdentifiers:
 setAvoidanceFrame:
 frameForPartWithIdentifier:
 alphaForPartWithIdentifier:
 setAlpha:forPartWithIdentifier:
 setOffset:forPartWithIdentifier:
 jiggleLockIcon
 setForegroundColor:animationParameters:
 setStyleRequest:animationParameters:
 enabledPartIdentifiers
 setAction:forPartWithIdentifier:
 statusBarServer:didReceiveStatusBarData:withActions:
 statusBarServer:didReceiveStyleOverrides:
 statusBarServer:didReceiveDoubleHeightStatusString:forStyle:
 statusBarStateProvider:didPostStatusBarData:withActions:
 defaultDoubleHeight
 setForegroundAlpha:animationParameters:
 _effectiveStyleFromStyle:
 actionForPartWithIdentifier:
 offsetForPartWithIdentifier:
 _initWithFrame:showForegroundView:wantsServer:inProcessStateProvider:
 setLegibilityStyle:animationParameters:
 _requestStyle:partStyles:animationParameters:forced:
 _implicitStyleOverrideForStyle:
 _effectiveDataFromData:activeOverride:canUpdateBackgroundActivity:
 _updateWithData:force:
 _requestStyle:partStyles:legibilityStyle:foregroundColor:animationParameters:
 _updateSemanticContentAttributeFromLegacyData:
 _dataFromLegacyData:
```

* _UIStatusBarLocalView
```
.cxx_destruct
initWithFrame:
willMoveToWindow:
statusBar
setStatusBar:
```
* _UIStatusBar
```
description
.cxx_destruct
setAction:
action
layoutSubviews
intrinsicContentSize
mode
initWithStyle:
setMode:
items
style
orientation
setOrientation:
foregroundColor
containerView
_updateDisplayedItemsWithData:styleAttrib
updateConstraints
setStyle:
gestureRecognizerShouldBegin:
currentData
traitCollectionDidChange:
setItems:
setForegroundColor:
areAnimationsEnabled
setSemanticContentAttribute:
updateWithData:
setStyleAttributes:
styleAttributes
itemWithIdentifier:
regions
_traitCollectionForChildEnvironment:
_accessibilityHUDGestureManager:HUDItemFo
_accessibilityHUDGestureManager:gestureLi
_accessibilityHUDGestureManager:shouldRec
_accessibilityHUDGestureManager:shouldTer
_accessibilityHUDGestureManager:showHUDIt
_dismissAccessibilityHUDForGestureManager
foregroundView
actionGestureRecognizer
avoidanceFrame
setEnabledPartIdentifiers:
setAvoidanceFrame:
frameForPartWithIdentifier:
alphaForPartWithIdentifier:
setAlpha:forPartWithIdentifier:
setOffset:forPartWithIdentifier:
enabledPartIdentifiers
setOverlayData:
setAction:forPartWithIdentifier:
setStyle:forPartWithIdentifier:
overlayData
updateCompletionHandler
setUpdateCompletionHandler:
setForegroundView:
visualProvider
_forceLayoutEngineSolutionInRationalEdges
resizeSubviewsWithOldSize:
currentAggregatedData
updateWithAnimations:
styleAttributesForStyle:
displayItemIdentifiersInRegionsWithIdenti
frameForDisplayItemWithIdentifier:
frameForPartWithIdentifier:includeInterna
dataAggregator
displayItemWithIdentifier:
regionWithIdentifier:
stateForDisplayItemWithIdentifier:
setDisplayItemStates:
_updateWithAggregatedData:
statusBarGesture:
_setVisualProviderClass:
_visualProviderClass
_prepareVisualProviderIfNeeded
_effectiveTargetScreen
_updateStyleAttributes
_performWithInheritedAnimation:
_effectiveStyleFromStyle:
_updateWithData:completionHandler:
_prepareAnimations:
_performAnimations:
_fixupDisplayItemAttributes
_delayUpdatesWithKeys:fromAnimation:
_updateRegionItems
_rearrangeOverflowedItems
_frameForActionable:actionInsets:
_gestureRecognizer:touchInsideActionable:
_gestureRecognizer:pressInsideActionable:
_frameForActionable:
_pressFrameForActionable:
_gestureRecognizer:isInsideActionable:
_regionsForPartWithIdentifier:includeInte
_actionablesForPartWithIdentifier:include
_itemWithIdentifier:createIfNeeded:
_statusBarWindowForAccessibilityHUD
_setVisualProviderClassName:
_visualProviderClassName
resetVisualProvider
actionForPartWithIdentifier:
offsetForPartWithIdentifier:
styleForPartWithIdentifier:
dependentDataEntryKeys
animationContextId
itemsDependingOnKeys:
itemIdentifiersInRegionsWithIdentifiers:
dataEntryKeysForItemsWithIdentifiers:
targetScreen
setTargetScreen:
displayItemStates
targetActionable
setTargetActionable:
accessibilityHUDGestureManager
setAccessibilityHUDGestureManager:
```
##### 相关数据 `看了这些就理解了别人是怎么挖到信息的`
```
(lldb) po [statusBar performSelector:@selector(currentData)]
<_UIStatusBarData: 0x7fee29fdada0: 
mainBatteryEntry=<_UIStatusBarDataBatteryEntry: 0x600001713c90: isEnabled=1, capacity=100, state=2, saverModeActive=0, prominentlyShowsDetailString=0, detailString=100%>, 
secondaryCellularEntry=<_UIStatusBarDataCellularEntry: 0x600003c4d9e0: isEnabled=1, rawValue=0, displayValue=0, displayRawValue=0, status=0, lowDataModeActive=0, type=5, wifiCallingEnabled=0, callForwardingEnabled=0, showsSOSWhenDisabled=0>, 
backNavigationEntry=<_UIStatusBarDataStringEntry: 0x60000191e540: isEnabled=0>, 
vpnEntry=<_UIStatusBarDataEntry: 0x600001aae360: isEnabled=0>,
radarEntry=<_UIStatusBarDataBoolEntry: 0x600001aae280: isEnabled=0, boolValue=0>, 
rotationLockEntry=<_UIStatusBarDataEntry: 0x600001aadf20: isEnabled=0>, dateEntry=<_UIStatusBarDataStringEntry: 0x60000191e560: isEnabled=1, stringValue=Thu Sep 26>, 
quietModeEntry=<_UIStatusBarDataBoolEntry: 0x600001aae250: isEnabled=0, boolValue=0>, 
timeEntry=<_UIStatusBarDataStringEntry: 0x60000191e580: isEnabled=1, stringValue=5:15 PM>, 
personNameEntry=<_UIStatusBarDataStringEntry: 0x60000191e5a0: isEnabled=0>, 
cellularEntry=<_UIStatusBarDataCellularEntry: 0x600003c4da40: isEnabled=1, rawValue=0, displayValue=0, displayRawValue=0, status=1, lowDataModeActive=0, type=5, string=Carrier, wifiCallingEnabled=0, callForwardingEnabled=0, showsSOSWhenDisabled=0>, 
assistantEntry=<_UIStatusBarDataEntry: 0x600001aae210: isEnabled=0>, 
bluetoothEntry=<_UIStatusBarDataBluetoothEntry: 0x60000191e5c0: isEnabled=0, state=0>, 
ttyEntry=<_UIStatusBarDataEntry: 0x600001aacbc0: isEnabled=0>, 
voiceControlEntry=<_UIStatusBarDataVoiceControlEntry: 0x60000191e5e0: isEnabled=0, type=0>, 
carPlayEntry=<_UIStatusBarDataEntry: 0x600001aacc00: isEnabled=0>, 
wifiEntry=<_UIStatusBarDataWifiEntry: 0x6000003b2880: isEnabled=1, rawValue=0, displayValue=3, displayRawValue=0, status=5, lowDataModeActive=0, type=0>, 
liquidDetectionEntry=<_UIStatusBarDataEntry: 0x600001aae160: isEnabled=0>, 
shortTimeEntry=<_UIStatusBarDataStringEntry: 0x60000191e600: isEnabled=1, stringValue=5:15>, 
studentEntry=<_UIStatusBarDataEntry: 0x600001aaeac0: isEnabled=0>, 
tetheringEntry=<_UIStatusBarDataTetheringEntry: 0x60000191e620: isEnabled=0, connectionCount=0>, 
alarmEntry=<_UIStatusBarDataEntry: 0x600001aae0e0: isEnabled=0>, 
activityEntry=<_UIStatusBarDataActivityEntry: 0x60000191e640: isEnabled=0, type=0, displayId=com.driver.feng>, 
locationEntry=<_UIStatusBarDataLocationEntry: 0x60000191e660: isEnabled=0, type=1>, 
airPlayEntry=<_UIStatusBarDataEntry: 0x600001aadfc0: isEnabled=0>, 
deviceNameEntry=<_UIStatusBarDataStringEntry: 0x60000191e680: isEnabled=0>, 
lockEntry=<_UIStatusBarDataLockEntry: 0x60000191e6a0: isEnabled=0, unlockFailureCount=0>, 
electronicTollCollectionEntry=<_UIStatusBarDataBoolEntry: 0x600001aae030: isEnabled=0, boolValue=0>, 
thermalEntry=<_UIStatusBarDataThermalEntry: 0x60000191e6c0: isEnabled=0, color=0, sunlightMode=0>, 
backgroundActivityEntry=<_UIStatusBarDataBackgroundActivityEntry: 0x600001713ed0: isEnabled=0, type=0>, 
forwardNavigationEntry=<_UIStatusBarDataStringEntry: 0x60000191e700: isEnabled=0>, 
airplaneModeEntry=<_UIStatusBarDataEntry: 0x600001aadfa0: isEnabled=0>>

```

```
(lldb) po [statusBar performSelector:@selector(items)]
{
    "<_UIStatusBarIdentifier: 0x600001907300: object=_UIStatusBarIndicatorTTYItem>" = "<_UIStatusBarIndicatorTTYItem: 0x6000003b1d40: identifier=<_UIStatusBarIdentifier: 0x600001907300: object=_UIStatusBarIndicatorTTYItem>>";
    "<_UIStatusBarIdentifier: 0x600001903a00: object=_UIStatusBarTimeItem>" = "<_UIStatusBarTimeItem: 0x600003b7f610: identifier=<_UIStatusBarIdentifier: 0x600001903a00: object=_UIStatusBarTimeItem>>";
    "<_UIStatusBarIdentifier: 0x600001907380: object=_UIStatusBarIndicatorAlarmItem>" = "<_UIStatusBarIndicatorAlarmItem: 0x6000003b1e00: identifier=<_UIStatusBarIdentifier: 0x600001907380: object=_UIStatusBarIndicatorAlarmItem>>";
    "<_UIStatusBarIdentifier: 0x600001906f80: object=_UIStatusBarSpacerItem>" = "<_UIStatusBarSpacerItem: 0x60000177ea00: identifier=<_UIStatusBarIdentifier: 0x600001906f80: object=_UIStatusBarSpacerItem>>";
    "<_UIStatusBarIdentifier: 0x600001903d60: object=_UIStatusBarActivityItem_Split>" = "<_UIStatusBarActivityItem_Split: 0x6000003b2000: identifier=<_UIStatusBarIdentifier: 0x600001903d60: object=_UIStatusBarActivityItem_Split>>";
    "<_UIStatusBarIdentifier: 0x600001903960: object=_UIStatusBarCellularCondensedItem>" = "<_UIStatusBarCellularCondensedItem: 0x600002aa8ea0: identifier=<_UIStatusBarIdentifier: 0x600001903960: object=_UIStatusBarCellularCondensedItem>>";
    "<_UIStatusBarIdentifier: 0x600001903a80: object=_UIStatusBarVoiceControlPillItem>" = "<_UIStatusBarVoiceControlPillItem: 0x600003b7f980: identifier=<_UIStatusBarIdentifier: 0x600001903a80: object=_UIStatusBarVoiceControlPillItem>>";
    "<_UIStatusBarIdentifier: 0x600001907400: object=_UIStatusBarIndicatorRotationLockItem>" = "<_UIStatusBarIndicatorRotationLockItem: 0x6000003b1e40: identifier=<_UIStatusBarIdentifier: 0x600001907400: object=_UIStatusBarIndicatorRotationLockItem>>";
    "<_UIStatusBarIdentifier: 0x600001907000: object=_UIStatusBarBluetoothItem>" = "<_UIStatusBarBluetoothItem: 0x6000003b1f00: identifier=<_UIStatusBarIdentifier: 0x600001907000: object=_UIStatusBarBluetoothItem>>";
    "<_UIStatusBarIdentifier: 0x600001906ee0: object=_UIStatusBarIndicatorAirplaneModeItem>" = "<_UIStatusBarIndicatorAirplaneModeItem: 0x6000003b2080: identifier=<_UIStatusBarIdentifier: 0x600001906ee0: object=_UIStatusBarIndicatorAirplaneModeItem>>";
    "<_UIStatusBarIdentifier: 0x600001907760: object=_UIStatusBarBuildVersionItem>" = "<_UIStatusBarBuildVersionItem: 0x60000177e310: identifier=<_UIStatusBarIdentifier: 0x600001907760: object=_UIStatusBarBuildVersionItem>>";
    "<_UIStatusBarIdentifier: 0x600001906d20: object=_UIStatusBarIndicatorVPNItem>" = "<_UIStatusBarIndicatorVPNItem: 0x6000003b1fc0: identifier=<_UIStatusBarIdentifier: 0x600001906d20: object=_UIStatusBarIndicatorVPNItem>>";
    "<_UIStatusBarIdentifier: 0x600001907480: object=_UIStatusBarIndicatorQuietModeItem>" = "<_UIStatusBarIndicatorQuietModeItem: 0x6000003b1e80: identifier=<_UIStatusBarIdentifier: 0x600001907480: object=_UIStatusBarIndicatorQuietModeItem>>";
    "<_UIStatusBarIdentifier: 0x6000019075a0: object=_UIStatusBarActivityItem_SyncOnly>" = "<_UIStatusBarActivityItem_SyncOnly: 0x60000177dad0: identifier=<_UIStatusBarIdentifier: 0x6000019075a0: object=_UIStatusBarActivityItem_SyncOnly>>";
    "<_UIStatusBarIdentifier: 0x6000019076c0: object=_UIStatusBarIndicatorLiquidDetectionItem>" = "<_UIStatusBarIndicatorLiquidDetectionItem: 0x6000003b1f40: identifier=<_UIStatusBarIdentifier: 0x6000019076c0: object=_UIStatusBarIndicatorLiquidDetectionItem>>";
    "<_UIStatusBarIdentifier: 0x600001907080: object=_UIStatusBarThermalItem>" = "<_UIStatusBarThermalItem: 0x6000003b1c80: identifier=<_UIStatusBarIdentifier: 0x600001907080: object=_UIStatusBarThermalItem>>";
    "<_UIStatusBarIdentifier: 0x600001906dc0: object=_UIStatusBarSecondaryCellularExpandedItem>" = "<_UIStatusBarSecondaryCellularExpandedItem: 0x60000366c380: identifier=<_UIStatusBarIdentifier: 0x600001906dc0: object=_UIStatusBarSecondaryCellularExpandedItem>>";
    "<_UIStatusBarIdentifier: 0x600001907500: object=_UIStatusBarIndicatorLocationItem>" = "<_UIStatusBarIndicatorLocationItem: 0x6000003b1ec0: identifier=<_UIStatusBarIdentifier: 0x600001907500: object=_UIStatusBarIndicatorLocationItem>>";
    "<_UIStatusBarIdentifier: 0x600001907100: object=_UIStatusBarIndicatorAssistantItem>" = "<_UIStatusBarIndicatorAssistantItem: 0x6000003b1cc0: identifier=<_UIStatusBarIdentifier: 0x600001907100: object=_UIStatusBarIndicatorAssistantItem>>";
    "<_UIStatusBarIdentifier: 0x600001907620: object=_UIStatusBarBatteryItem>" = "<_UIStatusBarBatteryItem: 0x600003b7f4d0: identifier=<_UIStatusBarIdentifier: 0x600001907620: object=_UIStatusBarBatteryItem>>";
    "<_UIStatusBarIdentifier: 0x600001903de0: object=_UIStatusBarNavigationItem>" = "<_UIStatusBarNavigationItem: 0x60000177fc90: identifier=<_UIStatusBarIdentifier: 0x600001903de0: object=_UIStatusBarNavigationItem>>";
    "<_UIStatusBarIdentifier: 0x600001907180: object=_UIStatusBarIndicatorAirPlayItem>" = "<_UIStatusBarIndicatorAirPlayItem: 0x6000003b1d00: identifier=<_UIStatusBarIdentifier: 0x600001907180: object=_UIStatusBarIndicatorAirPlayItem>>";
    "<_UIStatusBarIdentifier: 0x600001906c60: object=_UIStatusBarWifiItem>" = "<_UIStatusBarWifiItem: 0x6000003b2480: identifier=<_UIStatusBarIdentifier: 0x600001906c60: object=_UIStatusBarWifiItem>>";
    "<_UIStatusBarIdentifier: 0x600001903b60: object=_UIStatusBarVoiceControlItem>" = "<_UIStatusBarVoiceControlItem: 0x6000003b1f80: identifier=<_UIStatusBarIdentifier: 0x600001903b60: object=_UIStatusBarVoiceControlItem>>";
    "<_UIStatusBarIdentifier: 0x600001907200: object=_UIStatusBarIndicatorCarPlayItem>" = "<_UIStatusBarIndicatorCarPlayItem: 0x6000003b1d80: identifier=<_UIStatusBarIdentifier: 0x600001907200: object=_UIStatusBarIndicatorCarPlayItem>>";
    "<_UIStatusBarIdentifier: 0x600001903be0: object=_UIStatusBarPillBackgroundActivityItem>" = "<_UIStatusBarPillBackgroundActivityItem: 0x600003c4d560: identifier=<_UIStatusBarIdentifier: 0x600001903be0: object=_UIStatusBarPillBackgroundActivityItem>>";
    "<_UIStatusBarIdentifier: 0x600001906b20: object=_UIStatusBarCellularExpandedItem>" = "<_UIStatusBarCellularExpandedItem: 0x60000366c280: identifier=<_UIStatusBarIdentifier: 0x600001906b20: object=_UIStatusBarCellularExpandedItem>>";
    "<_UIStatusBarIdentifier: 0x600001907280: object=_UIStatusBarIndicatorStudentItem>" = "<_UIStatusBarIndicatorStudentItem: 0x6000003b1dc0: identifier=<_UIStatusBarIdentifier: 0x600001907280: object=_UIStatusBarIndicatorStudentItem>>";
}

```
```
(lldb) po [statusBar performSelector:@selector(containerView)]
<_UIStatusBarForegroundView: 0x7fee31804f30; frame = (0 0; 375 44); layer = <CALayer: 0x600001907720>>
```

```
(lldb) po [statusBar performSelector:@selector(setForegroundColor:) withObject:[UIColor whiteColor]]
UICachedDeviceWhiteColor
```

```
(lldb) po [statusBar performSelector:@selector(styleAttributes)]
<_UIStatusBarStyleAttributes: 0x60000332fd40: style=-6485417468062910266, mode=-6485417468062910314, traitCollection=<UITraitCollection: 0x600002db61c0; UserInterfaceIdiom = Phone, DisplayScale = 2, DisplayGamut = P3, HorizontalSizeClass = Compact, VerticalSizeClass = Compact, UserInterfaceStyle = Light, UserInterfaceLayoutDirection = LTR, ForceTouchCapability = Unavailable, PreferredContentSizeCategory = L, AccessibilityContrast = Normal, UserInterfaceLevel = Base>, effectiveLayoutDirection=0, iconScale=1, symbolScale=1, textColor=UIExtendedGrayColorSpace 1 1, imageTintColor=UIExtendedGrayColorSpace 1 1, imageDimmedTintColor=UIExtendedGrayColorSpace 1 0.2, imageNamePrefixes=(
    "Split_",
    "Black_",
    "Item_"
)>
```
