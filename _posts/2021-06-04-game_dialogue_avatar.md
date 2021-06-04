---
title: 【GMS】简易文本引擎——进阶对话框
date: 2021-06-04 17:00:00 +0800
categories: [game]
tags: [game]
seo:
  date_modified: 2021-06-04 17:00:00 +0800
---

------------

### 文字部分TODOLIST ###

- ~~输出中文字符~~
- ~~文字加载动画~~
- ~~文字颜色设置~~
- ~~文字翻页~~
- ~~对话框适配（宽高与定位适配）~~
- ~~对话框sprit绘制~~
- ~~对话框插入头像与小图标图片~~

------------

### 前言

最近几天一直在忙房子的事情，也就晚上大家都下班以后回到酒店的睡前两个小时有空研究研究。
借着这些难以集中精力写码的时间，看了不少别人做出来的成品，也许在完成对话框后我需要一段时间来好好重新把整个游戏的内容、玩法以及交互全部重新给细化出来。

### 设计

今天的目标是实现一个带头像和人名的对话框。

对话框的UI设计其实现在可以参考大部分游戏的排版。绝大部分无外乎三个元素：人物头像、名字、以及对话内容。虽然还有很多其他的样式，但对话内容只保留这三个部分已经足够应对未来要使用的场合了。
![2021060401](/assets/img/post/2021060401.png)
![2021060402](/assets/img/post/2021060402.jpeg)

三个部分都为独立的部分，需要独立绘制。与之前的那个简易对话框不同的是，本次要实现的对话框，其坐标使用的xy相对于摄像机的”绝对“位置，而非相对于对象的位置。

### 绘制

绘制共分为2个阶段——文本背景和文字。

由于无法复用之前给互动对象定义的content的array结构，此处我们重新将content定义成object，方便后续用于拓展其他对话控制的变量传递。

```
event_inherited();

content = {
    avatar_left_spr: spr_role_1,
    avatar_left_name: "泽克",
    cmds: [
        dialogue_prepare_cmd("{{FFFFFF}}这套衣服买了已经有一段时间了，\n是小弟当时送我的{{FF0000}}生日礼物{{FFFFFF}}。\n不过看着似乎还挺合身。\n但还是先穿着看看吧。"),
        dialogue_prepare_cmd("{{FFFFFF}}希望小弟会喜欢……"),
    ],
};

interaction = function(player) {
    dialogue_call(player, self, obj_dialogue_avatar);
}
```

对话框部分可以复用之前的代码，而基于原来的代码添加一些新的绘制属性。

```
avatarbox_x = 0;
avatarbox_y = 0

namebox_x = 150;
namebox_y = 290;
namebox_padding_x = 0;
namebox_padding_y = 0;
```

若想实现绘制的时候向下对其，则在绘制的初始化阶段时，需要重新计算其y的位置。
```
// 初始化 不同的立绘可能高度不一样 因此此处动态计算 后续统一则可以放到外层进行
avatarbox_x = 20;
avatarbox_y = camera_get_view_height(camera_get_default()) - sprite_get_height(content.avatar_left_spr);
```

绘制的部分使用`Draw GUI`事件（需要始终绘制在屏幕的中下方），基于之前的实现来进行一些修改。
```
// 未开始
if pg_cmd_index == -1 {
    return;
}

// 画背景
draw_dialogue_avatar_spr();
draw_dialogue_avatar_name();
draw_dialogue_avatar_bg();

// 画文字
while(true) {
    if draw_textbox_avatar_text() {
        break;
    }
}

// 告知停止 放在这里（上I次绘制结束后下一次flush之前）
if pg_stop_draw {
    exit; // 节约性能
}
```

人物立绘的绘制相对简单，需要注意的就是图片的尺寸不要过大即可。

文本背景部分则需要根据之前提到的Nine Slice方式进行。不过由于官方的功能灵活程度不如预期，此处自行手写实现也无妨。
```
function draw_dialogue_avatar_spr() {
    draw_sprite(content.avatar_left_spr, 0, avatarbox_x, avatarbox_y);
}

function draw_dialogue_avatar_name() {
    var _namel = string_width(content.avatar_left_name);
    var _nameh = string_length(content.avatar_left_name);
    var _part  = sprite_get_width(spr_dialogue_box_1_ns);

    // 左上角
    var _x1 = namebox_x - _part - namebox_padding_x;
    var _y1 = namebox_y - _part - namebox_padding_y;
    draw_sprite(spr_dialogue_box_1_ns, 0, _x1, _y1);
    // 右上角
    var _x2 = namebox_x + _namel + namebox_padding_x;
    var _y2 = _y1;
    draw_sprite(spr_dialogue_box_1_ns, 2, _x2, _y2);
    // 左下角
    var _x3 = _x1;
    var _y3 = namebox_y + _nameh + _part + namebox_padding_y;
    draw_sprite(spr_dialogue_box_1_ns, 6, _x3, _y3);
    // 右下角
    var _x4 = _x2;
    var _y4 = _y3;
    draw_sprite(spr_dialogue_box_1_ns, 8, _x4, _y4);
    // 上下左右
    draw_sprite_stretched(spr_dialogue_box_1_ns, 1, _x1 + _part, _y1, _namel + 2 * namebox_padding_x, _part);
    draw_sprite_stretched(spr_dialogue_box_1_ns, 7, _x3 + _part, _y3, _namel + 2 * namebox_padding_x, _part);
    draw_sprite_stretched(spr_dialogue_box_1_ns, 3, _x1, _y1 + _part, _part, _y3 - _y1 - _part);
    draw_sprite_stretched(spr_dialogue_box_1_ns, 5, _x2, _y2 + _part, _part, _y3 - _y1 - _part);
    // 中间
    draw_sprite_stretched(spr_dialogue_box_1_ns, 4, _x1 + _part, _y1 + _part, _x2 - _x1 - _part, _y3 - _y1 - _part);
    // 文字
    draw_set_color(c_white);
    draw_text(namebox_x, namebox_y, content.avatar_left_name);

    return true;
}
```

同理，由于Nine Slice的绘制涉及到许多坐标。考虑到padding也会对各种绘制位置产生影响，因此同样此处建议使用一些草稿纸来进行辅助，不然相关的计算特别容易乱……

文字的绘制可以完全复用之前的代码。这次为了方便直接进行复制修改，后续有空的话再进行代码抽象来减少冗余。
```
function draw_textbox_avatar_text() {
    var _char_display_length = 0; // 当前已经绘制过的字符的长度
    var _char_display_lines = 1; // 当前已经绘制过的字符的行数 默认1开始

    for (var _i = 0; _i < pg_cmd_index; _i++) {
        var _curC = resize_cmds[_i];
        switch (_curC.cmd) {
            case CMD_DRAW_SET_COLOUR:
                draw_set_color(_curC.value);
                break;
            case CMD_DRAW_NEW_LINE: // 滚动的实现分超过与不超过最大行数逻辑
                // 超过最大行数时 清屏(裁剪掉多余行 重新定位cmd_index后 重新开始绘制)
                if _char_display_lines >= textbox_line_maximun {
                    dialogue_cut_cmd_line(resize_cmds, textbox_line_maximun);
                    pg_cmd_index = 0;
                    return false;
                }
                // 没超过最大行数时 Y的坐标系向下平移 以及重置x
                _char_display_length = 0;
                _char_display_lines++;
                break;
            case CMD_DRAW_TEXT:
                var _x = textbox_x + _char_display_length;
                var _y = textbox_y + (textbox_line_height + textbox_line_space) * (_char_display_lines - 1);
                draw_text(_x, _y, _curC.value);
                _char_display_length += string_width(_curC.value);
                break;
        }
    }
    return true;
}
```

![2021060403](/assets/img/post/2021060403.png)

### 总结

由于这个对话框属于DEMO用，后续正式使用时一定会需要进行调整修改，因此此次实现尽可能留下了需要调整的参数。

对话框系统的学习目前我想应该需要告一段落了。最近这几天也不记得自己翻了多少次手册，但多多少少已经熟悉了内置变量与方法。接下来的几天估计会有不少房子的事情需要忙，希望期间能够利用一些空余的时间来，完善完善自己的游戏内容和玩法，以及整体GUI的设计。
