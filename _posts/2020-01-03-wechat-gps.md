---
layout: post
title:  "java获取微信签名流程,然后js获取当前经纬度，转换成百度经纬度"
categories: it
tags: [wechat,js,java]
---
>**背景**：项目需要用到就近打卡功能，当前条件在微信内，但是服务端用的是百度地图，需要用户在客户端做到钉钉附近打卡的功能，所以还要把微信的经纬度转换成百度的。

---


## 1. java服务端获取微信所需要的签名
### 1)首先根据appId和appSecret获取accessToken：

```
    @Value("${wechat.appID}")
    private  String appid;
    @Value("${wechat.AppSecret}")
    private  String secret;
    /**
     * AccessToken 缓存对象
     */
    @Builder
    @Data
    private static class AccessTokenCache{

        /**
         * wechat的AccessToken
         */
        private String accessToken;

        /**
         * 过期时间
         */
        private Date timeOutDateTime;

    }
    /**
     * 获取微信accessToken
     *
     * @return
     * @throws BizException
     */
    @Override
    public synchronized String getAccessToken() throws BizException {
        if(accessTokenCache!=null&&accessTokenCache.getTimeOutDateTime().after(new Date())){
            return accessTokenCache.getAccessToken();
        }
        //此处调用微信openapi获取accessToken
        WxAccessTokenBo tokenBo = wechatClient.getAccessToken(appid, secret, WeChatGrantTypeEnum.CLIENT_CREDENTIAL.getValue());
        if(StringUtils.isNotEmpty(tokenBo.getAccess_token())){
            accessTokenCache = AccessTokenCache.builder()
                    .accessToken(tokenBo.getAccess_token())
                    //微信过期时间前半小时
                    .timeOutDateTime(new Date(System.currentTimeMillis()+(tokenBo.getExpires_in()-1800)*1000))
                    .build();
            return tokenBo.getAccess_token();
        }
        throw new BizException("获取微信accessToken失败:"+tokenBo.getErrmsg());
    }
```

### 2） 获取JsapiTicket：
```
    /**
     * JsapiTicket 缓存对象
     */
    @Builder
    @Data
    private static class JsapiTicketCache{

        /**
         * wechat的jsapiTicket
         */
        private String jsapiTicket;

        /**
         * 过期时间
         */
        private Date timeOutDateTime;

    }
    /**
     * 获取微信jsapi_ticket
     *
     * @return
     * @throws BizException
     */
    @Override
    public String getJsapiTicket() throws BizException {
        if(jsapiTicketCache!=null&&jsapiTicketCache.getTimeOutDateTime().after(new Date())){
            return jsapiTicketCache.getJsapiTicket();
        }
        WxJsapiTicketBo jsapiTicketBo = wechatClient.getJsapiTicket(this.getAccessToken(),"jsapi");
        if(StringUtils.isNotEmpty(jsapiTicketBo.getTicket())){
            jsapiTicketCache = JsapiTicketCache.builder()
                    .jsapiTicket(jsapiTicketBo.getTicket())
                    //微信过期时间前半小时
                    .timeOutDateTime(new Date(System.currentTimeMillis()+(jsapiTicketBo.getExpires_in()-1800)*1000))
                    .build();
            return jsapiTicketCache.getJsapiTicket();
        }
        throw new BizException("获取微信jsapiTicket失败:"+jsapiTicketBo.getErrmsg());
    }
```
### 3)获取微信js需要的wx.config
```

    @Builder
    @Data
    private static class WxConfig {

    /**
     * wx的appId
     */
    private String jsapi_ticket;
    /**
     * wx的timestamp
     */
    private String timestamp;
    /**
     * 生成签名的随机串
     */
    private String noncestr;
    /**
     * url
     */
    private String url;
    /**
     * sign
     */
    private String sign;

}
    /**
     * 获取微信js   wx config
     *
     * @return
     * @throws BizException
     */
    @Override
    public WxConfig getWxConfig() throws BizException {
        WxConfig wxConfig = WxConfig.builder()
                .jsapi_ticket(getJsapiTicket())
                .noncestr(UUID.randomUUID().toString().replaceAll("-", "").substring(0, 16))
                .timestamp(String.valueOf(System.currentTimeMillis() / 1000))
               //前端页面的url，#之前的所有
               .url("http://192.168.1.104:8021/").build();
        wxConfig.setSign(JsUtil.generateConfigSignature(wxConfig.getNoncestr(),wxConfig.getJsapi_ticket(),wxConfig.getTimestamp(),wxConfig.getUrl()));
        return wxConfig;
    }
```
### 4)前端用的vue，此处以vue的demo为样例
```
//npm 安装weixin-jsapi
 import wx from 'weixin-jsapi'
export default {
  name: 'app',
  data(){
    return {
      longitude: "",
      latitude: "",
      speed:"",
      accuracy:""
    }
  },
  created(){
  //此处填上从服务端获取的wxConfig
    wx.config({
      debug: true, // 开启调试模式,调用的所有api的返回值会在客户端alert出来，若要查看传入的参数，可以在pc端打开，参数信息会通过log打出，仅在pc端时才会打印。
      appId: '********', // 必填，公众号的唯一标识
      timestamp: 1577945127, // 必填，生成签名的时间戳
      nonceStr: '43bb24913516412d', // 必填，生成签名的随机串
      signature: '67d0e11abe6f380887ae47421e5c2f60c6b90c8d',// 必填，签名
      jsApiList: ['getLocation','openLocation'] // 必填，需要使用的JS接口列表
    });
    wx.error(function(res){
      // config信息验证失败会执行error函数，如签名过期导致验证失败，具体错误信息可以打开config的debug模式查看，也可以在返回的res参数中查看，对于SPA可以在这里更新签名。
      console.log(res)
    });
    wx.ready((res1)=>{
      console.log(res1)
      wx.getLocation({
        type: 'wgs84', // 默认为wgs84的gps坐标，如果要返回直接给openLocation用的火星坐标，可传入'gcj02'
        success: (res)=>{
          // wx.openLocation({
          //   latitude: res.latitude, // 纬度，浮点数，范围为90 ~ -90
          //   longitude: res.longitude, // 经度，浮点数，范围为180 ~ -180。
          //   name: '', // 位置名
          //   address: '', // 地址详情说明
          //   scale: 1, // 地图缩放级别,整形值,范围从1~28。默认为最大
          //   infoUrl: '' // 在查看位置界面底部显示的超链接,可点击跳转
          // });
          this.latitude = res.latitude; // 纬度，浮点数，范围为90 ~ -90
          this.longitude = res.longitude; // 经度，浮点数，范围为180 ~ -180。
          this.speed = res.speed; // 速度，以米/每秒计
          this.accuracy = res.accuracy; // 位置精度
          console.log(this)
          let convertor = new BMap.Convertor();
          let ggPoint = new BMap.Point(this.longitude, this.latitude);
          let pointArr = [];
          pointArr.push(ggPoint);
          convertor.translate(pointArr, 1, 5, (data) =>{
            // 显示
            // this.$vux.alert.show({
            //   title: 'Vux is Cool',
            //   content: data,
            //   onShow () {
            //     console.log('Plugin: I\'m showing')
            //   },
            //   onHide () {
            //     console.log('Plugin: I\'m hiding')
            //   }
            // })
            alert(data);
            if (data.status === 0) {
              console.log("百度经度：" + data.points[0].lng);
              console.log("百度纬度：" + data.points[0].lat);
            }
          })
        }
      });
    })


    // getDictionaryInfoByType(1).then(res=>{
    //   this.$store.dispatch('setDictionaryInfo', res.data.data.map(x=>{
    //     return {key:x.dictionaryName,value:x.id}
    //   }))
    // })
  }
}
```
## 2.页面控制台打印结果
![image](http://image.itstabber.com/uPic/WX20200103-133111@2x.png)