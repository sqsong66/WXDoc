## 构建`OverseaPay.PayBuilder`
在构建`OverseaPay.PayBuilder`的时候，需要设置以下参数：

- `isGoogleSubscription`: 是否是Google Play订阅。  
- `userId`: 用户id。用于查询该用户是否有订阅，以及根据用户id(区分用户)来保存用户的漏单信息。  
- `hidePayPal`: 隐藏PayPal支付选项。Google Play订阅需要隐藏PayPal支付  
- `useNewGooglePay`: 是否启用新的Google Pay支付逻辑。公共库里面为了不影响其他app业务，独立出`NewGooglePayLogic`类，该类可启用订阅，同时兼容Google Play普通商品购买。

```java
OverseaPay.PayBuilder builder = new OverseaPay.PayBuilder();
String userId = userInfo.getUser() == null ? ""
         : userInfo.getUser().getUser_id();
//Google play
builder.setToken(userInfo.getIdentity_token())
    .setGoogleSubscription(isSubscription) // 是否是订阅商品
    .setHidePayPal(isSubscription) // Google Play订阅商品隐藏PayPal支付
    .setGoogleSku(googleSku) // 商品id
    .setUseNewGooglePay(true) // 启用新的Google Pay支付逻辑 
    .setUserId(userId) // 用户id
    .setShowPrice("$" + (isSubscription ? googlePrice : showPrice));
```

## 检测漏单并补充提交
根据自己的业务需求在适当的位置检测漏单情况。

- `identityToken`: 用户登录验证token, 提交订单数据需要使用；  
- `userId`: 用户id，查询该用户的漏单需要使用；
```java
GooglePayOrderManager.getInstance(getContext())
    .checkUnUploadGooglePayOrder(identityToken, userId);
```

## 漏单数据提交监听(可选)
可通过`GooglePayOrderManager`的`registerUploadListener`方法监听漏单提交成功失败情况。
```java
 private void checkGooglePlayOrder() {
        // 注册提交Google Play订单监听
        GooglePayOrderManager.getInstance(getApplicationContext()).registerUploadListener(new GooglePayOrderManager.GooglePayUploadListener() {
            @Override
            public void onUploadSuccess() {
                // 更新vip信息
                UserInfo userInfo = LoginManager.getInstance().getUserInfo();
                VipUtil.loadVipInfo(userInfo);
            }

            @Override
            public void onUploadFail(String errorJson) {
                Logger.e("onUploadFail: " + errorJson);
                LogRecordHelper.getInstance().uploadLogRecord(LogRecord.CLICK_GOOGLE_PLAY_ORDER_FAIL, errorJson);
            }
        });
    }
```
