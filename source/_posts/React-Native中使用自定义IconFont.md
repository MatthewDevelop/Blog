---
title: React-Native中使用自定义IconFont
date: 2018-12-15 09:20:34
tags:
    - React Native
    - IconFont
    - 自定义
---

> 初学React Native，在App中有一个TabBar需要用到图标。平时在开发Android时TabBar图标基本不用图片，都是使用字体图标替代图片，字体图标有自适应屏幕的特性，不会再不同分辨率上失真，从而免去了适配不同分辨率手机的困扰。所以在想React Native中是否可以使用字体图标，研究了一下，果然可行，记录如下。

<!-- more -->

用到的工具类库和资源：

- [react-native-vector-icons](https://github.com/oblador/react-native-vector-icons)
- [阿里巴巴字体图标库](https://www.iconfont.cn/)

## 最终效果

先展示一下使用效果：
```JavaScript
//导入组件
import Icon from './js/common/IconFont';
//使用组件
<Icon name='icon_hot' size={20} color='lightgreen' />
<Icon name='icon_trending' size={20} color='lightgreen' />
<Icon name='icon_collect' size={20} color='lightgreen' />
<Icon name='icon_user' size={20} color='lightgreen' />
```
效果如图：
![效果图](/images/icon_result.png)

## 安装 
1. 首先安装第三方的字体库组件,这里推荐使用 [react-native-vector-icons](https://github.com/oblador/react-native-vector-icons) ,安装步骤如下：
    - 安装依赖：
    `npm install react-native-vector-icons --save`
    - (可选)如果需要使用默认的字体文件，则需要链接资源库        
        - 自动：`react-native link`
        - 手动(不推荐)：[GitHub官方仓库Installation部分](https://github.com/oblador/react-native-vector-icons)
2. 这里需要重新编译运行项目。
3. 使用测试是否安装成功：

```JavaScript
//导入
import Icon from 'react-native-vector-icons/FontAwesome';
//使用组件
<Icon name="rocket" size={30} color="#900" />
```

## 使用自定义的字体文件

1. 从[阿里巴巴字体图标库](https://www.iconfont.cn/)下载需要的字体图标，添加到自己创建的项目中。
2. 点击下载，打包下载到本地：
![下载至本地](/images/icon_download.png)
3. 解压文件，`iconfont.ttf`文件就是后面需要用到的文件：
![下载至本地](/images/icon_compress.png)
4. 将文件放到指定的位置：
    - Android:
    在Android项目的assets目录下创建文件夹，名为`fonts`，将上面解压出来的ttf文件拷贝到此目录下。
    - IOS：
    为了便于字体的统一管理，在项目目录下创建fonts文件夹，并将下载的字体文件拖到文件夹中，Xcode会弹出下图提示框，`Add to target` 选择当前项目：
    ![添加文件](/images/icon_add_file.png) 确保添加文件后，项目的Build Phases的Copy Bundle Resources中有刚刚添加的字体文件，如下图：
    ![BuildPhases](/images/icon_refrence.png)接着再修改info.plist文件，在Information Property List下的Fonts provided by application下添加一个item,value就是字体文件的名字，如下图：
    ![BuildPhases](/images/icon_plist.jpg)至此，ios配置完成.
5. 使用自定义的字体文件，如何使用先看看源码：
    ```JavaScript
    /**
    * FontAwesome icon set component.
    * Usage: <FontAwesome name="icon-name" size={20} color="#4F8EF7" />
    */
    import createIconSet from './lib/create-icon-set';
    import glyphMap from './glyphmaps/FontAwesome.json';

    const iconSet = createIconSet(glyphMap, 'FontAwesome', 'FontAwesome.ttf');

    export default iconSet;
    ```
    FontAwesome.json文件内容如下：
    ```Json
    {
        "glass": 61440,
        "music": 61441,
        "search": 61442,
        "envelope-o": 61443,
        "heart": 61444,
    }
    ```
    FontAwesome字体返回的是一个iconSet,简单理解为是一个显示字体图标的组件，创建这个组件需要三个参数，分别为：
    >- glyphMap: 是一个json对象，存储的是图标的信息，其中key是字体图标的名字，value是图标的十进制编码。
    >- fontFamily: 字体库对应的fontFamily，可以在下载字体的网站查得到，以阿里巴巴图标库为例：创建的项目下有编辑项目，点开就可以看见图标库的fontFamily。
    >- fontFile: 字体文件的名称
6. 根据FontAwesome的创建方式，我们可以自己创建一个自定义的字体图标，目前已经可以拿到fontFamily和字体文件，还缺少包含字体名称和值的json对象。以阿里巴巴字体图标库为例，我们可以在项目图标下找到图标的十六进制的值，如下图：
![User](/images/icon_user.png)e663即为图标的16进制的值，我们需要将其转换为10进制的值，我用到字体图标不多，所以用[在线转换工具](http://tool.oschina.net/hexconvert/)就拿到了需要的10进制的值，如果图标比较多的话，可以写一段代码从下载下来的`iconfont.css`文件中截取再转换生成对应的json文件，还可以重复使用，有时间可以试试！
7. 创建自定义的图标组件，代码如下：
    ```JavaScript
    //IconFont.js
    import createIconSet from 'react-native-vector-icons/lib/create-icon-set';

    const glyphMap = {
        'icon_user': 58979,
        'icon_collect': 58928,
        'icon_hot':59222,
        'icon_trending':60788,
    }

    //映射表，fontFamily，字体文件
    const IconFont = createIconSet(glyphMap, 'iconfont', 'iconfont.ttf');

    export default IconFont;
    ```
    由于图标较少，我直接将字体键值放到了一个json对象，没有单独创建文件。
8. 使用方式
    ```JavaScript
    //导入组件
    import Icon from './js/common/IconFont';
    //使用组件
    <Icon name='icon_hot' size={20} color='lightgreen' />
    ```
    效果文章的开始已经展示过了，至此自定义字体库成功使用！
    

    

