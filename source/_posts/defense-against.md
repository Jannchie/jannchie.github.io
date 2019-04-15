---
title: 贫民服务器如何防御攻击？如何反爬虫？
tags: [后端,Java,Spring Boot,CDN]
categories: 命悬一线的热部署
date: 2018-12-21
---

我上线的一个网站[BiliOB观测者](www.biliob.com)受到了基友的攻击。

攻击方式其实非常简单，写一个小爬虫，不间断地POST注册请求，我的网站就直接崩了。好在对方并不是真的怀有恶意，一天时间也只往我服务器里打了几十万条数据。至少后台的核心爬虫业务没有受到很严重的影响。不过确实导致了一天之内网页无法正常访问。

出现这种情况的原因其实在于我的防范措施几乎没有，并且服务器配置极低，单机也完全可以打爆。因此需要紧急上线防御方案。

<!--more-->

---

首先需要解决的问题是：停止攻击方继续往数据库中填充垃圾数据，所以我直接暂且关闭了这一接口。

对方的攻击手段就是一种类似于爬虫技术的方式。而反爬模块对于我这种资料查询型网站是必须的。我本身也写了很多爬虫，对于这种反爬技术也有一定的了解。而所有反爬手段，最令人头疼并且简单易行的就是封禁IP。

那么如何获得对方的IP呢？因为我使用了CDN内容分发网络，所有通过域名访问到服务器的都是CDN节点，所以Request的发起方都是CDN节点而不是真正的用户。不过CDN节点回源请求时，会在请求头中加入`x-forwarded-for`、`Proxy-Client-IP`、`WL-Proxy-Client-IP`之类的字段，这些字段中存放的就是所有发送方经由的IP。即使对方使用了代理服务器，也依然能够获取对方客户端的真实IP地址。

我的后端使用Java实现，构建了一个获取IP地址的工具类：

``` java
/**
 * 源代码来自互联网，被多人转载使用已经无法追溯来源。我经过修改后摘录如下
 * @author jannchie
 */
class IpUtil {
  private static final String UNKNOWN = "unknown";
  private static final String COMMA = ",";
  private static final Integer IP_LENGTH = 15;

  static String getIpAddress(HttpServletRequest request) {
    String ipAddress = null;
    try {
      ipAddress = request.getHeader("x-forwarded-for");
      if (ipAddress == null || ipAddress.length() == 0 || UNKNOWN.equalsIgnoreCase(ipAddress)) {
        ipAddress = request.getHeader("Proxy-Client-IP");
      }
      if (ipAddress == null || ipAddress.length() == 0 || UNKNOWN.equalsIgnoreCase(ipAddress)) {
        ipAddress = request.getHeader("WL-Proxy-Client-IP");
      }
      if (ipAddress == null || ipAddress.length() == 0 || UNKNOWN.equalsIgnoreCase(ipAddress)) {
        ipAddress = request.getRemoteAddr();
      }
      // 对于通过多个代理的情况，第一个IP为客户端真实IP,多个IP按照','分割
      if (ipAddress != null && ipAddress.length() > IP_LENGTH) {
        if (ipAddress.indexOf(COMMA) > 0) {
          ipAddress = ipAddress.substring(0, ipAddress.indexOf(","));
        }
      }
    } catch (Exception e) {
      ipAddress = "";
    }
    return ipAddress;
  }
}
```

通过上述代码我们就能获得原始IP地址。我们接下来要做的就是获得这些IP地址的访问频率，酌情加入黑名单。

---

现在我们需要实现一个自定义的过滤器，在所有Controller执行之前，跑一遍过滤器的代码。这个过滤器需要记录所有用户（IP）的访问次数，如果在时限内超出一定次数，就直接加入黑名单。

在我的设计中，我把GET请求和其他请求分开计算，GET请求允许一秒GET5次，而POST只允许一秒一次，如果超出则会加入黑名单，黑名单由MongoDB维护。

过滤器维护一个计数器，计数器统计每个IP的访问次数。这个计数器每分钟会重置一次，使用的是spring boot的Schedule注解。

具体代码如下：

``` java
/**
 * @author jannchie
 */
@Component
public class IpHandlerInterceptor implements HandlerInterceptor {
  private static final Logger logger = LogManager.getLogger(IpHandlerInterceptor.class);
  private static final Integer MAX_CUD_IN_MINUTE = 60;
  private static final Integer MAX_R_IN_MINUTE = 360;
  private static final String IP = "ip";
  private static HashMap<String, Integer> blackIP = new HashMap<>(256);
  private final MongoTemplate mongoTemplate;
  public IpHandlerInterceptor(MongoTemplate mongoTemplate) {
    this.mongoTemplate = mongoTemplate;
  }
  /**
   * controller 执行之前调用
   */
  @Override
  public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
      throws Exception {
    String ip = IpUtil.getIpAddress(request);
    // 在黑名单中直接处决
    if (mongoTemplate.exists(query(where(IP).is(ip)), Blacklist.class)) {
      returnJson(response);
      return false;
    }
    Integer limitCount = Integer.MAX_VALUE;
    if (Objects.equals(request.getMethod(), HttpMethod.DELETE.name()) || Objects.equals(request.getMethod(), HttpMethod.POST.name()) || Objects.equals(request.getMethod(), HttpMethod.PATCH.name()) || Objects.equals(request.getMethod(), HttpMethod.PUT.name())) {
      // POST或PATCH或PUT或DELETE方法
      limitCount = MAX_CUD_IN_MINUTE;
    } else if (Objects.equals(request.getMethod(), HttpMethod.GET.name())) {
      // GET方法
      limitCount = MAX_R_IN_MINUTE;
    }
    if (blackIP.containsKey(ip)) {
      Integer newCount = blackIP.get(ip) + 1;
      blackIP.put(ip, newCount);
      if (newCount > limitCount) {
        mongoTemplate.insert(new Blacklist(ip), "blacklist");
        logger.info(ip);
        blackIP.remove(ip);
        response.setStatus(HttpStatus.FORBIDDEN.value());
        returnJson(response);
        return false;
      }
    } else {
      blackIP.put(ip, 1);
    }
    return true;
  }

  @Scheduled(cron = "0 0/1 * * * ?")
  public void refresh() {
    blackIP = new HashMap<>(256);
  }

  private void returnJson(HttpServletResponse response) throws Exception {
    PrintWriter writer = null;
    response.setCharacterEncoding("UTF-8");
    response.setContentType("application/json; charset=utf-8");
    try {
      writer = response.getWriter();
      writer.print("{\"msg\":\"YOU HAVE BEEN CAUGHT.\"}");

    } catch (IOException e) {
      logger.error("response error", e);
    } finally {
      if (writer != null) {
        writer.close();
      }
    }
  }
}
```

这样我们的过滤器就已经写好，现在能够正确地封禁IP了。但是，这样仍然不能完全阻止对方发送请求。因为只要对方一直发送请求，服务端就会不停地跑这个封禁流程。由于服务器比较弱鸡，一台家用的电脑几十行脚本就能跑爆，其他人仍然无法正常访问，仅仅如此果然还是不行的。

---

由于我采用的架构是：全国CDN节点 -> 我的服务器。其中的瓶颈在于我的服务器同时能处理的请求太少了。在这种情况下，我希望CDN节点能够直接过滤掉被封禁的IP。

我使用的是阿里云的CDN，他提供了黑名单的服务，在控制台上可以手动添加。然而我需要的是自动加入黑名单的功能。在这种情况下，我们就需要使用阿里的OpenApi，自己写个脚本实现这一功能。

脚本原理很简单，只需要定期地检查MongoDB中的黑名单集合，替换到阿里云CDN的黑名单列表即可。

而操作CDN的签名机制比较复杂，所幸从[阿里云CDN官方文档](https://help.aliyun.com/document_detail/27149.html)可以得到Python版本签名机制调用代码。

而我们只要写一段定期从数据库里得到黑名单数据的脚本即可：

``` python
#!/usr/bin/python3.6
# -*- coding:utf-8 -*-
# import ......

def run_threaded(job_func):
     job_thread = threading.Thread(target=job_func)
     job_thread.start()

def checkBlacklist():
    ip = coll.find()
    blockips=""
    for each in ip:
        print(each['ip'])
        blockips += each['ip'] + ','
    blockips = blockips[:-1]
    command = "python2 cdn.py Action=SetIpBlackListConfig BlockIps={blockips} DomainName=www.biliob.com,biliob.com".format(blockips=blockips)
    print(command)
    os.system(command)

schedule.every().minute.do(run_threaded,checkBlacklist)
while True:
    schedule.run_pending()
    time.sleep(60)
```

这样，恶意的IP就能够直接从CDN阶段就被Ban掉，不会回源到我的服务器了。

当然，这种做法只能防御住基本的攻击和基本的反爬手段，如果对方使用代理池，或者是真正的黑客，拥有数量茫茫多的肉鸡，那么我也无能为力。攻击是需要成本的，构建一个可用的ip池其实没有那么简单，但万一真的有人丧心病狂花费大量资源攻击，也只能认命。毕竟网络防御是一个提高对方攻击成本的过程，我所能做的，只是尽可能让攻击的门槛提高，只要攻击的成本比我服务器的资源价值更高，就能完成防御的目的。

最后，实名DISS一下攻击我的基友:**虽然我家没有扫地机器人，但这不是你往我家里丢一天纸屑的理由！！!DAMN YOU!**