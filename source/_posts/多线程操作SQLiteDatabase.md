---
title: 多线程操作SQLiteDatabase
date: 2019-11-09 09:43:53
tags:
    - 数据库
    - DataBase
    - SQLiteDataBase
---


# 问题产生初始缘由
> 前天有个客户反应在公司的智能售货柜上消费，没有产生订单。这不是等于白嫖？赶紧瞅瞅！    
抓回来日志看了一下，发现了问题。这台柜子网络信号非常非常非常的差，WebSocket发送的订单消息竟然没有成功抵达服务器，订单数据没有到服务器，当然产生不了数据。如此之大的漏洞，赶紧修复～


<!--more-->
与后端大佬商量了一下对策，给我个接口查询`订单key1`对应订单状态。*安卓端则在本地创建一个数据库，存储用户关门后产生的订单信息，并定时从数据库检索订单数据，与服务端确认对应订单是否已经产生，订单已产生则删除该订单本地记录，订单未产生则重新推送订单信息并保留本地数据，直至下一个周期重新确认该订单是否产生，确保订单信息已产生后，再删除本地记录。*

确定了方案就赶紧撸代码。*本地创建一张表，主键是自增长的id，`Integer`类型；还有一个字段是订单独一无二的key1，`text`类型；订单数据信息字段data，`text`类型；订单状态字段deleted，`Integer`类型。线程池中添加一个每隔二十分钟执行的定时任务，从数据库中查询，订单删除状态为`0`的数据，根据key1向后端查询该订单是否生成。已经生成则将订单删除状态从`0`改到`1`,表示该订单已经生成，下一次无需查询该订单；未生成订单的数据则再次通过`websocket`推送订单信息到服务器，确保所有的订单都有效都生成。*

代码修改好,打包Apk上传到服务器，首先推送到4台设备上升级。

过了一个小时，刷新了一下bugly，我靠，真的有bug。如下图：

![bugly](http://ww1.sinaimg.cn/large/006KjR9nly1g8zz3xty8gj30os0580tc.jpg)

错误提示非常明显：不能执行操作，因为数据链接池已经关闭。

我的代码中，每次操作数据库之后都会关闭链接，如下：
```java
    /**
     * 逻辑删除订单信息
     *
     * @param key1 key1
     * @return result
     */
    public boolean logicDelete(String key1) {
        SQLiteDatabase db = mHelper.getWritableDatabase();
        ContentValues contentValues = new ContentValues();
        contentValues.put("deleted", 1);
        int num = db.update(TABLE_NAME, contentValues, "key1=?", new String[]{key1});
        //关闭数据库
        db.close();
        return num != 0;
    }
```

上面代码，是单次操作一条数据，如果一个定时周期内产生了多条需要确认的订单，那么就可能会出现连续多次打开和关闭数据库的情况。经过我的实验发现，使用同一个`DataBaseHelper`对象打开一个数据库时，如果当前有未关闭的数据库链接对象，则会直接返回当前`SQLiteDatabase`对象。那么就会出现上述的问题：**连续执行数据库操作时，打开数据库时，返回的是同一个数据库对象，但是在操作数据库的时侯，出现前面的操作完成关闭了数据库，后面的操作继续操作数据库，那么就会抛出上面链接池已经被关闭的异常**。

# 解决

> 解决方法应该有很多，思想是，当所有的数据库操作都执行完毕之后，再关闭数据库。我这边写两种方法：

## 创建全局唯一的数据库链接

我没有使用这种方式，但思路应该是在`Activity`或者`Application`启动的时侯打开一个数据库链接，在`Activity`运行时或者程序运行时的数据库操作均使用同一个数据库链接，至于关闭链接则是在`Activity`或者`Application`退出时调用`close()`方法关闭链接。

我觉得程序运行过程中，操作数据库的地方有限，采用这种方式让数据库一直保持打开状态，会造成资源的浪费（虽然`StackOverflow`上有人推荐保持数据库打开）,我没有深入研究对程序有多大影响。我认为数据库用完就关，应该是正确的姿势，所以我推荐后面一种方式。

## 用线程安全的数据库管理类来管理数据库链接

> 思路是，使用一个线程安全的原子整形变量(`AtomicInteger`)来记录当前程序打开数据库的次数。打开数据库时，如果打开次数为零时打开一个新的数据链接，次数加1；如果打开次数不为零时，则返回已经打开的数据库链接对象，并把次数加1。关闭数据库时，如果打开次数为1时，则直接关闭数据库，并把次数减1；如果打开次数大于1，则不关闭数据库，但是次数减1。管理类使用单例设计模式，这样既保证来所有线程操作数据库时使用的是同一个数据链接，也保证了所有数据库操作结束再关闭数据库链接。

代码如下：
```java
package com.hgobox.hgobox.manage;
import android.content.Context;
import android.database.sqlite.SQLiteDatabase;
import java.util.concurrent.atomic.AtomicInteger;
/**
 * Author:matthew
 * <p>
 * Date:2019-11-08 19:52
 * <p>
 * Email:guocheng0816@163.com
 * <p>
 * Desc:数据库链接管理
 */
public class DBManager {
    //线程安全的原子变量记录数据库的打开此时
    private AtomicInteger mConnectionCount = new AtomicInteger();
    private static DBManager INSTANCE;
    private static DataBaseHelper HELPER;
    private SQLiteDatabase mDatabase;
    /**
     * 初始化
     *
     * @param context
     */
    private static synchronized void initInstance(Context context) {
        if (INSTANCE == null) {
            INSTANCE = new DBManager();
            HELPER = new DataBaseHelper(context);
        }
    }
    /**
     * 获取工具类实例
     *
     * @param context
     */
    public static synchronized DBManager getInstance(Context context) {
        if (INSTANCE == null) {
            initInstance(context);
        }
        return INSTANCE;
    }
    /**
     * 获取可读写的数据库
     *
     * @return database
     */
    public synchronized SQLiteDatabase getWritableDataBase() {
        //打开次数+1
        if (mConnectionCount.incrementAndGet() == 1) {
            mDatabase = HELPER.getWritableDatabase();
        }
        return mDatabase;
    }
    /**
     * 获取只读的数据库
     *
     * @return database
     */
    public synchronized SQLiteDatabase getReadableDataBase() {
        //打开次数+1
        if (mConnectionCount.incrementAndGet() == 1) {
            mDatabase = HELPER.getReadableDatabase();
        }
        return mDatabase;
    }
    /**
     * 链接数为零的时候关闭数据库
     */
    public synchronized void closeDatabase() {
        //打开次数-1
        if (mConnectionCount.decrementAndGet() == 0) {
            mDatabase.close();
        }
    }
}
```

**代码中方法都使用`synchronized`关键字修饰，保证多线程操作时的线程安全。**

代码已经运行了好多天了，Bugly没有上报异常～

> 以上记录一次踩坑～
