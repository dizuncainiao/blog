---
title: Greedy Snake
date: 2021-12-23 15:47:51
---

## 效果图

最近使用 JS 写了一个贪吃蛇游戏，效果如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210416221615434.gif#pic_center)


[谷歌浏览器点击直接在线玩](https://dizuncainiao.gitee.io/dz-greedysnake/)

## 前言

贪吃蛇作为一款经典又简单的小游戏，每个人都玩过。实现一个贪吃蛇游戏基本具有以下功能：

- 棋盘（也被称作 “地图”，我这里画的像一个围棋棋盘，索性就叫棋盘）
- 蛇 （细致一点分为：蛇头、蛇身、蛇尾）
- 方向（上下左右）控制，并且自动行走
- 碰撞检测（撞墙、撞自己）
- 食物在随机位置生成
- 蛇吃到食物，尾部生长一截

以上也便是我的实现步骤了，下面分享一些更详细的实现思路。

## 棋盘

棋盘就是蛇运动的区域，简陋一点甚至可以不要边框，但是为了更好的游戏体验，这里还是加上边框。但是此边框非彼边框，使用 `border` 来实现效果图中的边框也可以，但这里我采用了其他的方式，比如 **背景色** ，看下图：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210416221635387.gif#pic_center)


通过层级、定位，使得黑块成为了蓝块视觉上的边框，然后结合 `grid` 网格布局实现 **格格分明** 的棋盘便尤为简单。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210416221657600.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RpenVuY2Fpbmlhbw==,size_16,color_FFFFFF,t_70#pic_center)


这里一个大块采用 **黑色背景** ，400个小块采用 **白色背景** ，使用 `grid` 布局横、纵分别 20 个依次排列，再加上网格的行、列间距，便能形成围棋棋盘式的布局。这里的黑线实则就是 **遮挡+间隙** 形成的，代码如下：

```html
<style>
// 大块
#checkerboard {
    display: grid;
    grid-template-columns: repeat(20, 16px);
    grid-template-rows: repeat(20, 16px);
    grid-column-gap: 1px;
    grid-row-gap: 1px;
    padding: 1px;
    background: #000;
}

// 小块
.square {
    background: #fff;
}
</style>
```

因为是 `grid` 布局，所有一些 **上古浏览器** 就不能玩这个游戏了。

![在这里插入图片描述](https://img-blog.csdnimg.cn/2021041622172663.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RpenVuY2Fpbmlhbw==,size_16,color_FFFFFF,t_70#pic_center)


## 坐标

很多游戏都有坐标的概念，贪吃蛇支持自动行走、食物生成在随机位置以及碰撞检测等功能都是依据坐标来实现的。

我这里的坐标实现很简单：第一行第一个格子坐标为 **1-1**，第一行最后一个格子坐标为 **1-20**

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210416221959668.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RpenVuY2Fpbmlhbw==,size_16,color_FFFFFF,t_70#pic_center)


在生成 `div` 时将每一个格子 **对应的坐标** 通过 **css类名** 对应起来，代码如下：

```javascript
/**
 * 棋盘 20 * 20
 * 坐标：1-1, ... 1-20, ... 20-20
 */
class Checkerboard {
    constructor(container, gridNum) {
        this.container = container
        this.gridNum = gridNum
        this.generate()
    }

    // 每个格子类名为：SX轴坐标_Y轴坐标 S1_20
    generate() {
        const fragment = document.createDocumentFragment()
        for (let i = 0; i < this.gridNum; i++) {
            for (let j = 0; j < this.gridNum; j++) {
                const grid = document.createElement('div')
                grid.classList.add('square', `S${j + 1}_${i + 1}`)
                fragment.appendChild(grid)
            }
        }
        this.container.appendChild(fragment)
    }
}
```
这样第一行第一个 `div` 的类名是 `S1_1`，第一行最后一个 `div` 的类名为 `S1_20`。可以这么说，这个 <font color=red>**css 类名的设计就是这个游戏的核心设计**</font> ，因为后续的自动行走、食物生成在随机位置以及碰撞检测等功能全部都是在操作 **css类名** 。

更详细代码可以在底部查看源码~


## 蛇、食物的绘制

很多时候采用 **CSS 样式** 结合 **css类名** 的切换能更好、更快、更高性能的实现动画或者其他的逻辑。那么在这里 **蛇** 和 **食物** 绘制我直接通过加 **css类名** 就解决了，代码如下：

```html
<style>
.square {
    display: flex;
    align-items: center;
    justify-content: center;
    background: #fff;
}

.snake::after {
    display: block;
    content: '';
    width: 70%;
    height: 70%;
    border-radius: 100%;
    background: #9bbef8;
}

.snake.head::after {
    background: #0037ff;
}

.snake.tail::after {
    background: #7400d2;
}

.food::after {
    display: block;
    content: '';
    width: 70%;
    height: 70%;
    border-radius: 100%;
    background: #d54830;
}
</style>
```

将相应的 **css 类名** 加在 `.square` 上就生成了蛇和食物了，粗暴一点伪元素可以不用，这里为了美观一点使用 **伪元素+圆角** 修饰一下。

![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-Y7fVKbEn-1618582608058)(E:\gitStore\blog-md\纯JS实现贪吃蛇——超上瘾小游戏\image-20210416200314757.png)\]](https://img-blog.csdnimg.cn/20210416221757763.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RpenVuY2Fpbmlhbw==,size_16,color_FFFFFF,t_70#pic_center)


## 蛇的行走 & 碰撞检测 & 吃到食物

以上三个功能都是依据坐标来实现的，索性放在一起讲

#### 蛇的行走

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210416221826799.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RpenVuY2Fpbmlhbw==,size_16,color_FFFFFF,t_70#pic_center)


就如上图所示，实现蛇的行走其实非常简单，这里只需要想懂一点即可：**每次移动，除了蛇头坐标(蓝色圆点)是一个全新的坐标，其他圆点的坐标都是它之前的那一个坐标** 。这里蛇的原始坐标是 `['12-13', '11-13', '10-13']` ，那么它往右走的话只要将 `13-13` 作为蛇头坐标即可，上一段模拟代码：

```javascript
const arr = ['12-13', '11-13', '10-13']

// 删除蛇尾坐标
arr.pop()

// 加入新的蛇头坐标
arr.unshift('13-13')

// arr => ['13-13', '12-13', '11-13']
```

蛇尾坐标肯定要删的，不然蛇不吃食物都能生长了，可以脑补一下画面~

在一开始生成棋盘的时候，每个 **格子(div)** 都已经绑定上与坐标相对应的 **css类名** ，这里根据上面的坐标数组将相应的 div  **添加/移除css类名** 即可。

#### 碰撞检测

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210416221842838.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RpenVuY2Fpbmlhbw==,size_16,color_FFFFFF,t_70#pic_center)


**碰撞检测** 听起来好像 ”一头雾水😲😲什么东东😱😱“，但是一个贪吃蛇游戏的碰撞检测还能有多难。原理其实就是这样：蛇只能在棋盘内活动，蛇存在的 **有效坐标** 也只能包含在棋盘内。这里横纵坐标最小、最大值分别是 **1 、20**， 只需要判断 **蛇头坐标** 是否在 **1-20** 以内(包含1和20)就可以了。

#### 吃到食物

吃到食物其实也是碰撞检测的一种，判断 **新的蛇头坐标** 和 **食物坐标** 一样即为吃到食物，并且会在当前蛇尾周围可用坐标中随机生成一个新的蛇尾坐标，如下图：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210416221900703.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RpenVuY2Fpbmlhbw==,size_16,color_FFFFFF,t_70#pic_center)


当蛇头和食物重合后，淡蓝的圆点便成了新的蛇尾坐标，并且会在红色箭头所示的坐标中随机选一个当做新的蛇尾坐标。需要注意的是 **新的蛇尾坐标** 可能会不存在，那么此时就 game over 了。

## 总结

更多代码也就不贴了，以上也基本将这个小游戏的核心都讲遍了，关于 **模式** 实则是坐标的另一种简单运用罢了，**难度** 更仅仅是定时器的时间改变而已。还有很多想法后续有时间再实现吧，如下：

- 等级称号（达成相应的分数来实现相应的称号，比如：初出茅庐……）
- 加速功能（按住空格键来根据定时器的时间实现加速）
- 存档功能（按住个什么键保存当前蛇的坐标、模式等信息，存入本地缓存，下次再玩）
- 周期性出现一个 **大食物** 一次长好几节
- 等等等等~

## 源码

[https://gitee.com/dizuncainiao/dz-greedysnake](https://gitee.com/dizuncainiao/dz-greedysnake)

