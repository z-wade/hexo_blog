title: Xib Developer Tip
date: 2016-04-18 11:47:11
tags: iOS

---

# Xib 自定义View的多种方式
4.关于File’s Owner

作用： 让xib也能像storyboard一样连线. (storyboard默认生成的时候,class为对应的viewController,因此能直接和代码中方法进行连线)。

做法：
(1). 将xib中的class设置为对应的viewController.
(2). 并在loadNib时将owner设置为对应的viewController(即一般为self, 对象)。

File’Owner不限于viewController，可以是任何类.
Class定义为哪个类,就能在哪个类中进行连线, 而要在loadNib时owner传入相应的对象即可调用相应方法。（必须保持一致，否则调用方法时会出现找不到方法）
