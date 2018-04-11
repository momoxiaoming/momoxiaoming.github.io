# 利用UIPageViewController实现图片轮播(简单实用版本)

#### tips

今天项目中用到了轮播的功能,本来想着去网上找个第三方的,后来想一下自己实现一个也是挺简单的.于是就有了这篇文章

### 一. 需求分析

总结了一下,一个广告轮播必须含有以下几个功能;

1. 页面内容可以自定义;比如说可以是图片轮播,也可以是其他view
2. 可以实现定时轮播;
3. 能处理轮播页的点击事件;
4. 滑动到最后一页的时候再往下滑动可以到第一页,第一页再往回滑动可以到最后一页
5. 最好能封装成一个view,便于日后的使用

确定好要实现的功能,又到了我们最开心的编码环节.

### 二. 需求设计

本来我第一时间想到的是利用UiScrollview或者UiCollectionView等组件来实现,不过在以前一篇[仿今日头条新闻分页中](https://www.jianshu.com/p/4ecaf068df05)文章中我讲过一下他们和UIPageViewController的区别.所以今天我就选定UIPageViewController来当我们这次的主角啦!



##### 代码实现

我们新建一个BannerView的自定义view,所有的逻辑代码就写里面啦.
##### 1.首先我们确定一下需要的一些变量属性,具体详情看注释就明白了(小编代码注释写的贼多)

```
@property (nonatomic,strong)UIPageViewController *pageCon;
/**
 指示器
 */
@property (nonatomic,strong)UIPageControl *indicator;
/**
 存放所有的图片地址
 */
@property (nonatomic,strong)NSArray *imageArr;

/**
 存放所有的页面
 */
@property (nonatomic,strong)NSMutableArray *controlls;
/**
 当前tag值
 */
@property (nonatomic,assign)NSInteger tagIndex;
/**
 轮播定时器
 */
@property (nonatomic,strong)NSTimer *timeZ;
/**
 回调点击的图片所在的tag值
 */
@property (nonatomic,strong)bannerResult block;
/**
 是否在自动轮播
 */
@property (nonatomic,assign)BOOL isAuto;
```
##### 2. 初始化我们需要的相关view和一下相关变量

```
-(instancetype)initWithFrame:(CGRect)frame{
    self=[super initWithFrame:frame];
    if(self){
        [self initView];
        
    }
    return self;
    
}

-(void)initView{
    _controlls=[[NSMutableArray alloc]init];
    _tagIndex=0;//默认tag==0
    _pageCon=[[UIPageViewController alloc]initWithTransitionStyle:UIPageViewControllerTransitionStyleScroll navigationOrientation:UIPageViewControllerNavigationOrientationHorizontal options:nil];
    _pageCon.delegate = self;
    _pageCon.dataSource = self;
    _pageCon.view.frame=self.bounds;
    [self addSubview:_pageCon.view];
    
    _indicator=[[UIPageControl alloc]init];

    [self addSubview:_indicator];

	//这里用了Masonry框架代码,就是底部居中的意思,不想用该框架的可以用其他的约束
    [_indicator mas_makeConstraints:^(MASConstraintMaker *make) {
        make.centerX.equalTo(self);
        make.bottom.equalTo(self.mas_bottom).offset(10);
    }];
}
```

##### 3. 初始化一些页面数据

主要是创建轮播页以及轮播页面里面要添加的内容(这里我们只要放一个图片即可)

```
-(void)initData:(NSArray *)arr block:(bannerResult)block{
    _isAuto=false;
    _block=block;
    _imageArr=arr;
    _indicator.numberOfPages=_imageArr.count;

    [_imageArr enumerateObjectsUsingBlock:^(NSString* obj, NSUInteger idx, BOOL * _Nonnull stop) {
    	//创建轮播页
        UIViewController *con=[[UIViewController alloc]init];
        con.view.frame=self.bounds;
        UIImageView *image=[[UIImageView alloc]initWithFrame:self.bounds];
        image.contentMode=UIViewContentModeScaleAspectFill;
         [image sd_setImageWithURL:[NSURL URLWithString:obj]];
        [con.view addSubview:image];
        [_controlls addObject:con];
        
        [self setListener:con.view index:idx];  //这是设置每个页面点击事件的方法,
    }];
    
     [_pageCon setViewControllers:[NSArray arrayWithObject:[self pageControllerAtIndex:_tagIndex]] direction:UIPageViewControllerNavigationDirectionReverse animated:YES completion:nil];
    
}
```


##### 4. 添加UIPageViewController代理

这里我们要注意两点:
1.开始滑动和结束滑动时的事件,当开始滑动的时候,我们如果设置了自动轮播,就要先停止轮播,优先响应滑动事件,防止出现滑动和轮播冲突问题.
2.当滑到最后一页的时候,我们的代理方法中要给出第0页当成下一页,当滑到第一页再往回滑动的时候,我们要将最后一页当成上一页,这样就做到了循环滑动了.

```
//返回下一个页面
-(UIViewController *)pageViewController:(UIPageViewController *)pageViewController viewControllerAfterViewController:(UIViewController *)viewController{
    
    NSInteger index= [_controlls indexOfObject:viewController];
    NSLog(@"viewControllerAfterViewController-->%lu",index);

    if(index==(_imageArr.count-1)){
        index=0;
    }else{
        index++;

    }
    return [self pageControllerAtIndex:index];
}
//返回前一个页面
-(UIViewController *)pageViewController:(UIPageViewController *)pageViewController viewControllerBeforeViewController:(UIViewController *)viewController{
    //判断当前这个页面是第几个页面
    NSInteger index=[_controlls indexOfObject:viewController];
    NSLog(@"viewControllerBeforeViewController-->%lu",index);
    //如果是第一个页面
    if(index==0){
        index=_imageArr.count-1;
        
    }else{
        index--;

    }
    return [self pageControllerAtIndex:index];
    
}

//根据tag取出内容页面
-(UIViewController*)pageControllerAtIndex:(NSInteger)index{
    if(_controlls!=nil&&_controlls.count!=0){
        UIViewController *con=_controlls[index];
        return con;
    }
    return nil;
}
//结束滑动的时候触发
-(void)pageViewController:(UIPageViewController *)pageViewController didFinishAnimating:(BOOL)finished previousViewControllers:(NSArray<UIViewController *> *)previousViewControllers transitionCompleted:(BOOL)completed{
    
    NSInteger index=[_controlls indexOfObject:pageViewController.viewControllers[0]];
     _tagIndex=index;
    [_indicator setCurrentPage:_tagIndex];
    if(isAuto){//判断轮播是否开启,如果已开启,重新启动定时器
        [self openAuto];

    }
}
//开始滑动的时候触发
-(void)pageViewController:(UIPageViewController *)pageViewController willTransitionToViewControllers:(NSArray<UIViewController *> *)pendingViewControllers{
    [self closeAuto];
}




```

##### 5. 定时器的方法

为了方便对外部调用,提供了开启和关闭定时器的两个方法

```
//开启定时器
-(void)openAuto{
    
    _isAuto=true;
    
    //开启自动轮播
    ALWk(weakSelf);
    _timeZ=[NSTimer scheduledTimerWithTimeInterval:5 repeats:YES block:^(NSTimer * _Nonnull timer) {
        NSLog(@"定时切换--%lu",_tagIndex);
        weakSelf.tagIndex++;
        if(weakSelf.tagIndex>(weakSelf.imageArr.count-1)){
            weakSelf.tagIndex=0;
        }
        
        [_indicator setCurrentPage:weakSelf.tagIndex];
        [weakSelf.pageCon setViewControllers:[NSArray arrayWithObject:[weakSelf pageControllerAtIndex:weakSelf.tagIndex]] direction:UIPageViewControllerNavigationDirectionReverse animated:YES completion:nil];
      
    }];
}
//关闭定时器
-(void)closeAuto{
    if(_timeZ){
        _isAuto=false;

        [_timeZ invalidate];
        _timeZ=nil;
    }
    
}
```

至此,我们的核心代码就写完了,接下来我们只需要随便在个控制器中引入bannerView即可

### 三. 测试效果

先来张效果图吧

![](https://user-gold-cdn.xitu.io/2018/4/11/162b4e30ba855a6d?w=360&h=240&f=gif&s=1266771)


代码已上传github-->[Banner](https://github.com/momoxiaoming/Banner)

