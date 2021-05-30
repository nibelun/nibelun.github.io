---
title: 【GMS】简单的文本渲染引擎(1)
date: 2021-05-29 06:00:00 +0800
categories: [game]
tags: [game]
seo:
  date_modified: 2021-05-29 06:00:00 +0800
---

------------

### 文字部分TODOLIST ###

- ~~输出中文字符~~
- ~~文字加载动画~~
- ~~文字颜色设置~~
- 文字翻页
- 对话框适配（宽高与定位适配）
- 对话框sprit绘制
- 对话框插入头像与小图标图片

------------

文本引擎需要处理的问题有两个：文字渲染动画与文字颜色控制。
文字渲染动画的实现思路为“按字符逐帧绘制”，相对比较简单；文字颜色控制则相对困难一些。

GML里控制文本输出的两个关键方法：`draw_text_colour()`和`draw_text`。
[draw_text](https://manual.yoyogames.com/#t=GameMaker_Language%2FGML_Reference%2FDrawing%2FText%2Fdraw_text.htm&rhsearch=draw_text)
[draw_text_colour](https://manual.yoyogames.com/#t=GameMaker_Language%2FGML_Reference%2FDrawing%2FText%2Fdraw_text_colour.htm&rhsearch=draw_text)
没有Linux内bash的颜色控制那种现成的功能，但可以参考类似实现的思路来编写一个简单的文字引擎。
顺带一提，`draw_text_ext`与`draw_text_ext_color`虽然很方便但不利于定制化。后续在实现宽高定位适配的时候自己实现类似的间距以取代这两个ext函数。

draw_text_colour实现了四点色和alpha，不过这个功能暂时用不到，简单考量只需要实现颜色控制即可。

假设如果有一串文本，我们要给其中几个文字使用特定的颜色，可以讲颜色进行规定——将颜色渲染需要使用的信息夹放在两个`##`符号中间

```
ABCD\{\{FF0000\}\}EFG\{\{FFFFFF\}\}
```

例如`\{\{FF0000\}\}`，内部的数字中前6为代表十六进制RGB（可以使用`real`转换），则此表达式代表设置颜色为红色全不透明；`\{\{FFFFFF\}\}`还原回白色去。因此如上一段文本中EFG加工成红色，其余文字为白色。

为了防止在解析到`\{\{FF0000\}\}`设置颜色后跳到其他文本（强制翻页或者从头输出），需要在文本开头手动设置一次初始颜色。于是可以得到类似下列结构规范书写文本。

```
\{\{FFFFFF\}\}ABCD\{\{FF0000\}\}EFG\{\{FFFFFF\}\}
```


通过实现字符串解析的方式实现解析引擎。

```
  将对话的内容进行使用结构的预处理
  根据特定的格式 把文本处理成{cmd: xxx, value: xxx}指令集结构
  指令集包含有“绘制文本”和“绘制颜色”
*/ 
function dialog_prepare_text(str) {
    var commands = [];
    for (var i = 0; i < string_length(str); i++) {
        var targetChar = string_char_at(str, i + 1); // string_char_at的索引计算从1开始
    
        // 颜色探测 格式为{{FFFFFF}}
        if targetChar == "{" && string_char_at(str, i + 2) == "{" {
            var color = string_to_color(string_copy(str, i + 3, 6));
            array_push(commands, {
                cmd: CMD_DRAW_SET_COLOUR,
                value: color,
            });
            i += 9;
            continue;
        }
        
        // 普通字符
        array_push(commands, {
            cmd: CMD_DRAW_TEXT,
            value: targetChar,
        });
    }
    
    return commands;
}

// pobj 玩家对象 tobj 互动的对象
// 每一页的对话都是一个单独的dialog_simple 对话结束之后需要进行销毁 因此对话的页码状态记录到全局里
function dialog(pobj, tobj){
    global.userInDialogPg++;
    var contentPg = array_length(tobj.content);

    if global.userInDialog {
        instance_destroy(obj_dialog_simple);

        // 末页 对话结束
        if global.userInDialogPg >= contentPg {         
            global.userInDialog = false;
            global.userInDialogPg = -1;
            return;
        }
    }

    global.userInDialog = true;
    var ins = instance_create_layer(pobj.x, pobj.y, "Instances", obj_dialog_simple);
    ins.content = tobj.content;
}
```

上述代码`dialog_prepare_text`为一个简便的文本解析器，负责把原始文本转化为方便操作的指令对象。
`dialog`则负责创建和销毁对话框对象；若对话存在多页，则同时控制翻页。

实际绘制的时候则由对话框对象具体控制。
Draw事件中可以通过CMD来控制输出文字或设置颜色，因此就能达到颜色穿插的效果。
![2021052902](/assets/img/post/2021052902.png)
![2021052903](/assets/img/post/2021052903.png)

而后续实际的文本书写则可以暂时先采用这种方式。
![2021052901](/assets/img/post/2021052901.png)

可以通过alarm事件中反复添加alarm来达到类似interval的效果，于是乎简单的带文字和动画效果的对话文本绘制就完成了。
![2021052904](/assets/img/post/2021052904.png)
![2021052905](/assets/img/post/2021052905.png)

再通过在字符数新增（即pgCMDIndex变化）的时候附加一个声音音效，于是乎对话系统似乎又有更多内容了。
![2021052906](/assets/img/post/2021052906.png)

看上去简单的对话框系统实际上要完成这些功能没想到花费了我几乎一整天的时间，若不是以前自己尝试性写过一些游戏引擎，估计还得花更多时间去理解和调试，尤其是自己甚至把音频播放控制放在Draw函数里这种愚蠢的行为（（（。

这期间不断的摸索和调试过程中，自己也逐渐熟悉了很多基础的函数，同时在YOYO社区中也发现了不乏[FC's Dialogue System](https://marketplace.yoyogames.com/assets/6076/fc-s-dialogue-system)这类优秀的对话框系统，不得不感慨作者脑洞如此之大。

今天的目标达成，明天的目标是对话中断强制翻页以及宽高计算！