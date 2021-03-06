.. _quickstart:

快速入门
=================================================
希望快速开始使用 TqSdk?  本页面将介绍如何开始使用 TqSdk.

首先, 请确认:

* TqSdk 已经 :ref:`安装成功 <install>`
* TqSdk 已经更新到 :ref:`最新版本 <version>`

注意: TqSdk 使用了 python3 的原生协程和异步通讯库 asyncio，部分 IDE 不支持 asyncio，例如:

* spyder: 详见 https://github.com/spyder-ide/spyder/issues/7096
* jupyter: 详见 https://github.com/jupyter/notebook/issues/3397

可以直接运行示例代码，或使用支持 asyncio 的 IDE (例如: pycharm)

让我们从一个简单的例子开始


.. _quickstart_1:

获取实时行情数据
-------------------------------------------------
通过 TqSdk 获取实时行情数据是很容易的.

首先, 必须引入 tqsdk 模块::

    from tqsdk import TqApi, TqSim

创建API实例. 需要指定交易帐号. 如果使用API自带的模拟功能可以指定为 TqSim::

    api = TqApi(TqSim())

获得上期所 cu1812 合约的行情引用::

    quote = api.get_quote("SHFE.cu1812")

现在, 我们获得了一个对象 quote. 这个对象总是指向 SHFE.cu1812 合约的最新行情.
注意: api.get_quote 调用后会马上返回, 这个时候 q 里面还没有数据. 要等待行情数据收到, 我们还需要一些代码::

    while True:
        api.wait_update()
        print (quote["datetime"], quote["last_price"])

api.wait_update() 是一个阻塞函数, 程序在这行上等待, 直到收到数据包才返回. 在收到数据包后, 我们可以通过 quote 的各个 key 获得相应的行情字段

上面这个例子的完整程序请见 `t10.py <https://github.com/shinnytech/tqsdk-python/blob/master/tqsdk/demo/tutorial/t10.py>`_. 你也可以在自己电脑python安装目录的 site_packages/tqsdk/demo 下找到它

很简单, 对吗? 到这里, 你已经了解用 TqSdk 开发程序的几个关键点:

* 创建 TqApi 实例
* 用 api.get_quote() 或 其它函数获取数据引用对象
* 在循环中用 api.wait_update() 等待数据包.
* 收到数据包以后通过引用对象获得所需数据

下面我们将继续介绍 TqSdk 更多的功能. 无论使用哪个功能函数, 都遵循上面的结构.


.. _quickstart_2:

使用K线数据
-------------------------------------------------
你很可能会需要合约的K线数据. 在TqSdk中, 你可以很方便的获得K线数据. 我们来请求 cu1812 合约的10秒线::

    klines = api.get_kline_serial("SHFE.cu1812", 10)

跟 api.get_quote() 一样, api.get_kline_serial() 也是返回K线序列的引用对象. K线序列数据也会跟实时行情一起同步自动更新. 你也同样需要用 api.wait_update() 等待数据刷新.

一旦k线数据收到, 你可以通过 klines 访问 k线数据::

    while True:
        api.wait_update()
        print("最后一根K线收盘价", klines.close[-1])

这部分的完整示例程序请见 `t30.py <https://github.com/shinnytech/tqsdk-python/blob/master/tqsdk/demo/tutorial/t30.py>`_.

到这里为止, 你已经知道了如何获取实时行情和K线数据, 下面一段将介绍如何访问你的交易账户并发送交易指令


.. _quickstart_3:

交易账户, 下单/撤单
-------------------------------------------------
要获得你的账户资金情况, 可以请求一个资金账户引用对象::

    account = api.get_account()

要获得你交易账户中某个合约的持仓情况, 可以请求一个持仓引用对象::

    position = api.get_position("DCE.m1901")

与行情数据一样, 它们也通过 api.wait_update() 获得更新, 你也同样可以访问它们的成员变量::

    print("可用资金: %.2f" % (account["available"]))
    print("今多头: %d 手" % (position["volume_long_today"]))

要在交易账户中发出一个委托单, 使用 api.insert_order() 函数::

    order = api.insert_order(symbol="DCE.m1901", direction="BUY", offset="OPEN", volume=5)

这个函数调用后会立即返回, order 是一个指向此委托单的引用对象, 你总是可以通过它的成员变量来了解委托单的最新状态::

    print("委托单状态: %s, 已成交: %d 手" % (order["status"], order["volume_orign"] - order["volume_left"]))

要撤销一个委托单, 使用 api.cancel_order() 函数::

    api.cancel_order(order)

这部分的完整示例程序请见 `t40.py <https://github.com/shinnytech/tqsdk-python/blob/master/tqsdk/demo/tutorial/t40.py>`_.

到这里为止, 我们已经掌握了 TqSdk 中行情和交易相关功能的基本使用. 我们将在下一节中, 组合使用它们, 创建一个自动交易程序


.. _quickstart_4:

构建一个自动交易程序
-------------------------------------------------
在这一节中, 我们将创建一个简单的自动交易程序: 每当行情最新价高于最近15分钟均价时, 开仓买进. 这个程序是这样的::

    klines = api.get_kline_serial("DCE.m1901", 60)
    while True:
        api.wait_update()
        if api.is_changing(klines):
            ma = sum(klines.close[-15:])/15
            print("最新价", klines.close[-1], "MA", ma)
            if klines.close[-1] > ma:
                print("最新价大于MA: 市价开仓")
                api.insert_order(symbol="DCE.m1901", direction="BUY", offset="OPEN", volume=5)

上面的代码中出现了一个新函数 api.is_changing(). 这个函数用于判定指定对象是否在最近一次 wait_update 中被更新.

这部分的完整示例程序请见 `t60.py <https://github.com/shinnytech/tqsdk-python/blob/master/tqsdk/demo/tutorial/t60.py>`_.


.. _quickstart_5:

按照目标持仓自动交易
-------------------------------------------------
在某些场景中, 我们可能会发现, 自己写代码管理下单撤单是一件很麻烦的事情. 在这种情况下, 你可以使用 :ref:`target_position` 机制. 你只需要指定账户中预期应有的持仓手数, TqSdk 会自动通过一系列指令调整仓位直到达成目标. 请看例子::


    # 创建 rb1810 的目标持仓 task，该 task 负责调整 rb1810 的仓位到指定的目标仓位
    target_pos_near = TargetPosTask(api, "SHFE.rb1810")
    # 创建 rb1901 的目标持仓 task，该 task 负责调整 rb1901 的仓位到指定的目标仓位
    target_pos_deferred = TargetPosTask(api, "SHFE.rb1901")

    while True:
        api.wait_update()
        if api.is_changing(quote_near) or api.is_changing(quote_deferred):
            spread = quote_near["last_price"] - quote_deferred["last_price"]
            print("当前价差:", spread)
            if spread > 200:
                print("目标持仓: 空近月，多远月")
                # 设置目标持仓为正数表示多头，负数表示空头，0表示空仓
                target_pos_near.set_target_volume(-1)
                target_pos_deferred.set_target_volume(1)
            elif spread < 150:
                print("目标持仓: 空仓")
                target_pos_near.set_target_volume(0)
                target_pos_deferred.set_target_volume(0)


这部分的完整示例程序请见 `t80.py <https://github.com/shinnytech/tqsdk-python/blob/master/tqsdk/demo/tutorial/t80.py>`_.


.. _tutorial_backtest:

策略回测
-------------------------------------------------
自己的交易程序写好以后, 我们总是希望在实盘运行前, 能先进行一下模拟测试. 要进行模拟测试, 只需要在创建TqApi实例时, 传入一个backtest参数::

    api = TqApi(TqSim(), backtest=TqBacktest(start_dt=date(2018, 5, 1), end_dt=date(2018, 10, 1)))

这样, 程序运行时就会按照 TqBacktest 指定的时间范围进行模拟交易测试, 并输出测试结果.

这部分的完整示例程序请见 `backtest.py <https://github.com/shinnytech/tqsdk-python/blob/master/tqsdk/demo/tutorial/backtest.py>`_.


更多内容
-------------------------------------------------
* 更多TqSdk的示例 :ref:`demo`
* 要完整了解TqSdk的使用, 请阅读 :ref:`core`
