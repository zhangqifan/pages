---
layout: post
title: Photos Framework 使用小记
categories: iOS
tags: iOS Framework
date: 2016-09-22 14:47:00 +8000
---

## 交代背景 ##

总算是打起精神打算自己给自己累积点啥了，搭了这个blog，开始时有时无记点东西。

既然是个半吊子搞开发的，那就从我本职工作作为开始吧。

前两周，iOS 10 正式版发布。前前后后，林林总总出现了好多「手把手教你适配 iOS 10」、「iOS 10 适配整理」等“深度好文”。因为也没有提前去了解相关的资料，只知道新的 OS 整合了 User Notfication Framework 和一些杂七杂八的内容（比如 ATS 之类云云），所以有关适配 iOS 10 的计划也没有提上正式日程。

但，路还是要走的，泥潭还是要踩的。于是乎，在 iOS 10 最后一个 Beta 版本 Release 的时候，我 OTA 更新了。

更新之后，迫不及待打开开发的应用（插一句，公司里只有我和另一个同事负责开发 iOS 版本，也就是小公司的正常人员配置吧）。嗯……看起来还不错，等等，字体大小好像变大了，按钮的文字内容适配也出现了问题，权限一允许就 crash 了。然后，查阅了点资料，更新了约束，大致上就把文本的问题解决了。心想，大版本适配也不就那么回事嘛，我得意了一会，直到我通过 UIAlertController 打开了相册。<!-- more -->

## 认清错误 ##

上节说道，打开相册。

过去版本的项目中，因为需要在相册中进行多选，所以使用了一个框架，QBImagePicker<sup>[1]</sup>。之前版本久远，也没有用 cocoapods 来做依赖库管理，造成的一个问题就是无法及时更上 Github 上作者的更新。

在解决适配的问题之后，才发现这个库的依赖用的还是 2013 年的版本，当时只支持 AssetLibrary ，对于 iCloud 上传的照片，如果没有本地缓存的话，读取本地相册就是为空。也就是说，可能用户在更新 iOS 10 后，使用应用打开资源库，会出现备选相册列表空白的 issue。

好了，是时候更新库文件了。

~~别直接把库文件拖项目里，多用用 cocoapods 吧。~~

## 认识 PhotoKit ##

查看了 Github 上这个照片选择框架的 README ，发现了具体的使用限制，如下：

>Requirements
- Version `>= 3.0.0` : iOS 8 or later (Using PhotoKit)
- Version `< 3.0.0` : iOS 6 or later (Using AssetsLibrary)

~~唉，兼容 iOS 7 的程序员都是折翼的天使~~

果然，使用了 PhotoKit。（有关 PhotoKit 的好处 blablabla）索性，那就边学边换框架吧。

首先，当然是要了解 PhotoKit 是什么能做什么怎么用。简而言之，利用这个框架，可以帮助构建编辑照片的扩展程序；可以管理照片应用下的照片和视频资源，包括 iCloud 图片资源库。

对于 AssetLibrary 来说，它的工作原理就是通过遍历资源库然后取得照片等数据。对于 PhotoKit，文档中提到了「Fetch and Cache」这个手段，这就是整个框架的变革之处。我的理解，前者，就像是某条老街上路边开着的生活用品杂货店，琳琅满目商品摆在展台，实实在在的东西肉眼可见触手可摸，可是一到晚上收摊，摊主就要一个个把东西收回整理，第二天早上再一个个摆上货架。实物之鉴必然不错，可效率低下也是显而易见；后者，好像网上的商城，把一系列的商品快照给顾客展示挑选，待顾客下单购买后，提货人员从仓库把清单上的货物取出并快递。很显然，后者并不需要真实见到某样东西的具体实物，出售者也无需搬空货仓，需要什么取什么，节省资源开销。

认识一下 PhotoKit 的主要成员们吧（以静态图片为例）：

`PhotoLibrary` 这货反映了本地照片库的一个实例，包含了所有本地或者 iCloud 上的照片、视频或者是 Live Photo 的资源对象。它负责管理他们，监听这些资源元数据的变化、新的资源的创建或者验证访问照片的权限。

`PHAssetChangeRequest` 拍了照，通过什么来保存呢？于是用到了这个类，这个东西的实例方法能够新建照片资源、删除或修改资源。不过这些方法作用域有一点限制，以下会说明。

`PHAsset` 这货是不是很眼熟？对啦，跟 `ALAsset` 的意思差不多，也是代表一个资源的对象。不过它所包含的数据类型更丰富，开发者可以调用该资源的创建	/编辑修改时间、拍摄地点、是否被设置为隐藏以及甚至能够判断是否是连拍产生的。我认为它应该是整个框架中最为重要的数据来源。

`PHFetchResult` 既然提到了 `PHAsset` ，那当然要知道它是怎么被获取的吧。PhotoKit 提供了这个类来「Fetch」资源。可以把它看做是一个装着资源对象的数组。

`PHImageManager` 既然能够得到图片资源对象，那么要怎么做才能把它转换成 `UIImage` 呢。PhotoKit 中包含了这个图片管理器。对于静态图片（不包含 Live Photo），这个类的核心实例方法只有两个，两个方法的 Block 中分别回调了 `UIImage` 对象和 `NSData` 的图片数据对象。

`PHImageRequestOptions` 慢着，如果要获取不同质量不同尺寸的 `UIImage` 对象，那么就要呼叫这位仁兄啦。它可以看做是「请求图片对象的配置选项」，初始化后通过修改该实例的 `version` `deliveryMode` `resizeMode` 等来打包成类似一种请求图片的配置文件，作为 `PHImageManager` 中请求图片数据的参数，即可获得质量不等的图像。

`PHImageCachingManager` 这个类是 `PHImageManager` 的子类。如名所见，是用来处理图片缓存的。如果一个 `collection view` 的数据源主要以大量图片形式存在， 那么建议使用这个类来进行缓存，以便在滚动过程中提前加载。也就是给即将加载的图片「预热」。

好啦，大体上所用到的，给图片做处理的类就是以上这几个，接下来就是个人摸索的心路历程啦。

## 修改框架的血泪史 ##

回顾一下起因，由于使用了老框架，导致 iCloud 上照片无法请求到本地，结果调用相册显示空白。

因为 iOS 7 只支持 `AssetLibrary` 的调用方式，所以还是要保留旧框架，打算在 iOS 8 以上的版本应用最新的。不过 Xcode 8 已经不支持 iOS 7 的 SDK，手边也没有合适的测试机，那就把新的框架直接应用在 iOS 10 以上吧。

第〇步，修改 `Podfile` 并更新 Pod :
```
source 'https://github.com/CocoaPods/Specs.git'
platform: ios, '7.0'

target 'projName' do
	pod 'QBImagePickerController'
	...
end
```
如果直接这样安装/更新，那么在编译项目时，会出现文件重复的问题。所以，为了保留旧版本，我手动的把最新的框架文件 download 下来修改了类名加入了项目。

配置了一个全局的 `Configure.h` ，负责全局导入头文件。
```
#import "QBImagePickerController.h"
#import "QBPhotoKitImagePickerController.h"
```

第一步，让我们把改过名的框架加入到某个需要拍照和打开相册的 ViewController 中：
```
if (SDKVersion10){
    QBPhotoKitImagePickerController *imagePickerWithPhotoKit = [QBPhotoKitImagePickerController new];
    [imagePickerWithPhotoKit.selectedAssets removeAllObjects];
    [imagePickerWithPhotoKit.selectedAssets addObjectsFromArray:[GlobalFunction readArrayWithCustomObjFromUserDefaults:@"postImages"]];
    imagePickerWithPhotoKit.allowsMultipleSelection = YES;
    imagePickerWithPhotoKit.maximumNumberOfSelection = 9;
    imagePickerWithPhotoKit.delegate = self;
    imagePickerWithPhotoKit.mediaType = QBImagePickerMediaTypeImage;
} else {
    // 旧框架初始化
}
```
解释一下，主要的图片数据以 `PHAsset` 类型保存在 `selectedAssets` 这个序列集合中。

第四行的 `[GlobalFunction readArrayWithCustomObjFromUserDefaults:@"postImages"]` 目的是将图片以 `NSArray<NSData *>` 的格式从本地读取：
```objc
NSUserDefaults *defaults = [NSUserDefaults standardUserDefaults];
NSData *data = [NSKeyedArchiver archivedDataWithRootObject:myArray];
[defaults setObject:data forKey:keyName];
[defaults synchronize];
```
所以，归档时是以什么数据结构保存的至关重要。

接着，我们来实现 `QBPhotoKitImagePickerController` 提供的代理方法，该方法在确定照片选择后调用：
```objc
- (void)qb_imagePickerController:(QBPhotoKitImagePickerController *)imagePickerController didFinishPickingAssets:(NSArray *)assets {
    NSMutableArray *assetsArray = [NSMutableArray arrayWithCapacity:0];
    PHImageManager *imageManager = [PHImageManager defaultManager];
    [imagePickerController.selectedAssets enumerateObjectsUsingBlock:^(id  _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
        PHAsset *pObj = obj;
        PHImageRequestOptions *options = [PHImageRequestOptions new];
        options.synchronous = YES;
        [imageManager requestImageDataForAsset:pObj options:options resultHandler:^(NSData * _Nullable imageData, NSString * _Nullable dataUTI, UIImageOrientation orientation, NSDictionary * _Nullable info) {
            if (imageData) {
            	[assetsArray addObject:pObj.localIdentifier];
            }
        }];
    }];

    [GlobalFunction writeArrayWithCustomObjToUserDefaults:@"postImages" withArray:assetsArray];

    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        self.postImages = assetsArray;
        dispatch_async(dispatch_get_main_queue(), ^{
            [self.myTableView reloadRowsAtIndexPaths:[NSArray arrayWithObject:[NSIndexPath indexPathForRow:1 inSection:0]] withRowAnimation:UITableViewRowAnimationFade];
        });
    });
    [self dismissViewControllerAnimated:YES completion:NULL];
}
```
这个代理方法中，我通过遍历 `selectedAssets` 序列获取保存的 `PHAsset` 对象，再通过`PHImageManager` 来请求图片数据。这里需要注意的是需要自定义一个 `PHImageRequestOptions` 对象来灵活配置请求参数，其中的同步属性需要设置成 YES，否则存在本地的将会是一个空数组。在确认请求到了图片数据之后再继续接下来的 UI 操作。

在写入本地的方法中，我使用了 `PHAsset` 父类中的唯一标示字符作为存储 `localIdentifier`，这个属性是 `PHObject` 唯一可用的属性，用来区别唯一的资源对象。

然后，在 PickerController dismiss 之后，需要去在 collectionView 上刷新 cell 更新 UI，那么 `collectionView:cellForItemAtIndexPath:` 中简要的实现是这样的：
```objc
NSMutableArray *postImageArr = [GLobalFunction readArrayWithCustomObjFromUserDefaults:@"postImages"];
if (indexPath.row < postImageArr.count) {
    if (SDKVersion10) {
        NSString *assetLocal = [postImageArr objectAtIndex:indexPath.row];
        PHFetchResult *fetchResult = [PHAsset fetchAssetsWithLocalIdentifiers:@[assetLocal] options:nil];
        PHAsset *asset = [fetchResult firstObject];
        PHImageRequestOptions *imageOptions = [[PHImageRequestOptions alloc] init];
        imageOptions.synchronous = YES;
        imageOptions.deliveryMode = PHImageRequestOptionsDeliveryModeHighQualityFormat;
        imageOptions.resizeMode = PHImageRequestOptionsResizeModeExact;
        __block UIImage *copyOfOriginalImage;
        PHImageManager *imageManager = [PHImageManager defaultManager];
        [imageManager requestImageForAsset:asset targetSize:PHImageManagerMaximumSize contentMode:PHImageContentModeDefault options:imageOptions resultHandler:^(UIImage * _Nullable result, NSDictionary * _Nullable info) {
            copyOfOriginalImage = result;
        }];
        [imageView setImageWithUrl:nil placeholderImage:copyOfOriginalImage tapBlock:^(id obj) {
            // set image & callback event.
        }];
    } else {
        // 旧框架内容
    }
}
```
这样的话，调用->选择->保存->读取->更新 UI 整个流程都有了，看起来不错，让我们来运行一下。

## 来，让我们 fix issue ##

运行的环境背景是：老用户升级，他的本地保存着之前的草稿。恰恰是这个因素，让我发现了其中的问题。

进入到发送图文消息的控制器，因为有草稿记录，所以 collectionView 中的一个 cell 里存在着之前选择的照片。点开「选择相册中的照片」，读取了所有照片库的照片。嗯，非常不错，在选择了一张照片，点击完成后，应用 crash。

想必是历史存储的问题，因为之前存储照片的数据结构是 `ALAsset` ，所以当两个不同的数据结构存入时，遍历就出现了类型不匹配的 issue。

解决的话也很简单，只要确保在调用 `-(void)qb_imagePickerController:didFinishPickingAssets:` 时，所传入的 `selectedAssets` 序列集合中保存的都是 `PHAsset` 类型的元素就可以了。

我们先从初始化 PickerController 的时候开始统一类型。所以我的处理是，修改调用照片选择控制器，在读取本地资源和构建新的 `PHAsset` 数组前插入这么一段：
```
...
NSMutableArray *results = [GlobalFunction readArrayWithCustomObjFromUserDefaults:@"postImages"];
NSMutableArray *identiferArray = [NSMutableArray arrayWithCapacity:0];
for (id obj in results) {
    if ([obj isKindOfClass:[NSURL class]]) {
        NSMutableDictionary *dic = [GlobalFunction splitAssetURL:[obj absoluteString]];
        [identiferArray addObject:[dic objectForKey:@"id"]];
    } else {
        [identiferArray addObject:obj];
    }
}
PHFetchResult *fetchResult = [PHAsset fetchAssetsWithLocalIdentifiers:[GlobalFunction readArrayWithCustomObjFromUserDefaults:@"postImages"] options:nil];
NSMutableArray *assetsArray = [NSMutableArray arrayWithCapacity:0];
for (PHAsset *element in fetchResult) {
    [assetsArray addObject:element];
}
[imagePickerWithPhotoKit.selectedAssets addObjectsFromArray:assetsArray];
 ...
```
大意就是，在读取时快速遍历内含的元素是什么类型（我们之前在 `NSUserDefaults` 中保存图片地址），如果是 `NSURL` ，很明显，是类似 `assets-library://asset/asset.JPG?id=9C5EF6C3-0E1C-4F05-B592-CED2203EFE0F&ext=JPG` 这样的资源路径。这里我写了个工具方法，主要目的是为了把 URL 根据参数和值重新构建成一个新的字典返回。且慢，既然重新构建了字典，那么根据 id 取得的值不就是类似于 `9C5EF6C3-0E1C-4F05-B592-CED2203EFE0F` 这样的一个东西？那么这个东西是啥？

我们回到 `PHAsset` ，它有一个 `localIdentifier` 的属性（通过继承 `PHObject` 抽象类而来），如果把某张图片的唯一标示打印出来，会发现和 `assets-library` 中 id 的值结构类似： `9C5EF6C3-0E1C-4F05-B592-CED2203EFE0F/L0/001` 。也就是说，从旧框架中的 `assets-library` 解析出的 id 的值就是 `PHAsset` 的 `localIdentifier` 。

*注意：* 我为什么用了「就是」，看起来 `assets-library` 中的标识符和 `localIdentifier` 有点区别，少了 `/L0/001` 这部分东西。因为仅对于<strong>*JPG*</strong>照片来说的话，加不加没有影响。也就是说，直接将一长串的字符传入 `[PHAsset fetchAssetsWithLocalIdentifiers:options]` 也能正确请求回照片。

所以这里有一个将新标识符转成旧的 `assets-library` URL 的「黑魔法」：
```
NSURL *assetURL = [NSURL URLWithString:[NSString stringWithFormat:@"assets-library://asset/asset.JPG?id=%@&ext=JPG",localIdentifier]];
```
不过这种手段，Apple 文档中并没有提及，如果想要将 App 上架的同学请慎用。

在初始化 PickerController 的时候的修改完成，别忘了在选择相册之前的 collectionView ，它在重用 cell 时也需要读取本地保存的图片信息。修改的大意差不多，就是遍历数组，然后将 `NSURL` 类型的照片地址重构成 `localIdentifier` 的数组。

以上内容就把兼容旧版本数据的问题做好了。接着我想到一个问题，如果用户在添加照片结束后，从系统照片库中删除了这份照片，那会是什么结果？

结果是，会爆访问空数组。

修复这个问题也很简单，在根据 `localIdentifier` 请求回来的 `PHFetchResult` 之后，判断一下 result 的元素数量即可。如为0，则在删除本地保存的对应的 `localIdentifier` 记录，并重新 reload collectionview。

呼，完成！再一次召唤出 `QBPhotoKitImagePickerController` 来测试一下。既然它能把相册中的所有照片都读取，那我选择几张很久之前的试试吧。

选择完毕，按下「完成」！怎么卡住了！~~专业一点，UI 线程阻塞了 ！~~

稍微思考了一下，应该是久远的照片被保存到了 iCloud 上，而且我又在请求图片时开启了线程同步，所以造成了阻塞主线程的情况。

但是，请求同步又是必须的。那要怎么做呢？

有两种方法可以解决这个问题：
> 一、在从 iCloud 下载照片时，获取它的请求进度，然后编写对应的 UI 进度反馈给用户，让用户知道这份图片正在下载。
  二、判断这分照片是否必须从 iCloud 下载。如果是，则 dismiss 选择框，暂时不把该资源加入数组，并在后台下载。等待下载完毕，如用户再次添加则添加成功。

如选择第一种，Photos 框架（就是 PhotoKit）提供了一个 Block 实时监听下载情况。
```
PHImageRequestOptions *options = [PHImageRequestOptions new];
options.progressHandler = ^(double progress, NSError *__nullable error, BOOL *stop, NSDictionary *__nullable info) {
    // 这里写 UI 的进度。
};
```
我选择的是第二种。具体的实现思路就是判断回调过来的 `imageData` 是否存在，存在即添加入数组；不存在则允许网络请求，开启异步，并 dismiss VC。
```
...
// 修改 qb_imagePickerController:didFinishPickingAssets 代理方法。
options.synchronous = YES;
[imageManager requestImageDataForAsset:pObj options:options resultHandler:^(NSData * _Nullable imageData, NSString * _Nullable dataUTI, UIImageOrientation orientation, NSDictionary * _Nullable info) {
    // 判断此图片是否已存在本地，如存在iCloud则后台下载
    if (!imageData) {
        options.networkAccessAllowed = YES;
        options.synchronous = NO;
        [imageManager requestImageDataForAsset:pObj options:options resultHandler:^(NSData * _Nullable imageData, NSString * _Nullable dataUTI, UIImageOrientation orientation, NSDictionary * _Nullable info) {

        }];
    } else {
        // 添加图片id
        [assetsArray addObject:pObj.localIdentifier];
    }
}];
...
```
两个请求。第一个请求本地，第二个请求服务器。其实不仅是回调的 `imageData` 重要，`info` 数据包含的内容也非常丰富。以两张照片为例，一张是系统图片保存的原始图片，一张是根据前者的修改后的图片：
```
{
    PHImageFileDataKey = <PLXPCShMemData: 0x170c29660> bufferLength=1372160 dataLength=1372139;
    PHImageFileOrientationKey = 0;
    PHImageFileSandboxExtensionTokenKey = "5c7dbbef497b6840aa9fa5420ca33cbe14d6aebe;00000000;00000000;000000000000001a;com.apple.app-sandbox.read;00000001;01000003;00000000022891bb;/private/var/mobile/Media/DCIM/103APPLE/IMG_3110.JPG";
    PHImageFileURLKey = "file:///var/mobile/Media/DCIM/103APPLE/IMG_3110.JPG";
    PHImageFileUTIKey = "public.jpeg";
    PHImageResultDeliveredImageFormatKey = 9999;
    PHImageResultIsDegradedKey = 0;
    PHImageResultIsInCloudKey = 0;
    PHImageResultIsPlaceholderKey = 0;
    PHImageResultOptimizedForSharing = 0;
    PHImageResultWantedImageFormatKey = 9999;
}
```
```
{
    PHImageFileDataKey = <PLXPCShMemData: 0x170c29640> bufferLength=2347008 dataLength=2346893;
    PHImageFileOrientationKey = 0;
    PHImageFileSandboxExtensionTokenKey = "f5e42d7289b637792bad6848e52186c43b71a82a;00000000;00000000;000000000000001a;com.apple.app-sandbox.read;00000001;01000003;00000000022983cd;/private/var/mobile/Media/PhotoData/Mutations/DCIM/103APPLE/IMG_3113/Adjustments/FullSizeRender.jpg";
    PHImageFileURLKey = "file:///var/mobile/Media/PhotoData/Mutations/DCIM/103APPLE/IMG_3113/Adjustments/FullSizeRender.jpg";
    PHImageFileUTIKey = "public.jpeg";
    PHImageResultDeliveredImageFormatKey = 9998;
    PHImageResultIsDegradedKey = 0;
    PHImageResultIsInCloudKey = 0;
    PHImageResultIsPlaceholderKey = 0;
    PHImageResultOptimizedForSharing = 0;
    PHImageResultWantedImageFormatKey = 9998;
}
```
以上这个字典可见，其中包含了许多键值对，通过访问对应的 key 还能创造出许多不同的逻辑条件。其中有一个需要提一下：`PHImageResultIsInCloudKey` 。这个 key 代表了该图片资源是否来源于 iCloud ，那我之所以不采用这个 key 作为判断，是因为它虽然能判定来自 iCloud 但不能说明是否已经下载完成，所以我采用了 data 来做判断。

至此，通过相册来访问照片库已经基本适配完毕。再整理一下现在的项目逻辑：
> iOS 10 之前，本地保存的是 NSURL 类型的数组，通过 assets-library 进行管理。
  iOS 10 之后，本地保存的是 PHObject.localIdentifier 字符类型的数组，通过 Photos 框架管理。

## 什么！拍照也要换？！ ##

修改了相册入口，当然还需要兼顾拍照啦。

之前，Apple 提供了 `- (void)writeImageToSavedPhotosAlbum:orientation:completionBlock:` 来负责保存拍摄的图片。现在修改为可以通过 `PHPhotoLibrary` 调用 `- (void)performChanges:completionHandler:` 实现。
```
PHPhotoLibrary *photoLibrary = [PHPhotoLibrary sharedPhotoLibrary];
__block NSString *localId;
[photoLibrary performChanges:^{
    // 创建一个资源修改请求，目的是为了保存图片
    PHAssetChangeRequest *assetChangeRequest = [PHAssetChangeRequest creationRequestForAssetFromImage:pickerImage];
    // 从照片库中获取刚保存的图片，然后获取它的 localIdentifier                 
    localId = [[assetChangeRequest placeholderForCreatedAsset] localIdentifier];
} completionHandler:^(BOOL success, NSError * _Nullable error) {
    if (success) {
        selectedAsset = [GlobalFunction readArrayWithCustomObjFromUserDefaults:key];
        if (!selectedAsset) {
            selectedAsset = [NSMutableArray arrayWithCapacity:0];
        }
        PHFetchResult *result = [PHAsset fetchAssetsWithLocalIdentifiers:@[localId] options:nil];
        [result enumerateObjectsUsingBlock:^(id  _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
            PHAsset *asset = obj;
            PHImageRequestOptions *imageOptions = [[PHImageRequestOptions alloc] init];
            imageOptions.synchronous = YES;
            imageOptions.deliveryMode = PHImageRequestOptionsDeliveryModeHighQualityFormat;
            imageOptions.resizeMode = PHImageRequestOptionsResizeModeExact;
            PHCachingImageManager *cacheImageManager = [[PHCachingImageManager alloc] init];
            [cacheImageManager startCachingImagesForAssets:@[asset] targetSize:PHImageManagerMaximumSize contentMode:PHImageContentModeDefault options:imageOptions];
            [selectedAsset addObject:asset.localIdentifier];
        }];
        [GlobalFunction writeArrayWithCustomObjToUserDefaults:key withArray:selectedAsset];
        handler(selectedAsset);
    }
}];
```
需要注意的是，`performChanges:completionHandler` 是一个异步请求，如果想使用同步，这里有个方法可以替代：`-(BOOL)performChangesAndWait:error:`。

第二个注意点，`[PHAssetChangeRequest creationRequestForAssetFromImage:]` 和 `[assetChangeRequest placeholderForCreatedAsset]` 这两个方法的使用细节，文档是这么描述的：
> Call this method within a photo library change block to create a new asset.
  Use this property if you need to reference the asset created by a change request within the same change block.

也就是说，前面的方法必须在 `performChanges:completionHandler` 中调用，后面的方法需要在前面的方法创建图片后再进行获取。后者依赖前者，前者使用有条件。

可能这里有一个问题，`placeholderForCreatedAsset` 这是啥意思？简要的解释就是，我们通过相机拍摄完成保存的照片，如果想获取 `localIdentifier` 来 fetch 资源就可以通过该属性获取占位符。它是一个 `PHObjectPlaceholder` 类型的属性，其实他的父类就是 `PHObject` ，所以也就能访问 `localIdentifier` 了。

比较复杂的一点是这里使用了 `PHCachingImageManager` ，看名字就知道是为了图像缓存而设置的管理类。那么如何给 `PHAsset` 准备缓存呢？在取得资源对象后，通过缓存管理类单例调取 `startCachingImagesForAssets:targetSize:contentMode:options` 进行缓存。当需要通过对应资源调取图片对象时，通过 `requestImageForAsset:targetSize:contentMode:options:resultHandler:` 来进行调取。这里值得注意的是，只有缓存方法中的参数和请求图片的参数相同时，系统才会自己调用已经准备好的缓存图片。如两个参数对比有出入，那 Photos 框架则会正常加载图片，并重新缓存以便下次使用。

通过相机拍照保存的部分快差不多了，不过别忘了 iOS 10 的权限问题。在 `info.plist` 中需要添加两个键值对以保证能正常调用相机和相片资源库。添加完成后，还需要在代码中进行权限的判断，`PHPhotoLibrary` 也把权限的方法集成了进去，现在只需访问 `+ (PHAuthorizationStatus)authorizationStatus` 就能判断相册的权限。跟修改相册一样，请求权限也是通过一个 Block 来进行代码逻辑规整：`+ (void)requestAuthorization:(void (^)(PHAuthorizationStatus status))handler` 。大概如下意思：
```
if ([PHPhotoLibrary authorizationStatus] == PHAuthorizationStatusNotDetermined || [PHPhotoLibrary authorizationStatus] == PHAuthorizationStatusDenied || [PHPhotoLibrary authorizationStatus] == PHAuthorizationStatusRestricted) {
    [PHPhotoLibrary requestAuthorization:^(PHAuthorizationStatus status) {
        if (status == PHAuthorizationStatusAuthorized) {
            // 验证通过的code...
        }
    }
```

权限的问题也解决啦，让我们再走一遍整个流程。~~应该不会再出什么篓子了吧~~

## 最后的最后，内存泄漏 ##

在运行起来之后，我发现，有这么一个问题。在我多次通过相机拍摄之后，UI 会变的极其卡顿，平均超过三张后应用会出现闪退。看了一眼调试，发现出现了内存泄漏。

多次测试后，发现是在保存图片后再获取 placeholder 时产生了内存泄漏。我 google 了资料查看了 API 也只发现了这么一种方法。

难道是系统提供的方法有问题？

思索了一会，既然是取照片占位符出现了问题，那么取出来是要给 collectionView 去更新 UI 的。去看看 collectionView 会不会存在问题。

我仔细检查了其中的代码，发现涉及图片请求的存在着这么一段话：
```
[imageManager requestImageForAsset:asset targetSize:PHImageManagerMaximumSize contentMode:PHImageContentModeDefault options:imageOptions resultHandler:^(UIImage * _Nullable result, NSDictionary * _Nullable info) {
    copyOfOriginalImage = result;
}];
```
是不是因为这句话引起的呢？我尝试从 `PHImageManager` 中找出答案，其中有两个方法：
> - requestImageForAsset:targetSize:contentMode:options:resultHandler:
  - requestImageDataForAsset:options:resultHandler:

用法完全相同，只不过回调的数据类型不一样。我仔细分析了两个方法回调数据的大小差异，发现在相同尺寸的图片下，下面的方法的数据量明显比上面小。我把所有涉及到上面的方法都替换成了下面的，也解决了我的内存泄漏问题。

## Summary Time! ##

第一次感觉官方文档是多么的重要，它就像是一份 Lego 的组装手册，告诉我什么时候该使用那些方块（类）、这些方块能干什么（方法），以及方块们能组装什么样子的东西（数据结构）。

作为一个学艺不精的 iOS developer，第一次写这么一篇不算技术的技术文，如有纰漏/错误，还请参阅者多多指正。其实许多人都觉得学 iOS 非常简单，我觉得先不论简单与否，能想着深入挖掘以及不断学习新东西的态度才是最重要的。在这个学习的道路上，可能会遇到很简单的只要调用一下提供的 API 就可以了，或者会遇到怎么都调试不清楚的 issue ，一步一步踏实实践，愿意接受别人帮助，同时也乐于帮助别人，就好了。

最后感谢以下开源项目的大力支持，~~希望我也能加油发表自己的开源项目！~~：

<sup>[1]</sup> [questbeat/QBImagePicker](https://github.com/questbeat/QBImagePicker)

以及，那些帮助我解决以上问题的开发者们：

[High memory usage looping through PHAssets and calling requestImageForAsset](http://stackoverflow.com/questions/33274791/high-memory-usage-looping-through-phassets-and-calling-requestimageforasset/39202941#39202941)

[Photos Framework](https://developer.apple.com/reference/photos)

[Objccn.io 照片框架](https://objccn.io/issue-21-4/)

[MicroCai](https://www.zybuluo.com/MicroCai/note/51120)

实践万岁，理解万岁。