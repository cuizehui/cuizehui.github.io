---
layout:     post
title:      "爬虫-根据短信号码和短信内容爬取商家数据"
subtitle:   "基于爬虫技术的小工具"
date:       2022-03-14 11:58:00
author:     "Nela"
header-img: "img/post-bg-rwd.jpg"
tags:
- Python

---

## 简介

短信应用在使用时,会收到很多B2C商家发送的短信,那么如何识别他们是谁呢？

根据业务需要,本文介绍一种使用爬虫技术,通过B2C商家下发的信息,爬取商家信息（商家名称,官方网址,头像等）的一种方案。

## 方案概述

完成简介中所述需求,拆解为如下几个步骤。

1. 预处理短信号码和内容并**提取关键词**。
2. 根据关键词，国家地区等信息，**请求Google搜索**
3. 爬取Google搜索结果并保存html。
4. 分析Google搜索结果html提取当中URL,**将提取的URL和搜索的关键词进行匹配**
5. 获取匹配的URL爬取商家html文件并存储html,文件名使用url-md5值处理（充当URL管理器）,避免重复查询
6. 使用（网页解析器）**beautifulsoup**解析XML 获取商家信息
7. 使用Pandas和sql3工具将分析结果持久化。 
8. 增加Logging,记录运行时日志
9. todo 加入多线程读写

### 方案细节

上述方案，有几个关键的细节：

-  短信号码和内容如何提取关键词
-  重复请求Google会被Google限制访问，进行人机验证,如何处理
-  Google搜索结果URL很多,如何与关键字匹配。找出最佳匹配的URL（商家URL主页）
-  商家主页重定向问题,关键字主页是挂在某平台下（Twitter/faceBook）如何处理,等其他小问题。

## 爬虫简介

爬虫架构主要由五个部分组成，分别是调度器、URL管理器、网页下载器、网页解析器、应用程序（爬取的有价值数据）

- **调度器：**相当于一台电脑的CPU，主要负责调度URL管理器、下载器、解析器之间的协调工作。
- **URL管理器：**包括待爬取的URL地址和已爬取的URL地址，防止重复抓取URL和循环抓取URL，实现URL管理器主要用三种方式，通过内存、数据库、缓存数据库来实现。
- **网页下载器：**通过传入一个URL地址来下载网页，将网页转换成一个字符串，网页下载器有urllib2（Python官方基础模块）包括需要登录、代理、和cookie，requests(第三方包)
- **网页解析器：**将一个网页字符串进行解析，可以按照我们的要求来提取出我们有用的信息，也可以根据DOM树的解析方式来解析。网页解析器有正则表达式（直观，将网页转成字符串通过模糊匹配的方式来提取有价值的信息，当文档比较复杂的时候，该方法提取数据的时候就会非常的困难）、html.parser（Python自带的）、beautifulsoup（第三方插件，可以使用Python自带的html.parser进行解析，也可以使用lxml进行解析，相对于其他几种来说要强大一些）、lxml（第三方插件，可以解析 xml 和 HTML），html.parser 和 beautifulsoup 以及 lxml 都是以 DOM 树的方式进行解析的。

### 海外短信样本预处理

此部分为业务相关,对于海外市场B2C的短信市场,发送人ID分为3种 

 | senderId  |    content    |  country|
 | :---------------------: | :----------------------- | :------------------ |
 |JE-JioPay	 |Recharge of Rs. 15.0 is successful for your Jio number 7743060336.Entitlement: Benefits: Unlimited Data (1GB High Speed Data, thereafter unlimited at 64Kbps). Validity: Base Plans validityTransaction ID: 136957980597To view details of your current and upcoming plan, click https://www.jio.com/dl/my_plansTo share your recharge experience, click https://www.jio.com/en-in/jio-rech-exp-survey?custid=136957980597Dial 1991, to know your current balance, validity, plan details and for exciting recharge plans.|	en
 |888	|Layanan Internet Malam 12 GB Non Renewal berlaku s/d hari ini. Info paket lainnya, tekan *123*3 atau kunjungi https://my.smartfren.comInternet 100rb 30GB,ketik internet<spasi>vol100rb ke 123	|id	
 |JX-620016	|Need more data? Recharge your Jio number 7009117103 with 4G data voucher of Rs 51  and enjoy 3GB high speed data. Validity of this plan is as per your existing plan. Click www.jio.com/r/fbUSxtMpn to recharge. T&C apply|	en	
 
 [查阅资料](https://infofru.com/general/what-is-dm-am-vm-ad-lm-dz-prefix-in-sms-messages/) -前两位是运营商名称，非商家相关信息可去掉。
 
 senderID为纯英文，此时英文缩写辨识度较高,可直接去Google搜索。
 
 针对senderID为号码,则提取content中关键字,优先匹配短信内容中url,再次匹配标签,最后截取内容(可进行数据清洗聚类后)搜索截取内容
 
### 处理Google请求

解决问题二: 重复请求Google会被Google限制访问，进行人机验证,如何处理

针对单IP请求Google ,有以下方案可以实行

1. ip轮询:动态替换vpn代理地址
2. ua随机:动态替换UserAgent
3. domain随机:动态替换Google域名
4. 休眠:延迟请求

此处参考文章 并使用其开源代码[MagicGoogle](https://www.howie6879.cn/p/对于python抓取google搜索结果的一些了解/)


### Google搜索结果URL的分析与匹配

先看两组示例

示例一 ：关键字 AIRSND
    
     google搜索结果分析后爬取到如下url
    
    - 'https://maps.google.rs/maps?q=AIRSND&gbv=1&cr=countryEN&num=1&um=1&ie=UTF-8&sa=X&ved=0ahUKEwj4uojQ1872AhU6SPEDHUk8AGoQ_AUICCgB'
    - 'https://www.crunchbase.com/organization/airsnd'
    - 'https://support.google.com/websearch?p=ws_settings_location&hl=en-SG',
    - 'https://accounts.google.com/ServiceLogin?continue=https://www.google.rs/search%3Fq%3DAIRSND%26btnG%3DSearch%26gbv%3D1%26cr%3DcountryEN%26num%3D1&hl=en'
    - 'https://www.google.rs/preferences?hl=en-SG&fg=1&sa=X&ved=0ahUKEwj4uojQ1872AhU6SPEDHUk8AGoQ5fUCCCw'


示例二 ：关键字 KOTAKB
    
     google搜索结果分析后爬取到如下url
    
    - 'https://kotaku.com/'
    - 'https://www.google.com.gt/search?num=1&gbv=1&tbs=ctr:countryEN&ei=rcQxYreaMPPSmAWliKeoDw&q=KOTAKB&tbm=isch&sa=X&ved=2ahUKEwj3pNO6vsr2AhVzKaYKHSXECfUQ7Al6BAgCEAs'
    - 'https://zh.wikipedia.org/zh-tw/Kotaku', 'https://support.google.com/websearch?p=ws_settings_location&hl=zh-TW'
    - 'https://accounts.google.com/ServiceLogin?continue=https://www.google.com.gt/search%3Fq%3DKOTAKB%26btnG%3DSearch%26gbv%3D1%26cr%3DcountryEN%26num%3D1&hl=zh-TW'
    - 'https://www.google.com.gt/preferences?hl=zh-TW&fg=1&sa=X&ved=0ahUKEwj3pNO6vsr2AhVzKaYKHSXECfUQ5fUCCFQ'
   
获取URL较多，如何匹配出最佳URL进行爬取商家信息呢？

参考资料 有以下可选方案：
1. 使用余弦相似
2. 使用正则
3. Levenshtein 匹配库

由于很多关键词是在URL的path部分,例如商家公司可能归属于某个平台,或者其域名中没有关键字相关内容,关键字为某公司支线业务。

此时不能直接单纯匹配URL的hostName部分。

通过爬到的数据分析，目前我的方案是

1. 先过滤掉 hostName为www.google 、google.maps、support.google等Google推荐操作URL
2. 使用正则 全匹配字符顺序为关键字顺序的URL,此为优先级最高的匹配  
3. 使用Levenshtein库中的jaro算法 -当匹配度大于某个值则认为匹配上
4. 增加几个白名单列列表(facebook,ins,wiki)有规则可爬取的特定网站。处理那些上述方案都无法匹配的URL

```python 
    def is_url_match_key(self, key, url):
        match_key = "{1}.*".join(key.lower().replace("+", ""))
        match_obj = re.search(match_key, url, re.I)
        uri = urlparse(url)
        if uri.hostname.startswith("accounts.google") or uri.hostname.startswith(
                "maps.google") or uri.hostname.startswith("support.google") or uri.hostname.startswith("www.google"):
            return None

        if match_obj:
            return match_obj
        else:
            sim = Levenshtein.jaro(key.lower(), uri.hostname)
            log_print("match jaro %f" % sim)
            if sim > 0.5:
                return sim      
```

### 数据存储与处理

为方便数据分析，对爬取的html目前都做了本地存储,主要为两类文件 Google搜索页html和商家信息页html

如何减少相同关键字和URL重复请求次数并且便于查找信息呢？

在html文件存储至本地时,将其google搜索页的文件命名为"关键字.html",将商家信息页存储为"关键字_urlmd5值.html"。请求时先判断文件是否存在。

```python 
        shop_info.url = uri.geturl()
        md5hash = hashlib.md5(uri.geturl().encode('utf-8'))
        file_name = md5hash.hexdigest()
        shop_info_file_name = sender + "_" + file_name + "_info.html"
        if not self.fo.check_file_exist(shop_info_file_name):
            log_print("search_page : %s" % uri.geturl())
            shop_info_xml = self.sp.search_page(uri.geturl())
            if shop_info_xml:
                log_print("sava_result_to_file : %s" % shop_info_file_name)
                self.fo.sava_result_to_file(shop_info_file_name, shop_info_xml)

```

数据库使用pandas和sqlite3 可以很方便的与excel文件互相转换

### html信息提取

找到商家官网,其实此步骤已经需要人工核对并且提取相关信息了。但一些常规信息通常会存储至head中方便搜索引擎爬取,还有某些特征字段robots,此处需要分析html。不同网站通常不同

html示例截取部分
```html
    
    <!DOCTYPE html>
    <!--[if IE 9 ]>   <html class="no-js oldie ie9 ie" lang="en-US" > <![endif]-->
    <!--[if (gt IE 9)|!(IE)]><!--> <html class="no-js" lang="en-US" > <!--<![endif]-->
    <head>
            <meta charset="UTF-8" >
            <meta http-equiv="X-UA-Compatible" content="IE=edge">
            <!-- devices setting -->
            <meta name="viewport"   content="initial-scale=1,user-scalable=no,width=device-width">
    
    <!-- outputs by wp_head -->
    <meta name='robots' content='index, follow, max-image-preview:large, max-snippet:-1, max-video-preview:-1' />
    
        <!-- This site is optimized with the Yoast SEO Premium plugin v16.2 (Yoast SEO v18.3) - https://yoast.com/wordpress/plugins/seo/ -->
        <title>PSG VMS Recruiting Solution Saves Client $2.96M Annually - PSG Global Solutions</title>
        <link rel="canonical" href="https://psgglobalsolutions.com/case-study/vms-recruiting-case-study/" />
        <meta property="og:locale" content="en_US" />
        <meta property="og:type" content="article" />
        <meta property="og:title" content="PSG VMS Recruiting Solution Saves Client $2.96M Annually - PSG Global Solutions" />
        <meta property="og:description" content="Client Challenge Client was continuing to see more customers go to VMS recruiting solutions, which were resulting in compressed margins and made it more difficult to meet company profitability targets. Client wanted to drive higher volume lower complexity requisitions to lower cost full lifecycle recruiting resources. PSG Solution Program started with five (5) full time [&hellip;]" />
        <meta property="og:url" content="https://psgglobalsolutions.com/case-study/vms-recruiting-case-study/" />
        <meta property="og:site_name" content="PSG Global Solutions" />
        <meta property="article:publisher" content="https://www.facebook.com/psgglobalsolutions/" />
        <meta property="article:modified_time" content="2020-09-01T14:15:35+00:00" />
        <meta property="og:image" content="https://psgglobalsolutions.com/wp-content/uploads/2020/02/VMS-1-1.png" />
        <meta name="twitter:card" content="summary_large_image" />
        <meta name="twitter:label1" content="Est. reading time" />
        <meta name="twitter:data1" content="1 minute" />
        <script type="application/ld+json" class="yoast-schema-graph">{"@context":"https://schema.org","@graph":[{"@type":"Organization","@id":"https://psgglobalsolutions.com/#organization","name":"PSG Global Solutions","url":"https://psgglobalsolutions.com/","sameAs":["https://www.facebook.com/psgglobalsolutions/","https://www.instagram.com/psg_global/","https://www.linkedin.com/company/psg-global-solutions/"],"logo":{"@type":"ImageObject","@id":"https://psgglobalsolutions.com/#logo","inLanguage":"en-US","url":"https://psgglobalsolutions.com/wp-content/uploads/2020/12/PSG-New.png","contentUrl":"https://psgglobalsolutions.com/wp-content/uploads/2020/12/PSG-New.png","width":110,"height":110,"caption":"PSG Global Solutions"},"image":{"@id":"https://psgglobalsolutions.com/#logo"}},{"@type":"WebSite","@id":"https://psgglobalsolutions.com/#website","url":"https://psgglobalsolutions.com/","name":"PSG Global Solutions","description":"Offshore Recruiting Services","publisher":{"@id":"https://psgglobalsolutions.com/#organization"},"potentialAction":[{"@type":"SearchAction","target":{"@type":"EntryPoint","urlTemplate":"https://psgglobalsolutions.com/?s={search_term_string}"},"query-input":"required name=search_term_string"}],"inLanguage":"en-US"},{"@type":"ImageObject","@id":"https://psgglobalsolutions.com/case-study/vms-recruiting-case-study/#primaryimage","inLanguage":"en-US","url":"https://psgglobalsolutions.com/wp-content/uploads/2020/02/VMS-1-1.png","contentUrl":"https://psgglobalsolutions.com/wp-content/uploads/2020/02/VMS-1-1.png"},{"@type":"WebPage","@id":"https://psgglobalsolutions.com/case-study/vms-recruiting-case-study/#webpage","url":"https://psgglobalsolutions.com/case-study/vms-recruiting-case-study/","name":"PSG VMS Recruiting Solution Saves Client $2.96M Annually - PSG Global Solutions","isPartOf":{"@id":"https://psgglobalsolutions.com/#website"},"primaryImageOfPage":{"@id":"https://psgglobalsolutions.com/case-study/vms-recruiting-case-study/#primaryimage"},"datePublished":"2020-08-18T02:25:45+00:00","dateModified":"2020-09-01T14:15:35+00:00","breadcrumb":{"@id":"https://psgglobalsolutions.com/case-study/vms-recruiting-case-study/#breadcrumb"},"inLanguage":"en-US","potentialAction":[{"@type":"ReadAction","target":["https://psgglobalsolutions.com/case-study/vms-recruiting-case-study/"]}]},{"@type":"BreadcrumbList","@id":"https://psgglobalsolutions.com/case-study/vms-recruiting-case-study/#breadcrumb","itemListElement":[{"@type":"ListItem","position":1,"name":"Home","item":"https://psgglobalsolutions.com/"},{"@type":"ListItem","position":2,"name":"PSG VMS Recruiting Solution Saves Client $2.96M Annually"}]}]}</script>
        <!-- / Yoast SEO Premium plugin. -->
    
    
    <style type="text/css">.recentcomments a{display:inline !important;padding:0 !important;margin:0 !important;}</style><link rel="icon" href="https://psgglobalsolutions.com/wp-content/uploads/2020/01/PSG-New.png" sizes="32x32" />
    <link rel="icon" href="https://psgglobalsolutions.com/wp-content/uploads/2020/01/PSG-New.png" sizes="192x192" />
    <link rel="apple-touch-icon" href="https://psgglobalsolutions.com/wp-content/uploads/2020/01/PSG-New.png" />
    
            <!-- end wp_head -->
    </head>
    <body data-rsssl=1 class="case_study-template-default single single-case_study postid-7790 wp-custom-logo auxin-pro elementor-default elementor-template-full-width elementor-kit-354 elementor-page-7780 phlox-pro aux-dom-unready aux-full-width aux-resp aux-hd  aux-page-animation-off _auxels"  data-framed="">
    </body>
    </html>

```

解析html,使用BeautifulSoup此库,不详细介绍了,用到了下面一些API

```python 
    def deal_company_xml_data(self, xml_file_name, shop_info):
        log_print("deal_company_xml %s" % xml_file_name)
        contents = self.fo.read_file(xml_file_name)
        if not contents:
            log_print("not company contents")
            return False
            pass
        soup = BeautifulSoup(contents, 'lxml')

        # deal title
        if soup.head:
            title = ""
            if soup.title:
                title = soup.title.string

            title_property = soup.head.findAll(name="meta",
                                               attrs={"property": "title", "property": "og:title",
                                                      "property": "twitter:title"})
            for p in title_property:
                if p.attrs.get("content"):
                    title = p.attrs["content"]
            if title:
                log_print('title %s' + title)
                shop_info.shop_name = title

            # deal icon
            icon_link = ""
            icon = ""
            icon_property = soup.head.findAll(name="meta",
                                              attrs={"property": "og:image"})
            for p in icon_property:
                if p.attrs.get("content"):
                    icon = p.attrs["content"]
                if p.attrs.get("href"):
                    icon = p.attrs["href"]
            if icon:
                log_print('icon_property %s' + icon)
                shop_info.icon = icon
            icon_link = soup.head.findAll(name="link", attrs={"rel": "icon", "rel": "shortcut icon", "rel": "og:image"})
            for i in icon_link:
                log_print(i)
                if i.attrs["href"]:
                    icon_link = i.attrs["href"]
            if icon_link:
                shop_info.icon = icon_link
        return True
```

### 携带cookie爬取需要登录的网站

    todo

### 使用异步框架ruia的使用介绍

    todo

## 开源地址
    
    todo

## 参考

参考：

[https://www.zhihu.com/question/268204922](https://www.zhihu.com/question/268204922)

[https://www.howie6879.cn/p/对于python抓取google搜索结果的一些了解/](https://www.howie6879.cn/p/%E5%AF%B9%E4%BA%8Epython%E6%8A%93%E5%8F%96google%E6%90%9C%E7%B4%A2%E7%BB%93%E6%9E%9C%E7%9A%84%E4%B8%80%E4%BA%9B%E4%BA%86%E8%A7%A3/)

[https://geek-docs.com/python/python-tutorial/python-beautifulsoup.html#BeautifulSoup](https://geek-docs.com/python/python-tutorial/python-beautifulsoup.html#BeautifulSoup)

[https://infofru.com/general/what-is-dm-am-vm-ad-lm-dz-prefix-in-sms-messages/](https://infofru.com/general/what-is-dm-am-vm-ad-lm-dz-prefix-in-sms-messages/)