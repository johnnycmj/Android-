# Fragment 生命周期

Fragment完全的生命周期：

![fragment_life.png](http://upload-images.jianshu.io/upload_images/6356977-67b615e3ed36346b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1. onAttach是Fragment刚加载进Acitivty时触发。
2. onCreateView 初始化View视图。
3. onViewCreate是视图完全初始化完成，这里一般可以做一些数据初始化等的工作。
4. odDetach是Fragment和Activity完全分离。