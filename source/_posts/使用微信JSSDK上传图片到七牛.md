title: 使用微信JSSDK上传图片到七牛
date: 2016-06-01 17:43:15
tags: [JSSDK,Weixin,Qiniu]
toc: true
---

### 总体思路

在微信下选好图片后将图片上传到微信服务器，在后端使用微信服务器返回的图片 `serverId` 加上调用接口的 `access_token` 通过七牛的 `fetch` 接口向微信服务器下载多媒体文件的接口请求图片的二进制流，然后保存至自己七牛账号内的特定 `bucket`。

<!-- more -->

#### 调用微信 chooseImage 接口，成功后调用 uploadImage 接口

```js
wx.ready(function () {
            // 5.1 拍照、本地选图
            var images = {
                localId: [],
                serverId: []
            };
            document.querySelector('#chooseImage').onclick = function () {
                wx.chooseImage({
					count:1,
                    success: function (res) {
                        images.localId = res.localIds;
						$(".weui_uploader_file").show().css('background-image', 'url(' + images.localId[0] + ')');
//                        alert('已选择 ' + res.localIds.length + ' 张图片');
                    }
                });
            };

            // 5.3 上传图片
            document.querySelector('#uploadImage').onclick = function () {
                if (images.localId.length == 0) {
                    alert('请先选择图片');
                    return;
                }

                $.showLoading("请稍等...");
                var i = 0, length = images.localId.length;
                images.serverId = [];
                function upload() {
                    wx.uploadImage({
                        localId: images.localId[i],
                        success: function (res) {
                            i++;
                            //alert('已上传：' + i + '/' + length);
                            console.log(res);
                            images.serverId.push(res.serverId);

                            var url = "<?php echo base_url('vote/home/fetch_url_to_qiniu'); ?>";
                            $.get(url, {'media_id': res.serverId,'memo': $('.weui_textarea').val()}, function (response) {
                                $.hideLoading();
                                var data = $.parseJSON(response);
                                if (data.ret == 0) {
                                    $.toast(data.msg);
                                    window.location.href = "<?php echo base_url('vote/home/vote_list'); ?>";
                                } else {
                                    $.toast("操作失败", "forbidden");
                                    $.noti({
                                        text: data.msg,
                                        time: 3000
                                    });
									return;
                                }
                            });

                            if (i < length) {
                                upload();
                            }
                        },
                        fail: function (res) {
                            alert(JSON.stringify(res));
                        }
                    });
                }

                setTimeout(function () {
                    upload();
                }, 1000);

            };

            wx.error(function (res) {
                console.log(res.errMsg);
            });
        });
```

#### 在后台使用七牛的 fetch 接口向微信服务器请求文件并存入自己的七牛仓库

下面是七牛PHP fetch接口demo

```php
<?php
require_once __DIR__ . '/../autoload.php';
use Qiniu\Auth;
use Qiniu\Storage\BucketManager;
$accessKey = 'Access_Key';
$secretKey = 'Secret_Key';
$auth = new Auth($accessKey, $secretKey);
$bmgr = new BucketManager($auth);
$url = 'http://php.net/favicon.ico';
$bucket = 'Bucket_Name';
$key = time() . '.ico';
list($ret, $err) = $bmgr->fetch($url, $bucket, $key);
echo "=====> fetch $url to bucket: $bucket  key: $key\n";
if ($err !== null) {
    var_dump($err);
} else {
    echo 'Success';
}
```

其中需要特别注意的地方是，通过微信返回的 `serverId` 去微信服务器下载图片的接口`微信公众号`和`微信企业号`是不一样的（微信企业号开发文档没有提供媒体下载接口以为是同公众号下载接口一样，结果总是提示 `aceess_token`）

- 微信公众号上传下载媒体接口，[官方文档](http://mp.weixin.qq.com/wiki/12/58bfcfabbd501c7cd77c19bd9cfa8354.html)

- 微信企业号上传下载媒体接口，上传接口[官方文档](http://mp.weixin.qq.com/wiki/12/58bfcfabbd501c7cd77c19bd9cfa8354.html)

  下载接口：<a>https://qyapi.weixin.qq.com/cgi-bin/media/get?access_token=ACCESS_TOKEN&media_id=MEDIA_ID
  
 [微信 JSSDK 说明文档](http://mp.weixin.qq.com/wiki/7/aaa137b55fb2e0456bf8dd9148dd613f.html#.E4.B8.8A.E4.BC.A0.E5.9B.BE.E7.89.87.E6.8E.A5.E5.8F.A3)
 [七牛 fetch 接口说明文档](http://developer.qiniu.com/docs/v6/api/reference/rs/fetch.html)
 [七牛 PHP-SDK fetch 接口demo](https://github.com/qiniu/php-sdk/blob/master/examples/fetch.php)


