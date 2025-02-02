[QMT](http://docs.thinktrader.net/vip/pages/5841ba/)是迅驰的量化回测和实盘工具，是目前最适合个人的实盘交易软件，也应该是券商里面支持最多的一个平台。比起ptrade、掘金量化等要好一些。可以参考：[量化极客 - 盘点国内实盘股票量化交易平台有哪些?](https://www.bilibili.com/video/BV1mg411Q7r3)。

我其实不需要她的回测，我自己写了一个回测框架，之前是用的backtrader，后来还是觉得不顺手，索性就自己写了一个。所以，我不需要它的回测，我只对它的实盘交易感兴趣。

各个券商对QMT的支持力度不一，券商整体碍于监管，是不主动推QMT的，而且很多家都有资金门槛限制，我则选择了门槛比较低、最高调宣传的国信iquant，它们家也是包装了QMT，但是自己还立了个iquant的品牌对外主推，所以，我便选择它作为我的实盘交易软件。不过他们家不提供极速模式。

顺道吐槽一下，国内券商、业界，都对个人量化藏着掖着，不给很好实盘交易接口，主要是监管层就不太支持。

于是我在国信开了户，下载了iquant，在我的windows电脑中把它安装部署起来，开始尝试写一个我的实盘策略。这一路走下来，发现这东西，坑实在是多，我不由地想吐槽，为何就不能把一件事做到100分，非要做到80分，让别人难受呢？

# python环境支持

估计QMT是为了降低门槛，不用你单独装一个python环境，自带了一个python3.6.8，可是你想想，既然你都要做python的量化开发了，如果连python环境都搞不定，还搞个毛啊？！可是，他们偏偏就这么认为了，非集成了一个内嵌的python3.6.8进去。

嵌入也就罢了，不提供对外的任何pip、python解释器的入口，只有一个pythonw。那么问题来了，如果你要装一些其他的pip包，咋办? 答案是，没办法！靠！或者，你按照它的pdf文档中，用一种极其蹩脚的方法，用其他python替换掉它的3.6.8，文档写的很山寨，我反正没敢尝试。

我找到网上说的一个办法，额外装了一个python3.6.8（有人说，pytnon3.7,3.8都可以，我实测了python3.8，还是有兼容性问题），然后用这个新装的python/pip，来安装pip包，只是在pip install的时候，要制定一个target参数，指向iquant的python3.6.8的包目录，举个例子：

```jsx
pip install tushare --target c:\iquant\bin.x64\Lib\site-packages\
```

三方包解决了，我又考虑，是不是可以复用我自己的代码呢？

比如我在pycharm中自己调试好程序，然后，我把它安装成一个本地包，然后，我在iquant里面import，直接用就好了。

于是，我在我的项目中，利用python的`setuptools`，写了setup.py，然后运行，发现，不像pip，它无法支持target这个参数，那他支持啥？查来查去，发现要用prefix参数，来指定安装目录。

但是，但是，要先把这个iquant的package目录加入到环境变量PYATHONPATH中，才允许我安装到这个目录，但是，我把这个目录设置到windows的系统的环境变量PYATHONPATH后，iquant就会模型奇妙的闪退，最后没办法，我在每次运行的时候，都手动来设置这个环境变量。

```jsx
set PYTHONPATH=c:\iquant\bin.x64\lib\site-packages\
python setup.py install --prefix=c:\iquant\bin.x64\
python setup.py
```

最后，我发现安装完的egg文件，iquant还无法识别，必须要解压缩的方式，egg是zip的格式，是不行的。所以，我在setup.py中不得不加入了`zip_safe=False。`

最后，终于可以开心的在pycharm中写好、调试好程序，然后再在iquant/QMT中，import之。也可以pip安装第三方包了。

# 实盘和回测区别

## handlebar的处理在回测和实盘的区别：

摘自官方文档：

> 在**[QMT](https://www.qmtptrade.com/?tag=qmt)**中，回测handlebar和实盘handlebar的区别，回测handlebar是逐bar运行的，比如我们在回测参数中选取了15分钟作为回测的频率，则handlebar在回测模式下，会在每个15分钟的k线上运行一次，而在实盘模型下，无论是选择10分钟，或者15分钟，还是日线，其实都是每个tick运行一次handlebar，但是信号是根据每个bar的最后一个tick来判断，如果触发信号，则在下一根bar的第一个tick下单，如果我们把passorder函数的快速下单参数写为1立即下单，则我们的程序就变成了逐tick运行。
> 

在回测时候，确实是每个设置的周期（bar）触发一次，比如你是日频，就每日触发一次；如果是小时频度，就每小时触发一次。

但是，在实盘的时候，就有很大的差异了：

- 在实盘的时候，每个tick（3秒）都会调用一次handlebar，我观察了，确实是，每3秒，这个函数就会进入一次。我觉得这个函数应该叫handletick，呵呵。
- 在这个handlebar函数里，如果你创建了一个买单，不会立刻触发，而是要等到，下一个bar的第一个tick才会生效，（比如一个bar是1分钟，1个tick是3秒）。所以，如果你1个bar（比如1分钟）创建了一堆订单，它也会只触发一个订单，也就是你上一分钟里下过的最后一个订单。

## 不同情形下的handlerbar的细微区别

上面提到了回测和实盘。

其实，在QMT中还有个在编辑状态下的“运行”，我也不知道这个运行是什么意思，我理解，有点类似于策略部署后的模拟运行。但是，又和实盘下那个模拟交易有些区别：

下面是它们之间的细微差别：

- 如果是回测，handlebar会从真正的回测日期开始，
- 但是如果是运行，就会从一个诡异的20150104开始，
- 如果是实盘/模拟盘交易，就会从20221227（当前是20230112）开始，实盘和模拟盘都是如此，这个时候必须要用ContextInfo.is_last_bar()来做一个判断，直接跳到最后一个bar，否则，会从20221227日按照1分钟的级别，运行2700多个bar。

总之，实盘的时候，有很多细微的坑，吐槽无力中了……

# 关于买卖

我的策略中，创建了一个买单，但是死活不触发信号，也就是在”策略交易>策略信号”中，看不到下的买单。

我尝试了3个下单函数，order_value, order_lots和passorder，都无法触发。我百思不得其解，我还怀疑是不是实盘不触发，就切到实盘，也不触发。

绝望之余，我去对比了别人的代码，发现必须要设置ContextInfo.buy=True才可以。靠！我没有看到一个地方提到这个的，甚至文档中也没有强调。

于是，我加上了这个设置，信号立刻就出现了。

我还遇到另外一个诡异现象：

我在bar的第一个tick就下单，剩下的tick里面就不下单了，比如我的bar是1分钟，tick是3秒，我有20个tick，我只在第一个tick调用passorder下单了，其他的tick都不下单了。结果，死活无法触发买卖信号。貌似只有在最后一个tick的时候，你下单，才会被捕捉到。其他的tick下单（就是调用passorder），都不会触发。可是QMT缺没有提供任何API，来判断这个tick是bar的最后一个tick，却提供了第一个tick的标志，搞的我哭笑不得。

我后来的解决办法是，直接在passorder中，设置`quickTrade=1`

>passorder是对最后一根K线完全走完后生成的模型信号在下一根K线的第一个tick数据来时触发下单交易；采用quickTrade参数设置为1时，非历史bar上执行时（ContextInfo.is_last_bar()为True），只要策略模型中调用到就触发下单交易。quickTrade参数设置为2时，不判断bar状态，只要策略模型中调用到就触发下单交易，历史bar上也能触发下单，请谨慎使用。

这样，当前tick，就触发下单了，不用等到下一个bar的第一个tick才生效了。

当然，我觉得，如果你在每个tick中都创建买单也可以，因为之后最后一个tick会生效，这个设计，我觉得太垃圾了，根本原因是不应该每个tick都调用一次handlerbar，而是应该最后一个tick才调用，或者更精细点，设计成bar_begin和bar_end，2个函数多好。好了，不吐槽了。

另外，在模拟交易的时候，策略信号出现后，并不会产生委托单，真正的委托单要到真的下单才会出现的，所以后续的deal_callback还不会触发。

# 极速模式

我看[这个视频](https://www.bilibili.com/video/BV1ex4y1G7wT)说，QMT有个极速模式，可以在登录界面勾选上，会启动一个非常简洁的下单界面，只会完成下单的任务，那些烂七八糟的行情、策略啥的都没了。

然后，就可以用xtquant这个包，直接触发这个极速模式的界面的交易了。我理解，xtquant是这个交易界面的一个操控api，你不用登录，因为极速模式的时候已经登录了，你用xtquant这个python包，可以查询数据、下单啥的，都可以，但是最终都是靠极速模式的界面完成和体现的，这点，有点像IB的API（都是需要你先登录到IB的TWS软件客户端，然后用python api操纵TWS下单的）。

我很喜欢这个极速模式（或者叫简洁模式），可惜，国信的iquant，根本都没有这个选项，我去了它的package目录，看到了xtquant这个package，但是确实没有启动极速模式的地方，我怀疑，是国信把这个功能封掉了。

至于为何要封掉这样一个不错的接口，我觉得，可能是因为这个接口给我们这种普通人太多的权限和功能，可能不合规。