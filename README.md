# 项目概述

这个项目，主要帮助完成实盘交易的对接。

思路是：

客户端（往往是研究用的策略分析程序），会把挑选出来的股票，买信号，推给服务器；
或者把风控发现需要卖出的股票，推给服务器。

服务器接收到买卖信号，会先保存到服务器的sqlite数据库中（使用sqlite是为了避免数据库依赖），
然后等待地调度。

每天会有一个和开盘时间和交易日时间才工作的调度器，每隔10分钟会询问调度任务数据库，
如果发现有买卖信号，就去调用easytrader，去进行买卖交易。
目前只支持easytrader，将来可以支持更多的broker。
[easytrader](https://github.com/piginzoo/easytrader)我使用的是我自己的魔改版本，
毕竟丫都N年不更新了，不支持最新的同欢顺，喜欢的[自取](https://github.com/piginzoo/easytrader)。

另外，为了防止一些异常，我还创建了另外一个定时器，每天23:00去检查，
是否我的sqlite种的持仓信息，和easytrader上实盘的持仓信息是否一致，
如果不一致，就同步下来，并且剔除不在实盘上的仓位股票。
这个类似于财务中的对账，防止出现不一致的情况。

项目中还包含了一个notify模块，可以把一些异常、通知发送给企业微信和邮箱。
模仿了logging的级别，用于接受各类通知，了解系统运行的问题和情况。

*哈哈，我怎么觉得，我自己写了一个"策略易"出来啊。*

目前实现的有：
- 跟曹一起合作的Cao策略
- 重现Kim教父的量化策略

# 如何运行

0、安装各种包

```
pip install -r requirement.txt
```

1、安装easytrader

```
git clone https://github.com/piginzoo/easytrader
cd easytrader
python setup.py install 
```

2、安装同花顺软件，并配置路径

- 下载[同花顺](https://download.10jqka.com.cn/free/)，并安装
- 配置[conf/config.yml]，并修改其中的配置：
```yaml
# 下单服务器支持的券商信息
brokers:
        yinhe: # 银河证券
                uid: 'xxxxxxx'
                pwd: 'yyyyyyy'
                client_type: 'ths5.19'
                exe_path: 'c:\software\ths\xiadan.exe'
        mock: # 同花顺模拟炒股
                uid: 'xxxxxxx'
                pwd: 'yyyyyyy'
                client_type: 'universal_client'
                exe_path: 'c:\software\ths.mock\xiadan.exe'
```
其中不同券商的软件尽量分开（原因：银河证券登录框一旦激活，就无法切换到其他券商的登录框，所以要分开目录。）

3、启动服务器：
```batch
bin\server.bat
```

# 开发日志（一览）

这个用于记录一些比较tricky的开发日志


- 20220702
  - 加入了easytrader的集成，可以用它来做交易了，并对其进行了bug修复和扩展（项目已经4年未更新了，不适配5.19的同花顺xiadan.exe）    
  - 加入了web端实现，实现了在服务器端的一个调度：
    - 为了简便，使用sqlite（不用单独安装数据库）来记录所有的交易请求、交易日志、交易仓位
    - 实现了从客户端提交->保存到数据库->调度器读取任务进行买卖->归档到日志表和更新仓位表
    - 对各类异常进行处理，买入/卖出失败、重试、撤单
    - 实现了一个Web UI，可以查询各类信息，并且做了安全校验
    - 实现了企业微信、邮件的分级通知
  - 反复调试模拟盘，各种异常case测试，实现了一些集成测试类
- 20220706
    - 实现了一个仓位同步功能，每晚11:00启动一个调度器，来将真实仓位同步到逻辑仓位，防止遗留真实仓位，以及修正价格和股数信息，以真实仓位为准
    - 重新实现了WebUI，完全采用api方式+ajax-json方式了，不在使用页面方式了，主要是jinja模版太难用了
    - 重新修改UI显示，采用大按钮，布局更合理
    - 在WebUI中增加了删除任务的功能，用于删除那些僵尸任务：反复执行的（针对卖）
    - 实现了风控卖出，根据最大回撤找到股票，卖出；可以单独运行，也集成在了一键实盘信号获取的过程中
    - 实现了服务器端和客户端的代码分开，客户端尊崇tushare以.SH,.SZ结尾，服务器端使用标准代码6位无后缀
    - 加入了更惊喜的SIGNAL消息，统一了日志和通知，在客户端推送时也加入通知
    - 重写了买卖逻辑，分开成2个独立的job，增加了逻辑控制，更鲁棒，其中买在尝试多次后会自动删除归档；而卖则一直会尝试，重新挂单再卖，努力卖出去
    - 处理了没有获得委托单合同编号的遗漏情况，查到当日成交的同代码股票，就算成功，回抄委托单合同编号
- 20220729
  - 修正了卖出的时候，如果没获得entrust_no合同单，就会导致反复尝试，不会查单回填，的bug
  - 修正了双向仓位同步，以实际持仓为根本的bug
  - 加入了多账户支持，在查询的时候，可以查询不同券商仓位

# 开发日志（详细）

## 设计和实现回顾(2022.7.7)
终于上了模拟盘，就差最后一步上真实交易账户了，先跑跑看。
把之前的设计思路和回顾做一个简短的回顾，方便以后的review。

总的思路就是想干：
- 可以实现一个一键实盘信号发现和交易的环境。

细节：
- 上实盘的话，就得每天做一个回测，因为使用最长15个月的月均线，就取了3年内的数据
- 其他的指标，也都提前算好，而不是像之前回测的时候，现算，提高速度
- 还要做一个自动check数据是否是最新，决定回测当天还是前一天的细节逻辑
- 实盘跑策略，使用的是之前跑回测策略用的代码，复用了，区别在于忽略所有的日子，只剩下当天的日子
- 实盘后，会把出现的买交易信号，放到一个信号表中
- 还要去服务器上下载最新的持仓，按照买入日，计算最大回撤，决定是否要风控卖出与否
- 把这些信号都推送到服务器的任务表中，等待开盘去调度买入或者卖出
- 服务器实现了一个web服务器，可以查询持仓、待交易任务和交易日志，以及各种实盘软件上的信息
- 服务器上使用easytrader+同花顺，实现了实盘交易，easytrader是自己的fork的版本，修丁了很多错误
- 服务器上起了一个定时器，每隔10分钟，检查是否有待交易任务，从而触发买、卖操作
- 买卖操作是通过easytrader的UI机器人操作同花顺实现的，因为有验证码干扰，以及UI操作的滞后性，做了大量容错处理
- 买卖成功后，把最新价格、合同单号移入交易日志表，服务器上使用sqlite，方便数据库维护
- 交易成功与否、以及各种异常、以及各种信号出现，都会发送到邮件和企业微信群组，方便监控

总结：

遇到很多细节问题，都一一解决，一些感受和教训：
- easytrader非常不稳定，未来需要切换到其他通道（一创聚宽或者国信iQuant）
- 基本上实现了一个小框架，虽然不完善，但是完成了整个闭环，从策略研发到实盘交易
- 复盘模块，还没有开始写，未来考虑继续研发出来

## 设计和实现回顾(2022.7.30)

实际运行了一段时间了，还是发现一些细节问题，这里更新一下设计的调整和背后的意图：
- 推送到服务器的买卖任务，还是需要带着策略名，为将来接受更多的策略，以及回溯收益情况，做准备
- 修正了常见的错误，买入或者卖出的时候，未获得entrust_no合同单号，导致总是close不掉这个任务，
  原因是precheck的时候，对于可用股数=0的当做了未持股，所以卖出流程终止导致，去掉这个=0限制即可
- 持仓同步的时候，只考虑实盘=>逻辑持仓方向，未考虑逻辑持仓不在实盘内的情况，加入了这个反向检查
  
## EasyTrade和多账户问题解决(2022.8.4)
涉及到切换券商，之前没有特别测试，今天跑了一下发现了一些问题，并且，修正之：
- 在首页中，加入券商选项，可以查询不同券商的真实持仓、头寸等信息
- 发现切换账户登录还是有些问题，最好的办法就是提前把多个账户的交易软件都打开登录好
- 由于之前就发现切换到银河登陆界面，就无法切换回其他登录界面的问题，目前是模拟炒股（c:\software\ths.mock）和银河证券（c:\software\ths）的目录分开存放了
- 逻辑界面没有可以区分，没必要，他们会在显示的时候，增加一列
- 总之easytrader很不好用，唉，缺乏一个好的券商接口，真头疼

# 待实现需求（TODO List）
- [X] 在tradelog表中，增加策略字段，知道这只股票的交易完成是那个策略的功劳
- [ ] 拉下来交易日志，对策略进行评估，复盘模块 
- [ ] 手工完成交易的时候，应该去读一下持仓的买入价格 
