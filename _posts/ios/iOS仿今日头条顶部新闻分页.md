---
layout: post
title: iOS仿今日头条顶部新闻分页
date: 2017-11-26
tags: iOS   
---

近日闲来无事总是刷头条,突然发现了一个有趣的现象,如下图:

![](https://upload-images.jianshu.io/upload_images/1978245-822c05c52104ed4d.gif?imageMogr2/auto-orient/strip)
****
当你滑动或者点击分页的名字的时候,不管当时那个分页在哪,最后都会被滚动到最中间.我又去翻了其他的资讯类的app,发现基本很多都是这样做的.抱着求知的心态,自己也搞一个类似的新闻分页,分析一下其中的原理.

本文的目录结构:


[TOC]



### 一.确定需求

我们来分解以下我们的需求,新闻分页一般包括两个部分,一个头部的滚动部分,一个底部的内容显示部分;

给张图更清晰:

![](https://img-blog.csdn.net/20180330120119579?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4MzcyMzQ3/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)


1. 分页部分要做的效果就是点击标题5的时候,标题5要滚动标题三的位置.
2. 为了方便以后可以复用,我们需要将两个部分的代码隔离起来,独立成两个类
3. 添加相关的控制代码,当不需要将标题居中的时候,也可以达到不居中的效果.
4. 顶部标题最好是一个UIView视图,便于我们去自定义每个TAB的样子.

以上就是大概的设计和需求了,接下来就开始了我们最喜欢的动手环节了!!

### 二.代码架构

#### 2.1 顶部分页设计

滚动视图在ios中有很多种,UIScrollView,UICollectionView和UITableView都可以达到要求,这里我们采用UIScrollView来制作我们的顶部滚动区域.

至于为啥要选择UIScrollView来作为滚动视图,我这说明一下:

1. 根据需求,我们如果要将制定的标题滚动到中间,就需要计算到精确的偏移量,而UIScrollView刚好就可以帮我们实现到这点,当然UICollectionView和UITableView也是可以精确到的,因为他们继承了UIScrollView.但是能直接使用UIScrollView为啥我们要使用一个继承的备胎呢!!
2. UICollectionView和UITableView的机制就是系统会回收屏幕以外区域的item以便于减少资源消耗.但是在这里我们有时需要计算屏幕以外的tab偏移量,显然他们是无法或者不方便做到的.

##### 2.1.1 当我们需要顶部选中标题自动居中的情况

创建一个名叫TabScrollview的类集成UIScrollView,定义几个我们需要的关键属性




```

/**
 装载视图的数组
 */
@property (nonatomic,strong)NSArray<UIView*> *viewArr;


/**
 tab的宽度
 */
@property (nonatomic,assign)NSInteger tabWidth;

/**
 tab的高度
 */
@property (nonatomic,assign)NSInteger tabHeight;

/**
 tab下方标记线
 */
@property (nonatomic,strong)UIView *tagLine;
/**
 滚动的方向
 */
@property DirectionStyle direction;
/**
 记录位置下标
 */
@property (nonatomic,assign)NSInteger tagIndex;
/**
 tab下方标记线
 */
@property (nonatomic,strong)TabClickBlock clickBlock ;

```


viewArr是装载了所有头部tab视图(每栏的视图)的数组,假如你头部标题只想显示文字,你可以直接装载uiLabel.定义tab的宽度高度,以及一根标记线,方便我们标记滚动到哪个标题了,假如你不要标记线,只需要将标记线的颜色透明即可,
block 是为了回调我们选中的tab的下标,方便底部视图的切换.

```
#define tagLineheight 2 //默认标记线高度
#define tagLineColor [UIColor redColor]  //默认标记颜色
#define defTag 0  //默认标记0号位
#define openAutoCorrection true  //默认开启选中标题自动居中功能
typedef void(^TabClickBlock)(NSInteger index);
typedef NS_ENUM(NSInteger, DirectionStyle) {
    horizontal = 0,
    vertical = 1
};
```
然后定义一些基本的常量,比如标记线高度,颜色,默认的标题显示的小标,滚动的方向等等,还有一个重要的常量openAutoCorrection 就是我们设置的是否要将选中的标题居中的开关,不需要的时候直接这里设置false即可

另外我们要需要一个供外部调用的配置接口,该接口承担着大部分的赋值操作

```
-(void)configParameter:(DirectionStyle)directionStyle viewArr:(NSArray<UIView*>*)viewArr tabWidth:(NSInteger)tabWidth tabHeight:(NSInteger)tabHeight index:(NSInteger)index block:(TabClickBlock) clickBlock{
    _viewArr=viewArr;
    _tabHeight=tabHeight;
    _tabWidth=tabWidth;
    _direction=directionStyle;
    _clickBlock=clickBlock;
    if(_direction==horizontal){
        
        [self updateTag:index];
        [_viewArr enumerateObjectsUsingBlock:^(UIView * _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
            obj.frame=CGRectMake(idx*_tabWidth, 0, _tabWidth, _tabHeight);
            [self addSubview:obj];
            [self setListener:obj index:idx];
        }];
        
        //添加标记线
        [self addSubview:_tagLine];
        self.contentSize=CGSizeMake(_tabWidth*_viewArr.count, 0);
        
    }else{
        [self updateTag:index];
        
        [_viewArr enumerateObjectsUsingBlock:^(UIView * _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
            obj.frame=CGRectMake(0, idx*_tabHeight, _tabWidth, _tabHeight);
            [self addSubview:obj];
            [self setListener:obj index:idx];
            
        }];
        [self addSubview:_tagLine];
        
        self.contentSize=CGSizeMake(0, _tabHeight*_viewArr.count);
    }
  
}
```

接下来是设计标题的居中逻辑.这个部分作者想了挺久的(可能有点笨).先看下代码(由于我设计了水平和垂直两个方向,这里我直接介绍水平滚动):

```
 //获取scrollview宽度
        NSInteger maxWidth=self.frame.size.width;
        //获取当前点击的tab所处的位置大小
        CGFloat maxW=_tabWidth*index;
        //判断tab是否处于大于屏幕一半的位置,并计算出偏移量
        CGFloat offset_halfmaxWidth=maxW-maxWidth/2;
        //当tab偏移量不足tab宽度时,计算出最小的偏移量
        CGFloat itemOffset=offset_halfmaxWidth+_tabWidth/2;
        
        //当偏移量>0的时候,
        if(offset_halfmaxWidth>0){
            //假如偏移量小于一个tab的宽度,说明还没有到最初始位置,可以执行偏移
            if(offset_halfmaxWidth<_tabWidth){
                [self setContentOffset:CGPointMake(itemOffset, 0)];
                return;
            }
            NSInteger maxCont=_tabWidth*_viewArr.count;
            //获取偏移的页数,减1的作用是我们的偏移是从0开始的,所以需要减去一个屏幕长度
            NSInteger remainder_x=maxCont/maxWidth-1;
            //获取最后一页的偏移量
            NSInteger remainder_=maxCont%maxWidth;
            
            //获取到最大偏移量
            NSInteger maxOffset=remainder_x*maxWidth+remainder_;
            
            
            //假如我们的计算的偏移量小于最大偏移,说明是可以偏移的
            if(itemOffset<=maxOffset){
                //假如偏移量大于一个tab的宽度,判断
                if(itemOffset<=_tabWidth){  //当点击的偏移量小于tab的宽度的时候,归零偏移量
                    [self setContentOffset:CGPointMake(0, 0)];
                    return;
                }else{
                    [self setContentOffset:CGPointMake(itemOffset, 0)];
                }
                
            }else{
                [self setContentOffset:CGPointMake(maxOffset, 0)];
            }
        }else if(offset_halfmaxWidth<0){
            //判断往后滚的偏移量小于0但是却和半个tab宽度之和要大于0的时候,说明还可以进行微调滚动,
            if(itemOffset>0){
                [self setContentOffset:CGPointMake(itemOffset, 0)];
                return;
            }
            //最小偏移小于0,说明往前滚,将偏移重置为初始位置
            [self setContentOffset:CGPointMake(0, 0)];
        }else{
            [self setContentOffset:CGPointMake(0, 0)];
        }
```

大概思路就是,从右往左滚动的时候,当我们点击的tab标题的位置大于一个屏幕的宽度的时候,计算他和屏幕中线的偏移量,然后加上tab标题的半个宽度.就是我们scrollview需要滚动的偏移量.当我们滚动到不能再滚动的这个临界值得时候,就让其自动滚动.反之我们从左往右滚动的逻辑也是一样的.

这里面有几个难点,

1.

第一个就是从右往左滚动这个临界值.如图:

![](https://img-blog.csdn.net/20180330133705583?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4MzcyMzQ3/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

tab19已经是最后一个tab了,显然我们点击tab19的时候我们不能让tab19居中,因为这样的话右边就会空出一块没有内容的区域.

2.

第二个就是从左往右滚动的这个零界值:

![](https://img-blog.csdn.net/20180330133849425?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4MzcyMzQ3/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

tab0是第一个tab,我们显然也不能让其居中.甚至tab1,tab2都不能.否者就会超出scrollview的滚动范围

3.

第三个就是需要将选中标题居中的区域计算:

![](https://img-blog.csdn.net/20180330134402672?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4MzcyMzQ3/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

我们需要将这个范围内的选中标题全部居中,当你点击tab7的时候,tab7就会居中,同样点击tab8的时候,tab8也会居中,直到到达上面的两个灵界值为止.

##### 2.1.2 我们不要自动将选中标题居中的情况

这个是我们最经常使用的设计,原本我觉得这个设计会很简单,但是在设计的时候还是遇到点小破折.核心代码:

```
 //获取scrollview宽度
        NSInteger maxWidth=self.frame.size.width;
        CGFloat currOffset=_tabWidth*index;
        
        //获取scrollview移动了的距离
        CGFloat pointx= self.contentOffset.x;
        if(_tagIndex<index){ //往后滚
            
            NSInteger equal_value=maxWidth%_tabWidth;
            if(equal_value==0){ //假如tab宽度等分屏幕,说明屏幕右边一定能完全显示一个tab
                //直接计算一个tab宽度偏移
                if(currOffset>=maxWidth){
                    //偏移一个tab长度
                    [self setContentOffset:CGPointMake(pointx+_tabWidth, 0)];
                }
            }else{ //tab宽度不等分屏幕,说明屏幕右边肯定有一个tab显示不全
                //显示不全的时候,我们需要将不全的部分偏移也计算进去
                
                if((currOffset+_tabWidth)>=maxWidth){
                    //偏移一个tab长度
                    [self setContentOffset:CGPointMake(pointx+_tabWidth, 0)];
                }
                
            }
      
        }else{ //往前滚
            NSLog(@"移动了的距离---%f---当前tag--%f",pointx,currOffset);
            
            if(currOffset==0){//假如回滚到第一格,初始化偏移量
                [self setContentOffset:CGPointMake(0, 0)];
                return;
            }
            if(currOffset<pointx){
                //往后回滚一格
                [self setContentOffset:CGPointMake((pointx-_tabWidth), 0)];
            }
            
        }
```

由于我们没有使用选中居中的功能,当底部内容视图滚动的时候,我们顶部视图也要跟着滚动.于是可能会出现以下情况:

1.注意这里的tab5可不是最后一个tab哦,当我们继续滚动底部视图的时候,我们肯定是希望tab6会显示出来,代替tab5的位置

![](https://img-blog.csdn.net/20180330135145448?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4MzcyMzQ3/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)



2.往回滚的时候,我们也需要将tab10显示出来代替tab11的位置

![](https://img-blog.csdn.net/2018033013543420?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE4MzcyMzQ3/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)




#### 2.2 底部视图设计

由于我们同样需要滑动操作,uiscrollview和表格视图都可以帮我们实现,但是这里我使用UIPageViewController来实现. UIPageViewController是一个滑动页面控制器,广告轮播,启动页都可以使用该类来实现.

为什么不使用uiscrollview或者uicollectionview的原因有以下几点:

1. Uicollectionview 由于复用的特性,会将屏幕外的cell销毁,假如有3页的时候,我们滚动到底三页时,第一页就会被回收了,这显然不是我希望的,我希望的是创建了的页面就就保存起来,这样当往回滑动的时候就不用重新创建新的页面对象了.
2. uiscrollview 在创建视图的时候,就已经将所有的页面添加到了uiscrollview中,当页面过多的时候,可能会引起相关的渲染内存问题.
3. UIPageViewController的好处刚好弥补了Uicollectionview和uiscrollview的的缺点,他同一时间只会加载一个page在界面上,减少了绘制问题,且他不存在页面回收问题.

2.2.1 代码设计

创建一个TabContentView的类,集成UIView.

以下是.h文件代码:

```
typedef void(^TabSwitchBlcok)(NSInteger index);
@interface TabContentView : UIView<UIPageViewControllerDelegate,UIPageViewControllerDataSource>


/**
 page
 */
@property (nonatomic,strong)UIPageViewController *pageController;

/**
 内容页数组
 */
@property (nonatomic,strong)NSArray<UIViewController*> *controllers;
/**
 内容页数组
 */
@property (nonatomic,strong)TabSwitchBlcok tabSwitch;

//对外切换内容的接口
-(void)updateTab:(NSInteger)index;

//配置接口
-(void)configParam:(NSMutableArray<UIViewController*>*)controllers Index:(NSInteger)index block:(TabSwitchBlcok) tabSwitch;
@end
```

我们定义了一个block来回调滚动到的下标值, controllers是我们需要显示的所有内容页面集合.
configParam 是外部调用的配置方法,用来配置一些数值.

.m文件:

```
#import "TabContentView.h"

@implementation TabContentView
- (instancetype)initWithFrame:(CGRect)frame{
    
    self =[super initWithFrame:frame];
    if(self){
        [self initView];
    }
    return self;
}


-(void)initView{
    _pageController=[[UIPageViewController alloc]initWithTransitionStyle:UIPageViewControllerTransitionStyleScroll navigationOrientation:UIPageViewControllerNavigationOrientationHorizontal options:nil];
    _pageController.delegate = self;
    _pageController.dataSource = self;
    _pageController.view.frame=self.bounds;
    [self addSubview:_pageController.view];
}

-(void)configParam:(NSMutableArray<UIViewController *> *)controllers Index:(NSInteger)index block:(TabSwitchBlcok)tabSwitch{
    
    _tabSwitch=tabSwitch;
    _controllers=controllers;
    _tabSwitch=tabSwitch;
    //默认展示的第一个页面
    [_pageController setViewControllers:[NSArray arrayWithObject:[self pageControllerAtIndex:index]] direction:UIPageViewControllerNavigationDirectionReverse animated:YES completion:nil];
}

-(void)updateTab:(NSInteger)index{
    NSLog(@"updateTab---%lu",index);
    //默认展示的第一个页面
    [_pageController setViewControllers:[NSArray arrayWithObject:[self pageControllerAtIndex:index]] direction:UIPageViewControllerNavigationDirectionReverse animated:YES completion:nil];
    
}




-(void)layoutSubviews{
    [super layoutSubviews];
    _pageController.view.frame=self.bounds;
}

//返回下一个页面
-(UIViewController *)pageViewController:(UIPageViewController *)pageViewController viewControllerAfterViewController:(UIViewController *)viewController{
    NSInteger index= [_controllers indexOfObject:viewController];
    if(index==(_controllers.count-1)){
        return nil;
    }
    index++;
    return [self pageControllerAtIndex:index];
}
//返回前一个页面
-(UIViewController *)pageViewController:(UIPageViewController *)pageViewController viewControllerBeforeViewController:(UIViewController *)viewController{
    //判断当前这个页面是第几个页面
    NSInteger index=[_controllers indexOfObject:viewController];
    //如果是第一个页面
    if(index==0){
        return nil;
    }
    index--;
    return [self pageControllerAtIndex:index];
    
}

//根据tag值创建内容页面
-(UIViewController*)pageControllerAtIndex:(NSInteger)index{
    
    return [_controllers objectAtIndex:index];
    
}
//结束滑动的时候触发
-(void)pageViewController:(UIPageViewController *)pageViewController didFinishAnimating:(BOOL)finished previousViewControllers:(NSArray<UIViewController *> *)previousViewControllers transitionCompleted:(BOOL)completed{
    NSLog(@"didFinishAnimating");
    NSInteger index=[_controllers indexOfObject:pageViewController.viewControllers[0]];
    _tabSwitch(index);
}
//开始滑动的时候触发
-(void)pageViewController:(UIPageViewController *)pageViewController willTransitionToViewControllers:(NSArray<UIViewController *> *)pendingViewControllers{
    NSLog(@"pageViewController");

}
```

底部内容设计的代码不多,主要是UIPageViewController的初始化和相关代理资源方法的实现,都是苦力活,无技巧可言.唯一需要注意的是对外一个切换内容的接口.


显示的每一页内容的controller,我们需要用一下懒加载,很简单,设置一个bool值,页面显示在屏幕上的时候就加载,再次进入的该页面的时候不加载,除非手动触发.

```
-(void)viewWillAppear:(BOOL)animated{
    
    if(!_isFrist){
        //第一次进入,自动加载数据
        _isFrist=true;
    }else{
        NSLog(@"第二次进入--%@",_tag);

    }
    
}
```

以上我们就完成了所有的代码逻辑设计,下面开始看看效果.

#### 2.3 两个视图的使用

```
 _tabs=[[NSMutableArray alloc]initWithCapacity:20];
    _contents=[[NSMutableArray alloc]initWithCapacity:20];
    
    for(int i=0;i<20;i++){
        NSString *titleStr=[NSString stringWithFormat:@"tab%i",i];

        UILabel *tab=[[UILabel alloc]init];
        tab.textAlignment=NSTextAlignmentCenter;
        tab.text=titleStr;
        tab.textColor=[UIColor blackColor];
        

        [_tabs addObject:tab];
        
        int R = (arc4random() % 256) ;
        int G = (arc4random() % 256) ;
        int B = (arc4random() % 256) ;
        
        
        PageViewController *con=[PageViewController new];
        con.view.backgroundColor=[UIColor colorWithRed:R/255.0 green:G/255.0 blue:B/255.0 alpha :1];
        con.tag=titleStr;
        [_contents addObject:con];
        
    }
    
    
    
    //
    
    _tabScrollView=[[TabScrollview alloc]initWithFrame:CGRectZero];
    [self.view addSubview:_tabScrollView];
    [_tabScrollView mas_makeConstraints:^(MASConstraintMaker *make) {
        make.height.equalTo(@50);
        make.left.equalTo(self.view);
        make.right.equalTo(self.view);
        make.top.equalTo(self.view);
    }];

    [_tabScrollView configParameter:horizontal viewArr:_tabs tabWidth:60 tabHeight:50 index:0 block:^(NSInteger index) {

        [_tabContent updateTab:index];
    }];
    
    
    _tabContent=[[TabContentView alloc]initWithFrame:CGRectZero];
    [self.view addSubview:_tabContent];

    [_tabContent mas_makeConstraints:^(MASConstraintMaker *make) {
        make.left.equalTo(self.view);
        make.right.equalTo(self.view);
        make.top.equalTo(_tabScrollView.mas_bottom);
        make.bottom.equalTo(self.view);
    }];
    
    
    [_tabContent configParam:_contents Index:0 block:^(NSInteger index) {
        [_tabScrollView updateTagLine:index];
    }];
    
```
使用起来很简单,我们只要初始化两个视图,然后通过配置接口配置相关参数,通过回调的block接收下标值即可.


### 三.效果演示

1.首先我们展示一下和今日头条一样的标题居中功能.(设置10个标题)

![](https://upload-images.jianshu.io/upload_images/1978245-3a30bc794f2dbf38.gif?imageMogr2/auto-orient/strip)





2.关闭标题居中功能(设置10个标题)

![普通.gif](https://upload-images.jianshu.io/upload_images/1978245-2d7d96c1706fe125.gif?imageMogr2/auto-orient/strip)


代码已上传github----[AlTabScrollview](:https://github.com/momoxiaoming/AlTabScrollview)














