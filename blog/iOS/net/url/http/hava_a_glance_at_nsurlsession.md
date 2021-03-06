#NSURLSession第一篇:初窥NSURLSession
##1.通过GET请求天气数据 
先来看一个通过NSURLSession获取天气信息的例子：

	@interface ViewController ()
	@property (strong, nonatomic) NSURLSession *defaultSession;
	@property (strong, nonatomic) NSURLSessionDataTask *weatherTask;
	@end
	
	@implementation ViewController
	
	- (void)viewDidLoad {
	    [super viewDidLoad];
	    // Do any additional setup after loading the view, typically from a nib.
	    
	    // Example: Get Weather Infomation
	    NSURLSessionConfiguration *defaultConfigObject = [NSURLSessionConfiguration defaultSessionConfiguration];
	    self.defaultSession = [NSURLSession sessionWithConfiguration: defaultConfigObject delegate: nil delegateQueue: [NSOperationQueue mainQueue]];
	    NSURL *weaherURL = [NSURL URLWithString:@"http://www.weather.com.cn/adat/cityinfo/101280601.html"];
	    NSURLRequest *weatherReq = [NSURLRequest requestWithURL:weaherURL];
	    self.weatherTask = [self.defaultSession dataTaskWithRequest:weatherReq completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error) {
	        if (error != nil ) {
	            NSLog(@"http response with error %@", error);
	            return ;
	        }
	        
	        NSError *jsonError;
	        NSDictionary *weatherData = [NSJSONSerialization JSONObjectWithData:data options:NSJSONReadingAllowFragments error:&jsonError];
	        if (jsonError != nil ) {
	            NSLog(@"parse json with error %@", jsonError);
	            return;
	        }
	        NSDictionary *weatherInfo = [weatherData objectForKey:@"weatherinfo"];
	        NSString *htemp = [weatherInfo objectForKey:@"temp1"];
	        NSString *ltemp = [weatherInfo objectForKey:@"temp2"];
	        NSString *weather = [weatherInfo objectForKey:@"weather"];
	        NSString *city = [weatherInfo objectForKey:@"city"];
	        NSLog(@"[%@]:天气:%@  最高温：%@ 最低温：%@", city, weather, htemp, ltemp);
	        
	    }];
	    [self.weatherTask resume];
	}
这里通过向“http://www.weather.com.cn/adat/cityinfo/101280601.html”发送GET请求，获取到深圳的天气信息。获取的结果是一段JSON数据,解析得到深圳的天气、温度等信息。运行结果：

	[深圳]:天气: 多云  最高温：27℃ 最低温：21℃
	
关于国内获得天气的接口，知乎上面有个[有哪些免费开放且收录城市较完整的天气 API 接口？](https://www.zhihu.com/question/20521716)的总结。

上面在使用NSURLSession的时候，可以归纳出几个步骤：

1. 首先创建一个NSURLSessionConfiguration， 这里使用了默认的configureation
2. 用上面的configureation创建一个NSURLSession对象，这里使用了系统默认的回调处理（nil）并让其在mainOperationQueue中被调度执行
3. 创建一个NSURLRequest，并设置相关参数， 这里用NSURL创建了一个默认为GET的请求
4. 用上面的session和request创建一个Task，这里创建的是NSURLSessionDataTask用于一般的HTTP API请求（想象下Ajax）。
5. 最后调用task的resume方法，开始task的调度

这里执行了resume后，并非阻塞的，会立即返回，task会在operationQueue中调度。因此这里的task和session就不能是局部变量，否则会被ARC给干掉。

##2. NSURLSession于NSURLConnection

在NSURLSession之前，处理HTTP请求我们一般都是用的NSURLConnection，甚至AFNetwork最开始（AFNetwork2.0也是，但可选的用NSURLSession）就是用的NSURLConnection。NSURLConnection是Apple自2003和Safari一同发布的。时隔十年，在WWDC2013上Apple发布了iOS7和NSURLSession，作为N S U RLconnection的替代者。再现今AppStore上QQ/微信都是要求iOS7以上的时代，二者用哪个，想必不用过多解释了。objc上面有篇过度的文章[From NSURLConnection to NSURLSession](https://www.objc.io/issues/5-ios7/from-nsurlconnection-to-nsurlsession/)作者是大名鼎鼎的[Mattt](https://github.com/mattt)AFNetwork的主要作者（所以AFNetwork3.0直接就是用的NSURLSession），Ono（我的另一篇[使用Ono读取XML文件](http://www.jianshu.com/p/92d8d109bfc8)有介绍）的作者。

和NSURLConnection想比，NSURLSession最大的优点主要有：

1. 可以指定Session/Cache缓存	
2. 鉴权证书策略（ssl/tsl）的支持
3. 跳转的友好支持
4. Cookie的管理
5. 协议支持，如http/https/ftp/file
6. operationQueue的支持，以及后台运行 

其中session/cache通过configuration实现；后台运行通过不同的task来区别；跳转和证书通过session的回调进行处理。


##3. 三种Configuration
session通过指定其configuration来配置其类型，主要有三种类型
* 全局session，通过`[NSURLSessionConfiguration defaultSessionConfiguration]`获得。该类型的session中的cache/证书等数据都存储在全局的一个公共位置。
* 临时session，通过`[NSURLSessionConfiguration ephemeralSessionConfiguration]`获得。改类型的session中的cache/证书等数据与其他session中的是独立的，并且当session无效时也会跟着消失
* 后台session，通过`[NSURLSessionConfiguration backgroundSessionConfigurationWithIdentifier:(NSString *)identifier]`获得，此时需要给定一个identifier。当程序从后台被通知时，需要根据这个identifier来判断是哪个session的回应。

这么复杂的三种配置，到底何时该用哪个呢？其实简单的规则就是，普通HTTP API请求用全局Session，上传和下载用后台session，都不能满足时用临时session。

##4. 三种Task
配置好了session后，就需要通过该session产生一个个的task来进行HTTP请求了，这里的三种类型的task并不是和上面的配置一一对应的，而是根据我们日常使用进行区分的。这里可以将他们归为两类
* 普通HTTP API请求如Restful API，对应NSURLSessionDataTask。
* 上传/下载耗时类文件请求，分别对应NSURLSessionUploadTask和NSURLSessionDownloadTask。

这里session和task是1对多的关系，可以认为session是task的管理者。通过session创建出一个task，然后调用这个task的resume方法，将task塞入到session的operationQueue中进行调度。

一个task对应一个请求，可以用NSURLRequest来对请求的参数进行设置，如header、请求的Method、请求的Body数据。每个请求在完成时可以通过一个block来执行其完成后的动作，同时如果要对调整、鉴权证书控制等进行操作时，还需要在session中的delegate中进行操作。

所以在简单使用时，只要给task 一个completionHandler即可。其类型为`void (^)(NSData * __nullable data, NSURLResponse * __nullable response, NSError * __nullable error)` 。出错时通过error给出，回应头部信息通过NSURLResponse给出，Body部分则在data里面，一般Restful API返回的JSON数据就在data里面。

##5. 三种Delegate

对应上面的三种Task，分别有个对应到两个Delegate（NSURLSessionDataDelegate/NSURLSessionDownloadDelegate），在task的执行过程中，会根据需要回调到session中注册的回调。除了上面两种delegate，因为HTTP是TCP连接的流模型，因此还有个NSURLSessionStreamDelegate回调表示流数据中的事件，比如对端关闭事件。

图片关系

每个Delegate的意义在后续文章的介绍中再依次介绍。

##6. 后台运行
在WWDC2013的[What's New in Foundation Networking](https://developer.apple.com/videos/play/wwdc2013/705/)的session中有段演示把程序退到后台在回来后，头像图片就下载好的demo演示，对于这样的场景（下载头像、下载图片、上传头像等）就可以采用这样的方式。后台模式需要首先将session配置成backgroundSessionConfigurationWithIdentifier，然后再通过创建NSURLSessionUploadTask或者NSURLSessionDownloadTask来执行相关的动作。