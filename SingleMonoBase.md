
=================================================
```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
```

`using System.Collections`
 - `System` (系统)：这是微软提供的一个超级巨大的底层仓库。 
 - `Collections` (集合)：意思是“把一堆东西收集在一起”。
 - 大白话比喻：这是一个**“老式混装螺丝盒”。
 - 这个工具箱比较老。它里面提供了一些能装数据的“盒子”，但是**不分门别类。主板螺丝、M.2硬盘螺丝、风扇螺丝全混在一起，拿出来用的时候还得自己分辨这是什么型号，很容易用错。现在写代码其实很少直接用它了，但为了兼容老系统，电脑通常会默认把它拿出来。

`using System.Collections.Generic`
 - `Generic`(泛型)：就是咱们刚才讲单例模式时提到的那个 `<T>`。它的意思是“具体类型待定，可以广泛适用”。
 - 大白话比喻：这是一个“现代高级分类螺丝盒”。
 - **你平时装机，螺丝肯定是按型号分格收纳的。这个工具箱就是干这个的！它提供了带有 `<T>` 分类标签的收纳盒（比如一种叫`List<T>` 的列表，一种叫 `Dictionary` 的字典）。**   
 - **在你的游戏里，你的背包系统（用来装子弹和急救包）、你的武器库管理（用来记录你解锁了手枪还是步枪），全都依赖这个高级工具箱里的零件。**

`using UnityEngine`
 - `Unity`：就是你正在使用的这个游戏引擎的名字。 
 - `Engine`：引擎。
 - 大白话比喻：这是 `Unity` 官方独家定制的“专用配件包”。
 - 前两个工具箱（`System`开头）是微软提供的通用螺丝刀，不管你是做网站、做系统还是做软件都能用。   
 - 但 `UnityEngine` 这个工具箱里，装的全部是**只有在做游戏时才用得到的专属零件。比如：代表 3D 模型的 `GameObject`、代表空间坐标的 `Transform`（前后左右）、以及允许脚本挂载到物体上的基础芯片 `MonoBehaviour`。  
 - 如果没有借用这个工具箱，你的代码就瞎了，完全不知道游戏引擎里的任何东西。


=================================================
```csharp
public class SingleMonoBase<T> : MonoBehaviour where T : SingleMonoBase<T>{
```
`public`                   **公开的**  
`class`                     **类**
`SingleMonoBase`      **“标识符“ 其实就是你自己起的名字的总称 可以是变量 类名 方法名**  

--------------------------------------------------------------
**问**
 既然所有名字都是自己起的，那怎么判断是类名 还是变量 还是方法名呢
**一**  **看它后面的**
 在代码里，**名字后面跟着的东西**决定了它的身份。

 `< >`                                   **类名**
 `（）`                                  **方法名**
 `空格 + 另一个名字`             **变量名**
 `点 .`                                  **可能是任何东西 要看后面跟什么决定                                                   后面跟括号是方法，后面跟名字是变量**

 举个例子 
 **`public class SingleMonoBase<T>`   后面是 `<T>` → 它是类名**

 `protected virtual void Awake()`   **后面是 () → 它是方法名**

 `public static T INSTANCE`             **前面有 T，空格后才是它 → 它是变量名**

**二：看它穿的衣服（修饰符）**
 **名字前面的单词，就像是穿的衣服**，也能暴露身份。

 `class` **类名**
 `void`, `int`, `string  `  **方法名 或 变量名**
 `public static`            **变量名**

 实战判断：
 `public class SingleMonoBase<T>`    **有 class → 百分百是类名**

 `public static T INSTANCE`              **有 static，后面没括号 → 百分百是变量名**
 `protected virtual void Awake()`     **有 void，后面有 () → 百分百是方法名**

--------------------------------------------------------------

 






=================================================
```csharp
public static T INSTANCE;
```
























=================================================



```csharp
protected virtual void Awake()
    {
        if (INSTANCE != null)
            Debug.LogError(name + "不符合单例模式");
        INSTANCE = (T)this;
    }

protected virtual void OnDestroy()
    {
        INSTANCE = null;
    }
}
```
