---
title:        "谷歌应用内支付"
description:  "使用谷歌提供的应用内支付来赚你的金子吧~~"
image:        "http://placehold.it/400x200"
author:       "Shie1d Shen"
date:         "2018-03-21"
---
# Google In-app Billing
## 写在前面
项目中需要去提供用户可支付购买使用内容，在海外渠道，我们使用Google In-app Billing作为一种支付方式。
当然只是在看笔记的时候做下记录，可以直接去读文档更加方便。送你[电梯](https://developer.android.com/google/play/billing/index.html)

## 关于In-app Billing
其实In-app Billing也就是应用内支付功能是由Google Play提供，它给各个应用提供一个统一的支付方式，目的使用户在各个门户里去付款的时候体验统一，对开发者来说，它把各种繁复的网络请求以及商务流程都封装了起来，减轻了开发者的工作。
### 提前需要注意的地方
应用内支付谷歌允许我们在上面提供非实体的商品的销售，比如mp3，图片等等，或者应用内的功能等，而实体的物件一概不支持~~

谷歌会允许你在他们的控制台里面配置各种商品信息，但注意，你每配置一个商品，它对应的始终是一项物品，用户购买后它就不再支持购买，直到用户消耗了这件商品。其实这个消耗过程就是我们向Google Play发送一个请求说用户用掉了这个商品，然后用户才能再去购买这个物品。
### 使用应用内支付的商品
所有的商品我们得去谷歌控制台配置，一个应用对应一批商品，商品在应用间不通用，即使这些应用的拥有者是相同的。
配置的商品分为订阅以及一般商品。

关于订阅，类似于连续包月这种，我们可以在里面配置订阅周期，比如周月年季度，然后用户在到了时间后会由GooglePlay执行自动扣款，我们不去维护这个商务流程，需要做的就是在用户使用的时候去查询一下这个订阅的状态。

而一般商品在谷歌上配置对应相应的id，一个商品用户在谷歌账户里只能拥有一件，也就是说，如果用户在应用内购买了这件商品，在这件商品没有被消耗掉之前是不能再次购买。消耗这个概念就是告诉谷歌，用户这个商品已经用掉啦，可以再买啦。
### 推荐的用法
在用户要去购买或者使用商品之前，我们可以调用API去查询一次用户所拥有的商品，以及需要展示的商品的信息来更新界面。如果某一个商品id在应用内需要被反复购买，那么我们应该在用户购买完毕后，及时去消耗掉这个商品，并自己维护好用户购买的状态。

### 使用方式
具体的调用API谷歌已经写好示例代码，大家去搂一眼就懂了。[电梯](https://github.com/googlesamples/android-play-billing)

没什么原理，实现方式就是依靠GooglePlay，我们通过IPC告诉GP去获取内容然后返回给我们。
### 关于坑
GooglePlay对商品信息有个缓存，这个缓存就尴尬了，我们没有办法去控制，所以在地理位置切换场景下如果谷歌缓存不刷新，我们就完全没有办法了...

然后就是示例代码里面IabHelper的实现...它的回调很乱....他会在异步的查询方法中加上限制，在一次查询结束之前，你再去查询会崩！！！然而这个限制去掉也不影响查询结果(自己测试的，具体大坑不知道~~)，在某些需要同时查询不同模块商品的场景下，如果加了限制会很蛋疼。

再然后就是里面对setup状态的维护...它的理想状态是跟随Activity走，然而业务中有各种场景，在Activity中创建然后使用就很僵硬，你如果不按规则走，那么效率以及释放就是你要考虑的问题...

## 写在后面
又懒了几个月...这次写个业务文档..磨磨性子。











