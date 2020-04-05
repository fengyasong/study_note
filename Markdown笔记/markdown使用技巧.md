# markdown笔记

## markdown基本语法

* 标题（从大到小取决于#的数量）

    一级标题一个# ，二级标题## 。。。

* 段落

    段落之间留有空行

* 字体样式
    1. 斜体*这是斜体样式*，有些也可以是 _这也是斜体_，但有些markdown流派会视为下划线文本。
    2. 粗体**粗体**。
    3. 斜体并加粗 ***斜体加粗***。
    4. 文字删除线 ~~文字删除线~~
* 插入链接
    - 方式1：[本项目地址](https://github.com/fengyasong/study_note)
    - 方式2：用尖括号包含url，<https://github.com/fengyasong/study_note>
* 图片

    和链接一样，只要在前面加个!
    
    ![图片名称](图片地址)

* 块引用 >
    > 与君初相识，犹如故人归
* 列表 * - +
    + (* - +)加空格无序列表
    - 数字+点+空格 有序列表
* 分隔符 （连续3个以上的 --- 添加分隔符）
-------
* 表格
    + 使用 -| 把内容分割为合适的表格样式
    - 使用 : 符号标识对齐

    姓名|性别|年龄
    :-:|-:|:-
    居中对齐|右对齐|左对齐
    张三|男|18

* 代码引入

    `console.log('该代码被引入了')`
    ```
    public static void main(String[] args) {
        SpringApplication.run(SeekfairyApplication.class, args);
    }
    ```
    
