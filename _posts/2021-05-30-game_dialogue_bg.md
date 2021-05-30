---
title: 【GMS】简单的文本渲染引擎——对话框背景和换行滚动
date: 2021-05-30 23:30:00 +0800
categories: [game]
tags: [game]
seo:
  date_modified:2021-05-30 23:30:00 +0800
---

------------

### 文字部分TODOLIST ###

- ~~输出中文字符~~
- ~~文字加载动画~~
- ~~文字颜色设置~~
- ~~文字翻页~~
- ~~对话框适配（宽高与定位适配）~~
- ~~对话框sprit绘制~~
- 对话框插入头像与小图标图片

------------

我自己都没有发现其实昨天已经将翻页的功能实现了……
以及今天由于一些低级操作导致大部分代码丢失，花了一晚上终于把代码重新写了回来，顺便再统一了一下代码的书写格式。
今天的目标是对话框适配以及适配后加上对话框背景。

对话框的绘制需要用到Nine slices功能。油管上（[UI Boxes + Windows](https://www.youtube.com/watch?v=Cgb6yw1WdRw)）正好看到一个作者有对这个功能进行讲解，思路和我一开始设想的差不多，但我没想到实际实现起来需要考虑的因素还有很多……

如果对话框为一个标准rectangle，则可以抽象为9个部分：四个角落、上下左右、中间填充。
![2021053001](/assets/img/post/2021053001.png)
在对话框进行`resize`的过程中，四个角落不变化；上下左右进行横向纵向拉伸；中间则进行填充操作。

2.3版本以后GMS虽然已经实现了nine slices，但在激活该功能的情况下将无法调用一系列部分绘制的函数，如果只是拖动OBJ到ROOM的方式还可行；
但对于需要动态绘制对话框宽高的需求则相对不灵活，resize只提供scale比例方式缩放，不实用与像素计算。
官方文档和示例：[https://www.yoyogames.com/en/blog/slick-interfaces-with-9-slice](https://www.yoyogames.com/en/blog/slick-interfaces-with-9-slice)

假设我们需要输出一段对话，则对话框实际显示的时候，文字会出现在中间部分，上下左右和四个角落都属于无内容的区域。
![2021053002](/assets/img/post/2021053002.png)

因此在绘制之前，我们需要先知道如下信息。

1. 绘制的起始位置坐标xy（可以根据玩家的坐标或者互动对象坐标，基于对话框宽高修正得出）
2. 绘制的对话框宽高（需要知道当前绘制的文本长度与行数）
3. 文字填充区域的宽高（需要知道当前绘制的文本长度与行数）

可以使用`string_width`直接获取某个字符串总长度，使用这个函数要注意，不同的字体、字号会影响其结果，并且中文与英文的宽度也会受到字体影响。

但由于需要保留“文本逐字显示”的动效，同时若文本框有宽度限制和高度限制是需要实现自动换行，因此`string_width`的使用需要额外处理，并且要另外实现**横向自动换行**和**纵向滚动**能力。（看样子前面的代码就避免不了要升级一次了）

### 对话框盒模型抽象

![2021053007](/assets/img/post/2021053007.png)

假设，盒子的坐标需要基于玩家定位（当然可以切换成任何NPC的XY），则可以预留一段`fix_y`空间让内容不会太紧凑。
对话框则和CSS3盒模型类似，确定一个盒子的大小与位置，需要知道盒子宽、高；
宽与高需要根据对话的内容和换行来确定，因此需要先实现“手动换行”和“自动换行”

### 手动换行

文本编辑中加入**手动换行符**逻辑，并且追加解析逻辑。

```
function dialogue_prepare_cmd(_str) {
    var _commands = [];
    for (var _i = 0; _i < string_length(_str); _i++) {
        var _targetChar = string_char_at(_str, _i + 1); // string_char_at的索引计算从1开始
    
        // 颜色探测 格式为
        if _targetChar == "{" && string_char_at(_str, _i + 2) == "{" {
            var _color = string_to_color(string_copy(_str, _i + 3, 6));
            array_push(_commands, { cmd: CMD_DRAW_SET_COLOUR, value: _color, });
            _i += 9;
            continue;
        }
        
        // 换行符检查
        if _targetChar == "\n" {
            array_push(_commands, { cmd: CMD_DRAW_NEW_LINE, value: "", });
            continue;
        }

        // 普通字符
        array_push(_commands, { cmd: CMD_DRAW_TEXT, value: _targetChar, });
    }
    
    return _commands;
}
```

![2021053008](/assets/img/post/2021053008.png)

### 自动换行

纵向行数则会根据最大宽度，对进行cmd重组，并在适当的位置主动插入换行符。
注意，`string_width`的计算基于字体（类型与大小），因此在使用这个函数计算像素之前务必确保`draw_set_font`的设置正确。

P.S.：GML不支持多值返回，写起来代码避免不了有点难看……

```
// 对commands进行重组拆行 在特定的位置塞入滚动指令
// 返回多个结果 1.被resize后的cmds 2.被resize的次数 3.resize后N行中最长的一行文本宽度
function dialogue_resize_cmd(_cmds, _max_width) {
    var _ret = [];
    var _cur_width = 0;
    var _resize_times = 0;
    var _his_max_width = 0;

    for (var _i := 0; _i < array_length(_cmds); _i++) {
        // 手动分页
        if _cmds[_i].cmd == CMD_DRAW_NEW_LINE {
            array_push(_ret, { cmd: CMD_DRAW_NEW_LINE, value: "", });
            if _cur_width > _his_max_width {
                _his_max_width = _cur_width;
            }
            _cur_width = 0;
            _resize_times++;
            continue;
        }

        // 非文字指令
        if _cmds[_i].cmd != CMD_DRAW_TEXT {
            array_push(_ret, _cmds[_i]);
            continue;
        }

        // 获取该字符的长度 不同的字体unicode和ascii的长度不一定相同
        var _ud = string_width(_cmds[_i].value);
        // 长度超过最大限制 在追加字符之前追加换行
        if _cur_width + _ud > _max_width {
            _resize_times++;
            array_push(_ret, { cmd: CMD_DRAW_NEW_LINE, value: "", });
            _cur_width = 0;
            _his_max_width = _max_width;
        }
        
        _cur_width += _ud;
        array_push(_ret, _cmds[_i])
    }

    if _cur_width > _his_max_width {
        _his_max_width = _cur_width;
    }
    return [_ret, _resize_times, _his_max_width,];
}
```

### 绘制变量预处理

有了对话文本宽高后，我们再在外层包上padding，根据触发事件的Object的xy值，就能计算出上图中所有的信息了。
![2021053009](/assets/img/post/2021053009.png)

### 开始绘制

绘制部分分两个阶段——背景图绘制与文字绘制。
由于已经有了所有绘制需要的坐标和数据，因此绘制就相对简单很多了。
本次因为实现的是简单的对话框，因此绘制就暂时不考虑`draw_sprite_part`直接用矩形堆叠一些背景图来替代好了。

```
// 用简单的方式先绘制一个临时对话框
function draw_dialogue_simple_bg(_fix_x, _fix_y) {
    var _c1 = $20304d;
    var _c2 = $577ab9;
    var _c3 = $b0e4ef;

    // 修正两个object之间流出一些距离
    var _ox = x + _fix_x;
    var _oy = y + _fix_y;

    var _x11 = _ox - textbox_width / 2 - textbox_padding_x;
    var _y11 = _oy - textbox_height - textbox_padding_y * 2;
    var _x12 = _x11 + textbox_width + 2 * textbox_padding_x;
    var _y12 = _oy; 
    draw_rectangle_color(_x11, _y11, _x12, _y12, _c1, _c1, _c1, _c1, false);
    
    var _x21 = _x11 + 2;
    var _y21 = _y11 + 2;
    var _x22 = _x12 - 2;
    var _y22 = _y12 - 2; 
    draw_rectangle_color(_x21, _y21, _x22, _y22, _c2, _c2, _c2, _c2, false);
    
    var _x31 = _x21 + 2;
    var _y31 = _y21 + 2;
    var _x32 = _x22 - 2;
    var _y32 = _y22 - 2; 
    draw_rectangle_color(_x31, _y31, _x32, _y32, _c1, _c1, _c1, _c1, false);
    
    var _x41 = _x31 + 2;
    var _y41 = _y31 + 2;
    var _x42 = _x32 - 2;
    var _y42 = _y32 - 2; 
    draw_rectangle_color(_x41, _y41, _x42, _y42, _c3, _c3, _c3, _c3, false);
}
```

P.S.：有一些坑，没踩过是很难发现的，比如GMS使用的十六进制色值并非RGB，然而`maker_color_rgb`的入参顺序是RGB。
这种前后风格不一致实在是特别容易困扰……
![2021053003](/assets/img/post/2021053003.png)

对话框绘制完毕以后就是文本部分了。
文本部分相对复杂，换行部分的坐标逻辑计算有点混乱，建议用一张笔和纸做草稿一边画图一边写代码……
（这种时候有编程经验的优势就体现出来了。）

```
// return true: 绘制结束 false:绘制未结束(换行重置)
function draw_textbox_simple_text() {
    var _char_display_length = 0; // 当前已经绘制过的字符的长度
    var _char_display_lines = 1; // 当前已经绘制过的字符的行数 默认1开始

    for (var _i = 0; _i < pg_cmd_index; _i++) {
        var _curC = resize_content[_i];
        switch (_curC.cmd) {
            case CMD_DRAW_SET_COLOUR:
                draw_set_color(_curC.value);
                break;
            case CMD_DRAW_NEW_LINE: // 滚动的实现分超过与不超过最大行数逻辑
                // 超过最大行数时 清屏(裁剪掉多余行 重新定位cmd_index后 重新开始绘制)
                if _char_display_lines >= textbox_line_maximun {
                    dialogue_cut_cmd_line(resize_content, textbox_line_maximun);
                    pg_cmd_index = 0;
                    return false;
                }
                // 没超过最大行数时 Y的坐标系向下平移 以及重置x
                _char_display_length = 0;
                _char_display_lines++;
                break;
            case CMD_DRAW_TEXT:
                var _x = x - textbox_width / 2 + _char_display_length;
                var _y =  y + textbox_player_space_y - textbox_padding_y - textbox_height + (textbox_line_height + textbox_line_space) * (_char_display_lines - 1);
                draw_text(_x, _y, _curC.value);
                _char_display_length += string_width(_curC.value);
                break;
        }
    }
    return true;
}
```

实现了绘制文本和绘制背景图后，则在对象中直接调用即可。
![2021053010](/assets/img/post/2021053010.png)

最后来看看效果。
![2021053011](/assets/img/post/2021053011.png)

大功告成！

### 结尾

前几天写的代码现在回过头来看，简直看不下去，于是花了一些时间来进行了整理，例如参数传递，局部变量和全局变量的使用规范等等。

文本适配应该是绘制中最难的部分了，当然如果要实现其他乱七八糟的效果（彩虹字体、抖动文字）则计算难度还要再提升一个级别，不过现阶段用不到那么多乱七八糟的效果，因此这算是把这块骨头啃下来了。

后天要去重庆，晚上得收拾行李，因此明天应该就是最后一天了。
正好明天也是文本引擎最后的一个部分：在对话框中加入头像，争取一次搞定吧！
