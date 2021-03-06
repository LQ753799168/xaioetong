# 基于Python实现小鹅通付费视频爬取        
　　很久都没在这个公众号进行更新了，分享一个最近折腾的东西吧。<br>
　　最近通过某个培训机构报名了一个付费课程，该课程的教学视频是通过小鹅通平台进行播放的。因为觉得这套课程讲的很不错，就想着把它搞下来，方便后期多次回看（视频有播放时效性）。一开始是设想着和小伙伴一人录播一个视频，分工合作，不过大家都懒，这个计划直接夭折了。后来又去淘宝问了下，发现有专门的商家是帮忙代下载的，声称登录账号后即可帮忙下载，本着勤（kou）俭（kou）持（sou）家（sou）的精神，就想自己动手爬下来。<br>
　　言归正传，接下来大致分享我爬取的整个分析过程。<br>
　　首先，这个课程是通过微信小程序登录小鹅通，在登录小鹅通后，点击我购买的课程中会看到所有的课程，打开其中一个课程，选中第一个课程内容进行播放。此时我们在fiddler中会发现时不时就有一个视频请求，如下图所示。<br>
![1650761870(1)](https://user-images.githubusercontent.com/35380099/164951143-c2278c8b-f3e1-42bb-b3df-1b1fa17a8487.jpg)

　　通过分析第一个视频请求的数据，我们可以较为清楚的看出每一次视频请求，返回的都是一个ts视频片段。如下图所示，请求链接中存在.ts的字样。<br>
![1650761888(1)](https://user-images.githubusercontent.com/35380099/164951151-102271ea-c623-49a9-af4f-6e5bb49bc989.jpg)

　　同时，我们可以在请求参数中看到start和end信息，几乎就能肯定这是一个视频片段，start和end代表这个片段在整个完整视频中的开始时间和结束时间。因此，我们可以大致猜测，这是基于m3u8索引文件，进行的分段式的播放。<br>
![1650761896(1)](https://user-images.githubusercontent.com/35380099/164951154-279bc48c-25ea-42be-84f1-154eb8117413.jpg)

　　基于这样的猜测，我们在该请求前仅找到了这样的一条请求信息。该请求是一个get请求，参数为m3u8，返回了一串字符串。<br>
![1650761907(1)](https://user-images.githubusercontent.com/35380099/164951160-5e5c1dce-d00f-4f5d-bb90-ccde9253d0da.jpg)

　　初步猜测，返回中的字符串就是我们寻找的m3u8索引信息，但是这样乱糟糟的一串字符，根本看不出什么有用的信息。对其进行一系列常见的解码操作，都没有得到有用的信息，其中进行base64解码时，还提示有非法字符。sad，此时我们的线索在此中断。<br>
　　由于m3u8文件是有固定的格式，所以我们尝试从网上找了一份明文的m3u8文件，通过这份文件进行各种加密尝试，我们发现其中base64的加密和我们的请求结果中获取到的字符串有极高的相似度。<br>
![1650761954(1)](https://user-images.githubusercontent.com/35380099/164951177-e6af9ce0-7101-42b7-9bc5-879402428293.jpg)
网上找的m3u8文件进行base64加密<br>


![image](https://user-images.githubusercontent.com/35380099/164951263-90fc700c-cbe0-4533-a37d-78e8841d63b8.png)
爬取到的m3u8加密数据<br>

　　通过简单比对即可断定，该网站对m3u8数据进行了base64的加密，但如果直接对爬取到的数据进行base64解码会提示编码错误。细心排查一下，我们就会发现，该加密字符串进行了一些特殊字符的替换，替换如下：<br>

特殊字符  | 被替换的字符
---- | ----- 
  @  | 1 
  \#  | 2 
  $  | 3 
  %  | 4 

　　将替换后的字符串再进行base64解密，即得到了我们明文m3u8文件。<br>
　　得到m3u8文件后，我们就需要弄清楚，在这个请求中带的那一串字符是怎么来的，又代表什么含义呢，直接将这个字符串在所有的请求中进行搜索，并没有找到相同的字符串，但根据以往的经验，我们可以猜测出，相关的信息肯定就在这次请求的前几条中。<br>

　　其中这条video/base_info的请求引起了我们的注意。在这条请求的返回数据中，存在video_urls和videoUrl的字段，但这字段也是加密过的，如下图所示。<br>
![1650762256(1)](https://user-images.githubusercontent.com/35380099/164951274-00c6ea45-eb23-4c94-a522-6d613e97dd24.jpg)


　　通过上面替换特殊字符的方式，我们还原了video_urls字段信息，如下图所示。其中encrypt字段和我们的m3u8请求的链接一模一样，故我们找到了这个请求地址的由来了。<br>
![1650762269(1)](https://user-images.githubusercontent.com/35380099/164951281-0e3b2bc7-fb31-4e9b-97a2-1cf388f8de4d.jpg)

　　此处顺带提一下，这个请求中的m3u8所带的参数是什么意思呢？它的组成也是很有意思的。它是由两个字符串构成的，如下所示。<br>
![1650762277(1)](https://user-images.githubusercontent.com/35380099/164951288-50fc368c-85a9-42a2-b6b7-c11110d578a7.jpg)

　　拆解后的第一部分字符串又暗藏着玄机，它是由两串子串交替出现，子串1为base64加密的一个网址，子串2是一句话，大致的意思可能是说小鹅通的视频加密是一个靓仔，是不是骚得很。<br>
　　再往前面的一些请求包括微信扫码登录，已购课程列表获取等等操作，此次就省略一万字了，不再赘述了，留给各位读者自己去研究吧，基本难度不大。<br>
　　当我们获取m3u8文件后，就可以逐个下载ts片段视频，随机播放一个ts片段，会发现，这个片段也是经过加密的，直接播放是行不通的，我们还需要逐个进行解密后才能播放。这边是通过这条请求发现的，在请求获取到m3u8文件后，又有一个get_video_key的请求了，如下图所示。<br>
![1650762285(1)](https://user-images.githubusercontent.com/35380099/164951295-967401f5-2029-4591-963d-12a9467372e6.jpg)

　　可以看出这个key也是加密的，而且有了密钥以后，我们对ts文件的加密方式也是一无所知的，此处再次省略一万字分析过程，直接说结论。这里的ts的加密采用了AES加密的CBC模式，iv偏移为0，key的密钥如上所示（每个课程的密钥是不一样），只要采用字节形式表示，无视所有乱码，直接通吃。因此这边的解密我们就不在话下了。对了这边的密钥获取的请求地址，是在m3u8文件中获取到的，具体可以查看解码后的m3u8文件。<br>
　　当我们逐个进行视频解密后，我们就可以通过ffmpeg的concat命令进行视频合并了。此次留个小思考，若直接通过ffmpeg进行视频合并，会发现合并后的视频在片段拼接处存在声音卡顿现象，像是数据丢失的导致的，而网址上播放的课程是没有该问题，小伙伴们可以思考下如何可以解决该问题。<br>
　　终于，万年一更的文章终于搞定了，如果觉得我的分享对你有收获的话，可以帮忙点赞分享一波，感谢我的小伙伴们。<br>
  ![1650763105](https://user-images.githubusercontent.com/35380099/164951572-5a028924-677f-4219-b036-af1aa946b97f.jpg)

