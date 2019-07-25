#_微信小程序微信支付_

###官方流程图如下：
[微信小程序微信支付官方流程图链接](https://pay.weixin.qq.com/wiki/doc/api/wxa/wxa_api.php?chapter=7_4&index=3)
![avatar](https://pay.weixin.qq.com/wiki/doc/api/img/wxa-7-2.jpg)

###我简化的流程：
1. 本地发起下单请求调用云函数并传送数据
2. 云函数处理数据并返回5个参数
3. 本地接受5个参数，发起支付请求
4. 交易结束

主要代码：

```javascript
//第一步，本地发起下单请求并传送数据。这一步，在你的wxml中的某个元素
//中绑定事件<button bindtap='pay'></button>。通过这个pay函数，
//触发云函数并传递一些数据
pay: function(){
	//需要上传给云函数的数据
	let uploadData = {
		//此次需要支付的金额，单位是分。例如¥1.80=180
		"total_fee": "180",
		//用户端的ip地址
     	"spbill_create_ip": "123.123.123.123"
	}
	//调用云函数
	wx.cloud.callFunction({
		//云函数的名字，这里我定义为payment
		name: "payment",
		//需要上传的数据
		data: uploadData
	}).then(res => {
		//这个res就是云函数返回的5个参数
		//通过wx.requestPayment发起支付
		wx.requestPayment({
			timeStamp: res.result.data.timeStamp,
			nonceStr: res.result.data.nonceStr,
			package: res.result.data.package,
			signType: res.result.data.signType,
			paySign: res.result.data.paySign,
			success: res => {
				//支付成功
			},
			fail: err => {
				//支付失败
			}
	})
}


```

##以上就是本地端的全部代码，接下来我们只需要搞定云函数的代码就完成全部的工作了。


###云函数的内容是：

1. 调用小程序登陆API -> Openid
2. 生成商户订单
3. 调用统一下单API -> prepay_id
4. 将组合数据再次签名,返回5个参数

---

>我创建的云函数命名为payment，此时云函数结构应该为

```
payment
|__index.js
|__package.json
```

##每一步的详细做法

##1. 调用小程序登陆API -> Openid

这一步**目的是获取用户的Openid**

>在云函数的index.js中加上以下代码

```javascript
//获取云实例
const cloud = require('wx-server-sdk')
//云初始化
cloud.init()
//获取微信调用上下文信息，其中包括Openid，Appid等
const wxContext = cloud.getWXContext()
//获取用户openid
const openid = wxContext.OPENID
```
到这里我们已经达成我们第一步的目的了。
-
##2. 生成商户订单

[微信支付开发文档-统一下单](https://pay.weixin.qq.com/wiki/doc/api/wxa/wxa_api.php?chapter=9_1&index=1)

这一步的目的是为了**生成调用支付统一下单API的订单**。根据官方文档，我们需要以下数据：

1. appid（小程序ID）
* openid（用户OPENID）
* mch_id（商户号）
* nonce_str（随机字符串）
* body（商品描述）
* out\_trade\_no（商户订单号）
* total_fee（标价金额）
* spbill\_create\_ip（终端IP）
* notify_url（通知地址）
* trade_type（交易类型）
* key（密钥）
* sign（签名）

我们一个一个解决。

####1. appid
小程序管理员进入公众平台、使用小程序帐户登录后，点击左侧菜单中的「设置」，在「开发设置」一项，就可以查询到小程序的AppID。
>示例值`wxd678efh567hg6787`

在云函数的index.js中加上以下代码：

```javascript
const appid='wxwxd678efh567hg6787'
```

####2. openid
第一步已经获得。
>示例值`oUpF8uMuAJO_M2pxb1Q9zNjWeS6o`

####3. mch_id
登陆微信支付商户平台<https://pay.weixin.qq.com>,点击上方「账户中心」，在「个人信息」中的「登陆账号」就是mch_id。
>示例值`1230000109`

在云函数的index.js中加上以下代码：

```javascript
const mch_id='1230000109'
```

####4. nonce_str
任意生成的随机数，不超过32位。你可以自己写个函数。

>示例值`5K8264ILTKCH16CQ2502SI8ZNMTM67VS`

>我在云函数中创建了一个新的JS文件(random.js)来保存这个函数，此时云函数的结构如下

```
payment
|__index.js
|__package.json
|__random.js
```
其中random.js的内容为:

```javascript
function random(){
  var result = ''
  const wordList = ['a', 'b', 'c', 'd', 'e', 'f', 'g', 'h', 'i', 'j', 'k', 'l',
  'm', 'n', 'o', 'p', 'q', 'r', 's', 't', 'u', 'v', 'w', 'x', 'y', 'z', '1', '2',
  '3', '4', '5', '6', '7', '8', '9', '0']
  for(let i=0;i<31;i++){
    result += wordList[Math.round(Math.random()*36)]
  }
  return result
}

module.exports = random()

```
然后在云函数index.js中加上以下代码:

```javascript
const random = require("random.js")
```

####5. body
格式为：商家名称-销售商品类目
>示例值`腾讯-游戏`

在云函数index.js中加上以下代码:

```javascript
const body = "腾讯-游戏"
```

####6. out\_trade\_no
商户系统内部订单号，要求32个字符内，只能是数字、大小写字母_-|\*且**在同一个商户号下唯一**。由自己定义，推荐用当下时间+商品编号组成。
>示例值`20150806125346`

在云函数index.js中的exports.main函数中加上以下代码:

```javascript
//这里我只使用了当下时间。只要这个数字不是重复的就可以。
const out_trade_no = Date.parse(new Date()).toString()
```

####7. total_fee
订单总金额，单位为分。比如当前需支付¥6.80，则total_fee为680。
>示例值`88`

```这里需要用到我们上传过来的值，先不管```
####8. spbill\_create\_ip
支持IPV4和IPV6两种格式的IP地址。调用微信支付API的机器IP。
>示例值`123.12.12.123`

```这里需要用到我们上传过来的值，先不管```
####9. notify_url
异步接收微信支付结果通知的回调地址，通知url必须为外网可访问的url，不能携带参数。在这里可以填上你自己服务器的url。
>示例值`http://www.weixin.qq.com/wxpay/pay.php`

在云函数index.js中加上以下代码:

```javascript
//随便填写个服务器就行，我在使用中没有遇到什么问题
const notify_url = 'http://www.weixin.qq.com/wxpay/pay.php'
```
####10. trade_type
小程序的trade_type为JSAPI。
>示例值`JSAPI`

在云函数index.js中加上以下代码:

```javascript
const trade_type = 'JSAPI'
```
####11. key
key为商户平台设置的密钥key，是由你自己设置的。key设置路径：微信商户平台(pay.weixin.qq.com)-->账户设置-->API安全-->密钥设置。
>示例值`1a79a4d60de6718e8e5b326e338ae533`

在云函数index.js中加上以下代码:

```javascript
const key = '1a79a4d60de6718e8e5b326e338ae533'
```
####12. sign
将以上除key外所有信息按照参数名ASCII码从大到小拼接成字符串，用&分割，将key放在最后。
>字符串示例值:

```
appid=wxd678efh567hg6787&body=微信-游戏&mch_
id=1230000109&nonce_str=5K8264ILTKCH16CQ2502SI8ZNMTM6
7VS&notify_url=http://www.weixin.qq.com/wxpay/pay.php&
openid=oUpF8uMuAJO_M2pxb1Q9zNjWeS6o&out_trade_no=2015080
6125346&spbill_create_ip=123.12.12.123&total_fee=88&trad
e_type=JSAPI&key=1a79a4d60de6718e8e5b326e338ae533
```

此字符串的MD5码的大写就是sign。因为上面的total_fee与spbill\_create\_ip我们还没处理，所以这个数据放到下面再处理。
>MD5码示例值`C380BEC2BFD727A4B6845133519F3AD6`

-
到此为止你的云函数结构应该为:

```
payment
|__index.js
|__package.json
|__random.js
```
其中index.js的内容应该为:

```javascript
//云函数入口文件
const cloud = require('wx-server-sdk')
cloud.init()
const openid = cloud.getWXContext().OPENID
const appid = 'wxwxd678efh567hg6787'
const mch_id = '1230000109'
const random = require('random.js')
const body = "腾讯-游戏"
const notify_url = 'http://www.weixin.qq.com/wxpay/pay.php'
const trade_type = 'JSAPI'
const key = '1a79a4d60de6718e8e5b326e338ae533'

//云函数入口函数
exports.main = async (event, content) => {
	const out_trade_no = Date.parse(new Date()).toString()
}
```
接下来我们处理上面没有处理的total_fee与spbill\_create\_ip，以及sign。
其中total\_fee和spbill\_create\_ip是由客户端上传的，这两个数据就在云函数入口函数的参数event中，所以我们在云函数入口函数里面加上以下代码

```javascript
const total_fee = event.total_fee
const spbill_create_ip = event.spbill_create_ip
```
最后，我们需要处理sign，按照12.sign提到的规则，在云函数入口函数里面加上以下代码

```javascript
let stringA = `appid=${appid}&body=${body}&
mch_id=${mch_id}&nonce_str=${random}&
notify_url=${notify_url}&openid=${openid}&
out_trade_no=${out_trade_no}&
spbill_create_ip=${spbill_create_ip}&
total_fee=${total_fee}&trade_type=${trade_type}&
key=1a79a4d60de6718e8e5b326e338ae533`
```
我们现在需要将这个字符串进行MD5码加密，所以需要安装一个npm包来完成这个任务。右键点击云函数pyament，选择「在终端打开」，输入下面的命令：

```bash
npm install --save crypto
```
完成后在云函数入口文件处加上以下代码

```javascript
const crypto = require("crypto")
```

这样我们就成功地将crypto这个加密工具包引入我们的云函数里了。然后我们需要在云函数入口函数里使用它对刚刚的stringA进行MD5加密。所以我们在`let stringA = ...`这行代码下面添加以下代码

```javascript
var sign = crypto.createHash('md5').update(stringA).digest('hex').toUpperCase()
```

###以上除key的11个信息就是我们调用统一下单API所需要的全部数据了
我们现在需要将这些数据转成xml格式,例如：

```xml
<xml>
<appid>wxd930ea5d5a258f4f</appid>
<mch_id>10000100</mch_id>
<device_info>1000</device_info>
<body>test</body>
<nonce_str>ibuaiVcKdpRxkhJA</nonce_str>
<sign>9A0A8659F005D6984697E2CA0A9CF3B7</sign>
...
</xml>
```

在云函数中新建一个requestData.js，写下如下函数，用来完成将数据转成xml的任务

```javascript
function requestData(
  appid,
  mch_id,
  nonce_str,
  sign,
  body,
  out_trade_no,
  total_fee,
  spbill_create_ip,
  notify_url,
  trade_type,
  openid
){
  let data = "<xml>"
  data += "<appid>"+appid+"</appid>"
  data += "<mch_id>"+mch_id+"</mch_id>"
  data += "<nonce_str>"+nonce_str+"</nonce_str>"
  data += "<sign>"+sign+"</sign>"
  data += "<body>"+body+"</body>"
  data += "<out_trade_no>"+out_trade_no+"</out_trade_no>"
  data += "<total_fee>"+total_fee+"</total_fee>"
  data += "<spbill_create_ip>"+spbill_create_ip+"</spbill_create_ip>"
  data += "<notify_url>"+notify_url+"</notify_url>"
  data += "<trade_type>"+trade_type+"</trade_type>"
  data += "<openid>"+openid+"</openid>"
  data += "</xml>"
  return data
}

module.exports = requestData
```

>此时云函数的结构为

```
payment
|__index.js
|__package.json
|__package-lock.json //由npm install产生的文件
|__random.js
|__requestData.js
```

我们需要将requestData.js文件导入到我们的项目。在云函数入口文件那里添加以下代码

```javascript
const requestData = require("requestData.js")
```

现在，我们可以生成调用支付统一下单API的订单了,这个dataBody就是订单。

```javascript
let dataBody = reqData(
    appid,
    mch_id,
    random,
    sign,
    body,
    out_trade_no,
    total_fee,
    spbill_create_ip,
    notify_url,
    trade_type,
    openid
  )
```
到这里我们已经达成我们第二部的目的了。
-

##3.调用统一下单API -> prepay_id

[官方文档](https://pay.weixin.qq.com/wiki/doc/api/wxa/wxa_api.php?chapter=9_1)

我们需要对官方提供的链接`https://api.mch.weixin.qq.com/pay/unifiedorder`发起统一下单，所以这里我们需要一个npm包来帮我们完成request请求，并且由于发起请求后的返回值是xml格式的，所以我们还需要一个npm包来帮助我们解析xml格式文件。故右键点击云函数payment，选择「在终端打开」，输入下面命令：

```bash
npm install --save request
npm install --save xmlreader
```

在云函数入口文件中引入上面两个包：

```javascript
const request = require("request")
const xmlreader = require("xmlreader")
```

然后就可以在云函数入口函数中发起对统一下单API的request请求了，由于request是异步请求，所以我们需要返回一个Promise。

```javascript
return new Promise(reslove => {
	request({
		//官方统一下单api的url
		url: 'https://api.mch.weixin.qq.com/pay/unifiedorder',
		//请求方法，post
		method: "POST",
		//需要传送的订单，就是刚刚我们生成的dataBody
		body: dataBody
	}, body => {
		//body就是我们收到的数据，我们需要得到其中的prepay_id
		//使用xmlreader解析body，获得其中的prepay_id
		xmlreader.read(body, res => {
			//此时我们已经完成第三步的目的了
			let prepay_id = res.xml.prepay_id.text()
		}
	}
}
```
第三步目的完成
-

##4.将组合数据再次签名，返回5个参数

已知wx.requestPayment()需要五个参数，分别是

1. timeStamp
2. nonceStr
3. package
4. signType
5. paySign

其中，timeStamp为时间戳，可由`Date.parse(new Date()).toString()`取得。
nonceStr为随机字符串，可由我们的随机函数取得。
package就是上一步获得的prepay_id，
signType是签名类型，我们选择的是MD5。所以我们现在只剩下paySign未知，得到paySign的方法为

```
paySign = MD5(appId=wxd678efh567hg6787&nonceStr=5K826
4ILTKCH16CQ2502SI8ZNMTM67VS&package=prepay_id=wx20170
33010242291fcfe0db70013231072&signType=MD5&timeStamp=
1490840662&key=qazwsxedcrfvtgbyhnujmikolp111111) = 22D
9B4E54AB1950F51E0649E8810ACD6
```

所以我们在上一步的代码中接着写

```javascript
return new Promise(reslove => {
	request({
		url: 'https://api.mch.weixin.qq.com/pay/unifiedorder',
		method: "POST",
		body: dataBody
	}, body => {
		xmlreader.read(body, res => {
			let prepay_id = res.xml.prepay_id.text()
			let timeStamp = Date.parse(new Date()).toString()
			let str = `appId=${appid}&nonceStr=${random}&package=prepay_id=${prepay_id}&signType=MD5&timeStamp=${timeStamp}&key=1a79a4d60de6718e8e5b326e338ae533`
			let paySign = crypto.createHash('md5').update(str).digest('hex')
			//返回上面的五个参数
			reslove({
				data: {
					timeStamp: timeStamp,
            		nonceStr: random,
            		package: `prepay_id=${prepay_id}`,
            		signType: 'MD5',
            		paySign: paySign
            	}
            })
		}
	}
}
```

至此，微信小程序支付流程结束。
-

此时云函数结构为：

```
payment
|__index.js
|__package.json
|__package-lock.json
|__random.js
|__requestData.js
```

index.js:

```javascript
//云函数入口文件
const cloud = require('wx-server-sdk')
cloud.init()
const openid = cloud.getWXContext().OPENID
const appid = 'wxwxd678efh567hg6787'
const mch_id = '1230000109'
const random = require('random.js')
const body = "腾讯-游戏"
const notify_url = 'http://www.weixin.qq.com/wxpay/pay.php'
const trade_type = 'JSAPI'
const key = '1a79a4d60de6718e8e5b326e338ae533'
const crypto = require("crypto")
const requestData = require("requestData")
const request = require("request")
const xmlreader = require("xmlreader")

//云函数入口函数
exports.main = async (event, content) => {
    const out_trade_no = Date.parse(new Date()).toString()
    const total_fee = event.total_fee
    const spbill_create_ip = event.spbill_create_ip
    let stringA = `appid=${appid}&body=${body}&mch_id=${mch_id}&nonce_str=${random}&notify_url=${notify_url}&openid=${openid}&out_trade_no=${out_trade_no}&spbill_create_ip=${spbill_create_ip}&total_fee=${total_fee}&trade_type=${trade_type}&key=1a79a4d60de6718e8e5b326e338ae533`
    var sign = crypto.createHash('md5').update(stringA).digest('hex').toUpperCase()
    let dataBody = reqData(
	    appid,
	    mch_id,
	    random,
	    sign,
	    body,
	    out_trade_no,
	    total_fee,
	    spbill_create_ip,
	    notify_url,
	    trade_type,
	    openid
	  )
	return new Promise(reslove => {
	    request({
	        url: 'https://api.mch.weixin.qq.com/pay/unifiedorder',
	        method: "POST",
	        body: dataBody
	    }, body => {
	        xmlreader.read(body, res => {
	            let prepay_id = res.xml.prepay_id.text()
	            let timeStamp = Date.parse(new Date()).toString()
	            let str = `appId=${appid}&nonceStr=${random}&package=prepay_id=${prepay_id}&signType=MD5&timeStamp=${timeStamp}&key=1a79a4d60de6718e8e5b326e338ae533`
	            let paySign = crypto.createHash('md5').update(str).digest('hex')
	            //返回上面的五个参数
	            reslove({
	                data: {
	                    timeStamp: timeStamp,
	                    nonceStr: random,
	                    package: `prepay_id=${prepay_id}`,
	                    signType: 'MD5',
	                    paySign: paySign
	                }
	            })
	        }
	    }
}
```

random.js:

```javascript
function random(){
  var result = ''
  const wordList = ['a', 'b', 'c', 'd', 'e', 'f', 'g', 'h', 'i', 'j', 'k', 'l',
  'm', 'n', 'o', 'p', 'q', 'r', 's', 't', 'u', 'v', 'w', 'x', 'y', 'z', '1', '2',
  '3', '4', '5', '6', '7', '8', '9', '0']
  for(let i=0;i<31;i++){
    result += wordList[Math.round(Math.random()*36)]
  }
  return result
}

module.exports = random()
```

requestData.js:

```javascript
function requestData(
  appid,
  mch_id,
  nonce_str,
  sign,
  body,
  out_trade_no,
  total_fee,
  spbill_create_ip,
  notify_url,
  trade_type,
  openid
){
  let data = "<xml>"
  data += "<appid>"+appid+"</appid>"
  data += "<mch_id>"+mch_id+"</mch_id>"
  data += "<nonce_str>"+nonce_str+"</nonce_str>"
  data += "<sign>"+sign+"</sign>"
  data += "<body>"+body+"</body>"
  data += "<out_trade_no>"+out_trade_no+"</out_trade_no>"
  data += "<total_fee>"+total_fee+"</total_fee>"
  data += "<spbill_create_ip>"+spbill_create_ip+"</spbill_create_ip>"
  data += "<notify_url>"+notify_url+"</notify_url>"
  data += "<trade_type>"+trade_type+"</trade_type>"
  data += "<openid>"+openid+"</openid>"
  data += "</xml>"
  return data
}

module.exports = requestData
```