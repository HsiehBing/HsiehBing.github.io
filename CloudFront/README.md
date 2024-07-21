## 說明
本文中將介紹CloudFront各種功能，並透過一些小Lab驗證。

- Last update :2024/07/07

## General

- **Price class** - 使用不同CloudFront節點會有不同費用
- **Alternate domain name (CNAME) - optional** - 如果CloudFront有串接CNAME時的設定
- **Custom SSL certificate - optional** - 當有使用ACM時會需要在這設定
- **Supported HTTP versions** - 設定HTTP
- **Default root object - optional** - 設定default訪問的路徑。
以S3為例在設定時僅能設定對應的檔案，default的路徑是在Origin的Origin path設定。因此當需要在訪問cloudfront的default路徑為s3的/path2/index2.html，就需要將origin path設定為path2，並在General的Default root object設定

- **Standard logging** - 輸出log至S3 Bucket中
當需要檢視log時可以透過Athena檢視[3]。
- **Description - optional** - CloudFront在頁面中顯示的名稱，以分辨不同CloudFront

- **Continuous deployment** - CloudFront可以測試在不同CloudFront 設定下對服務的影響，因此當檢視CloudFront時會有staging以及Production。
當在測試時，可以透過weight或者是不同header的方式測試Staging以及Production。header的開頭需要為aws-cf-cd-
當測試完確認後，可以直接選擇Promote切換Stag以及Prod，當Promote完成後CloudFront的distribute不會改變，而是將Stag的設定套用至Prod

## Security
- **WAF** - 是否啟用WAF
- **CloudFront geographic restrictions** - CloudFront可以根據國家直接設定白名單以及block名單

## Origin
- **Origin domain** - 可以訪問哪些domain
該domain default設定是S3、Load Balancer、API Gateway、MediaStore container、Mediapackage container、Mediapackage V2 endpoint。
除了上述之外也可以使用EC2或者自定義的domain
如果S3已經設定為靜態網站，要使用對應的endpoint

- **Origin path - optional** - 訪問時訪問該origin的哪個分頁
以S3為例 - 如果Origin path設定為/demo2時，當使用者訪問cloudFront/index.html時，會訪問s3的/demo2/index.html

- **Name** - 名稱
單純區別的名稱

- **Add custom header**
該處的header設定主要是由CloudFront傳至orgin的設定。
以使用情境來說，當CloudFront傳至EC2時，EC2可以透過header的方式只允許帶有header的請求進行訪問。
以下是簡單的FLASK測試範例，當使用者未帶header訪問時會回傳401。
```python=
from flask import Flask, request

app = Flask(__name__)

@app.route('/')
def hello():
    if request.headers.get('test123') == 'values456':
        return 'Hello World'
    else:
        return 'Unauthorized', 401

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=80)
```

**Enable Origin Shield** - Origin Shield主要是提供靜態資訊快取的設定

**Additional settings** - Connection 的設定

**Origin groups** - 當CloudFront提供多個origin時，當A Origin發生指定的錯誤時(4xx 5xx)會依順序response B Origin。


## Behaviros
- **Path pattern** - 主要是針對請求的時候的反應
如：CloudFront使用OAC串接S3靜態圖片時，可以設定.png皆為使用快取

- **Origin and origin groups** - 與先前的Origin做合併

### Viewer

- **Viewer protocol policy** - 當http或https請求至CloudFront時，CloudFront可以採取的措施。


- **Allowed HTTP methods** - 可允許的http請求，僅能透過選項選取。

- **Restrict viewer access**-如果需要在訪問時使用進一步限制

    如果是使用S3 OAC至少要選擇Trusted key groups (recommended)。
    
    如果是Ec2會需要提供public key，Public key 可以透過openssl建立當建立完憑證後需要將public key 上傳至CloudFront的key manager中，並且建立Key group。
        
    使用者訪問時會需要提供以下內容```https://{distrubute}cloudfront.net/path?Expires={expire time(Unix Timestamp)}&Signature={下方說明}&Key-Pair-Id={Key management中Public keys的ID}```
    
    Signature 設定[4] -
    (1)建立canned-policy，格式如下
    ```
    {
  "Statement":[
    { 
      "Resource":"https://dqofozeouve3j.cloudfront.net/0m-HR.mp4",
      "Condition":{
        "DateLessThan":{ "AWS:EpochTime":1704038400 }
           }
        }
      ]
    }
    ```
    (2)透過ssl 計算 Signature
    ```
    cat canned-policy | tr -d " \t\n\r" | openssl sha1 -sign private_key.pem | openssl base64 | tr -- '+=/' '-_~' | tr -d "\n" | perl -ne 'print "$_\n"'
    ```
    計算後會得到Signature值
    ```
    AtVt~jTeY-~oPMtRpeLw1PdCtF7xZh-sHwcoY0VmgNMdLtN1l3YeY4mvw0I85NBIfEgoWNjn2B4BbUGB5mNPTyhlCXWaWxPsBd2YHkIcjOHII-WnlvQTqi9QpvVvPAKpUgrOWcAkIhaLRYHxRq1NMuzVzDfjbP1ec7fcfdI5sZnh2dYQsq1kNpqONMwCaUhOQdBiATwbRtjWmvQbSo7e7jQ8ZDbHeknvUVlkIZBQCZnUZk47qLpYsrUOSzadXCRAi81xaeLCiKJuqvLVQ~wB0hL8Z5q-wWqMtqRHnsLV1hKwqtoA0cKLfHYIMBsRMJixhgKOK7uYeJY3nMwmu1RBKQ__
    ```
    若使用Trusted signer會需要透過root做設定，這邊不建議使用。

### Cache key and origin requests
Cache的使用主要是透過CloudFront就近的節點讓使用者可以更快速訪問

**cache設定** - 
(1)CloudFront中Legacy cache settings直接設定cache時間
(2)若是串接S3，可以點選S3的Object設定Metadata，設定Type選擇 System defined； Key 選擇 Cache-Control或Expires
(3)header
(4)query string: 當請求帶有string時```https://d111111abcdef8.cloudfront.net/images/image.jpg?color=red&size=large```當中就可以設定當出現color=red時使用快取。順序前後以及字串大小寫會影響快取(先color再size或者先size再color)
(5)cookie

**Origin request settings** - 設定Request進來之後會將哪些導致Origin中，如果是選擇All，那麼就不會有快取，所有請求都會導向Origin。


### Response headers policy - optional

該處主要是對於CORS的設定[5][6][7][8]

以下將進行簡單的測試，主要是建立一個CORS resource server 以及consume server，並在consume server對resource請求，當沒有設定CORS的情況下會回傳```Error: Could not connect to Server1. [Errno 111] Connection refused```，以下將是測試內容。

1. CORS resource server

```
#!/bin/bash
python3 -m venv tutorial-env
source tutorial-env/bin/activate
pip3 install FLASK
pip3 install flask_cors
pip3 install httpx
```
- app.py
```
from flask import Flask
from flask_cors import CORS

app = Flask(__name__)
CORS(app)
CORS(app, resources={r"/.*": {"origins": ["http://1.2.3.4"]}})

@app.route('/')
def hello_world():
    return 'Hello, World! From CROS Origin'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```


2.consume server

```
#!/bin/bash
python3 -m venv tutorial-env
source tutorial-env/bin/activate
pip3 install FLASK
pip3 install httpx
```

app2.py
```
from flask import Flask
import httpx

app = Flask(__name__)

SERVER1_URL = "http://1.2.3.4:5000/"  # 替換為 Server1 的實際 IP 和端口

@app.route('/')def proxy_to_server1():
    try:
        with httpx.Client() as client:
            response = client.get(SERVER1_URL)

        if response.status_code == 200:
            return response.text
        else:
            return f"Error: Received status code {response.status_code} from Server1", 500
    except httpx.RequestError as e:
        return f"Error: Could not connect to Server1. {str(e)}", 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```


### Additional settings
* Smooth streaming
如果使用Microsoft Smooth Streaming時並且沒有使用IIS server就可以選擇此功能

* Field-level encryption
進行更近一步的資料加密

* Enable real-time logs
real-time log

### Function associations
除了上述的方法外，如果需要按照自己的需求設定request以及response的內容，那麼可以透過function做設定。設定時可以選擇CloudFront Functions或者是Lambda@Edge。
CloudFront function主要是透過JavaScript快速的以及簡易的回覆，如果需要較為複雜的回覆可以透過Node.js 或者 Python的Lambda@Edge。

1. 使用情境
- CloudFront function

(1) Cache key normalization – 將 HTTP request attributes 修改並建立optimal cache key以增加快取命中率

(2) Header manipulation – 對request 或者 response 的header進行修改

(3) URL redirects or rewrites – 根據request提供的不同訊息進行redirect

(4) Request authorization – 確保只有授權的用戶可以訪問內容。


- Lambda@Edge
(1) 需要幾毫秒或更長時間才能完成的函數(函數持續時間可至5秒)

(2) 需要可調式 CPU 或記憶體的功能

(3) 依賴第三方程式庫的函式 (包括 AWS SDK，以便與其他程式庫整合 AWS 服務)

(4) 需要網路存取才能使用外部服務進行處理的功能

(5) 需要文件系統訪問或訪問 HTTP 請求主體的功能




## Error pages - 
當發生錯誤時可以設定在不同HTTP error時對應的的回應error page以及可以根據需求設定error code。


## Invalidations
當請求成功後會將快取存在節點中，如果需要手動更新節點就可以使用此功能，透過指定位置將該位置的快取清除。

## 額外資料

### .NET環境建置
EC2 linux2
```
sudo rpm -Uvh https://packages.microsoft.com/config/centos/7/packages-microsoft-prod.rpm
sudo yum install dotnet-sdk-6.0 -y
sudo yum install libicu -y
export DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=1

dotnet new webapp -n MyWebApp
cd MyWebApp
vim Pages/Index.cshtml
```
將內容修改為
```html=
@page
@model IndexModel
@{
    ViewData["Title"] = "Home page";
}

<div class="text-center">
    <h1 class="display-4">Welcome to my AWS EC2 .NET Web App</h1>
    <p>This is a simple ASP.NET Core web application running on AWS EC2 Linux 2.</p>
</div>
```
執行.NET
```
dotnet run
```
修改接聽的port
```
vim Properties/launchSettings.json
```
修改以下內容
```
"applicationUrl": "http://0.0.0.0:80"
```
執行程式
```
dotnet run
```


### HTTP request

## 參考資料：
[1]AWS CloudFront說明
https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html

[2]AWS Best Practices for DDoS Resiliency
https://docs.aws.amazon.com/whitepapers/latest/aws-best-practices-ddos-resiliency/aws-best-practices-ddos-resiliency.html

[3]Athena for CloudFront log
https://docs.aws.amazon.com/athena/latest/ug/cloudfront-logs.html

[4]SignedURL 設定
https://ithelp.ithome.com.tw/articles/10328887?sc=rss.iron

[5]CORS
https://aws.amazon.com/tw/what-is/cross-origin-resource-sharing/

[6] CORS-2
https://developer.mozilla.org/zh-TW/docs/Web/HTTP/CORS

[7] CORS-3
https://www.shubo.io/what-is-cors/#google_vignette

[8] CORS-4
https://www.maxlist.xyz/2020/05/08/flask-cors/


