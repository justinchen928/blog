---
title: "透過AWS SQS + lambda完成大量發送服務"
date: 2020-01-03T08:05:10+08:00
author: justinchen
type: post
slug: message-delivery-service-with-sqs-and-lambda
share_img: https://miro.medium.com/v2/resize:fit:1400/format:webp/1*heKgKewNwN3wIqY8SO8U6g.png
categories:
  - aws
tags: ["aws","lambda","sqs"]
---

# 大量發送服務 AWS SQS + lambda

![Cover](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*heKgKewNwN3wIqY8SO8U6g.png)


年初時，接到一個需求是需要在短時間內發送大量的訊息，以公司現有的架構(PHP)要做到這件事就是要提早幾個小時先產出要發送的資料和名單，然後另外一支程式負責抓出來一筆一筆寄出去。
<!--more-->
這樣的發法如果是發送無時效性的內容是不會有問題，但如果發送的內容是限定當天早上才有的折扣碼，如：
> 早安，這coupon送你，快來用！

當全部發完時可能都變午安或晚安了，這很明顯不可行。

### 需求

那到底要怎麼做呢？來分析一下我們的需求：我們希望能夠在預定的時間，把事先編輯好的訊息在短時間(可能是幾分鐘，甚至是幾秒鐘)內發送給一兩百萬個用戶。

所以我們可以怎麼做？如果要發送給所有用戶的訊息內容是一樣的，我們可以先把所有的訊息內容和用戶的資訊都事先準備好放在一個地方，當時間到的時候，有個程式就只要把資料抓出來送出去就好。

然而只有一支程式明顯不夠，最好是有一百支、一千支甚至是一萬支程式同時來做這件事，這樣就算有一千萬筆也只要幾秒鐘就可以完成了！

所以我們需要兩個條件：

1. 可以存放準備好的資料的地方(queue)。

1. 一支會影分身的程式(worker)，可以從存放資料的地方抓出資料送出去。

當然最好是可以在預定的時間，放資料的同時就一起發送。

最後我們找到AWS的服務：Simple Queue Service(SQS)以及Lambda完全符合我們的需求。

### 程式架構

先來看一下架構圖：

![](https://cdn-images-1.medium.com/max/2000/1*E1jtwZBES8Sx-r6Znkkbdw.jpeg)

首先會有一隻排程程式當時間到的時候會把需要發送的內容和目標塞到SQS的Queue裡，當有Messages塞到Queue裡時會啟動我們寫好的Lambda Functions，Lambda Functions會根據SQS event會自動擴展到需要的數量，接著lambda function會將訊息發送給用戶。

## AWS SQS

首先我們新增一個名為 testQueue 的AWS SQS Standard Queue，這部分[AWS SQSDeveloperGuide](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html)寫得很詳細，這邊就不贅述了。

接下來我們要塞發送的資料到這個Queue裡面，這部分我們透過Node.js實作，你可以透過npm安裝 aws-sdk

    npm install aws-sdk

這裡提供一個簡單的範例

<iframe src="https://medium.com/media/f181cfc4fe110f23f2a7cda5ff6d5f96" frameborder=0></iframe>

* QueueUrl會是從 SQS Management Console拿到的Queue url

* MessageBody 是一個大小在256KB以下的字串

接著直接在專案下執行：

    node sqs.js

成功執行的response會長這樣：

    {
      ResponseMetadata: { RequestId: '171f82e8-939c-576c-a8fc-d469ea5ea676' },
      MD5OfMessageBody: '07cb7a171fb4297016d4a7294a0f25f9',
      MessageId: '7afcbc8a-3aaf-45c5-a663-fc5714c035a1'
    }

這時候去AWS SQS的管理頁面可以看到在 Messages Available 從 0變成 1 了。

![成功insert訊息到queue裡](https://cdn-images-1.medium.com/max/2000/1*-7gccbRW43uPho_i_DvjVQ.png)*成功insert訊息到queue裡*

在寄送訊息的 params 還有很多參數可以設定，可以到[***文件](https://docs.aws.amazon.com/zh_tw/AWSJavaScriptSDK/latest/AWS/SQS.html#sendMessage-property)***看看。

## AWS Lambda

新增一個lambda function，選擇從最簡單的Hello World開始，這邊透過Node.js來實作， Permissions 的部分你可以新建一個role或用現有的都可以。

![](https://cdn-images-1.medium.com/max/4848/1*h6r6O45OL2ig5QYIcTivQw.png)

點選 Create function 後你會來到 testLambda 的頁面。

![](https://cdn-images-1.medium.com/max/3322/1*Q6IIeIVsD6f9d4kBRobpbw.png)

接著我們要選擇一個服務(就是剛剛新增的SQS)來啟動這個lambda function，點選 Add trigger 。

![](https://cdn-images-1.medium.com/max/2000/1*yNzV_Zv8mGLzkFvplzFalQ.png)

可以看到有很多服務都可以trigger lambda functions，選取SQS後下方的 SQS queue 會列出目前有哪些queue可以選，一樣選擇我們剛剛新增的 testQueue ，接著 Batch size選擇10筆，你可以選擇是不是要馬上透過SQS啟動lambda function，這邊直接選擇啟動。

選擇透過SQS啟動lambda function後就是要寫發送SQS的資料的程式

![](https://cdn-images-1.medium.com/max/3260/1*8MLK6vP7A0QHq0fu7iOJog.png)

回到剛剛的lambda頁面，往下滾動可以看到 Function Code的部分，你可以直接在這邊寫程式碼。這邊我們只先把收到得 event 印出來，不做任何事。

<iframe src="https://medium.com/media/c6fd5cefecd91e34183e066b7c3f7686" frameborder=0></iframe>

接著我們用剛剛寫的 sqs.js 塞一筆資料到 testQueue 裡。接著我們到 testLambda 的頁面點選 Monitoring 可以看到這個 lambda function的一些數據，在下方有lambda執行的log，可以連結到cloudWatch，就可以看到我們剛剛塞的訊息 test message body ，以及執行時間和memory使用量。

![](https://cdn-images-1.medium.com/max/2722/1*tAHS333SBbjXTV0nYZyc7A.png)

lambda是根據請求數量和程式碼執行持續時間計算，價格根據memory使用量而不同，相關的內容都可以在lambda頁面設定。詳細請看 [AWS Lambda定價](https://aws.amazon.com/tw/lambda/pricing/)。

當Lambda成功執行完後，會刪除queue裡的訊息。但如果lambda function在執行時有錯誤，Lambda並不會刪除該訊息，訊息會重新出現在queue裡，這點要特別注意。

另外在開發lambda function時會需要SQS發一個event給lambda function，我們才會知道lambda function是否有照我們預期的執行。然而這樣每次測試都需要塞一筆訊息到SQS，有點麻煩。

可以點選lambda function頁的右上角 Test 然後選擇 Configure Test Event 來設定一個 SQS 的測試 event，之後改完lambda function後按 Save 儲存之後直接點 Test 就會直接發一個event給lambda function。開發速度快上許多。

![](https://cdn-images-1.medium.com/max/2000/1*k_ACEVsop1M0ldhDPIMa4w.png)

如果需要在lambda使用package，可以查看[ AWS Lambda Deployment Package in Node.js](https://docs.aws.amazon.com/lambda/latest/dg/nodejs-create-deployment-pkg.html)。

### 參考

* [AWS SQSDeveloperGuide](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html)

* [AWS Nodejs SDK](https://aws.amazon.com/tw/sdk-for-node-js/)
