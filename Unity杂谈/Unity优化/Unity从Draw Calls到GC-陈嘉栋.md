https://www.zhihu.com/people/jiadongchen/posts?page=4
https://zhuanlan.zhihu.com/p/21316573
# CPU方面的优化
## 1.DrawCalls
DrawCall是CPU调用底层图形接口。比如有上千个物体，每一个渲染都需要取调用一次底层接口，而每一次的调用CPU都需要做很多工作，那么CPU必然不堪重负。但是对于GPU来说，图形处理的工作量是一样的。所以对DrawCall的优化，主要就是为了尽量解放CPU在调用图形接口上的开销。所以针对drawcall我们主要的思路就是每个物体尽量减少渲染次数，多个物体最好一起渲染。所以，按照这个思路就有了以下几个方案:
    - 使用Draw Call Batching,也就是描绘调用批处理。Unity在运行时可以将一些物体进行合并，从而用一个描绘调用来渲染他们。具体下面会介绍。
    - 通过把纹理打包成图集来尽量减少材质的使用。
    - 尽量少的使用反光、阴影，因为那么会是物体多次渲染。

###Draw Call Batching
首先我们要先理解为何2个没有使用相同材质的物体即使使用批处理，也无法实现DrawCall数量的下降和性能上的提升。
因为被“批处理”的2个物体的网格模型需要使用相同材质的目的，在于其纹理是相同的，这样才可以实现同时渲染的目的。因而保证材质相同，是为了保证被渲染的纹理相同。
因此，为了将2个纹理不同的材质合二为一，我们就需要进行上面理出的第二步，将纹理打包成图集，具体到合二为一这种情况，就是将2个纹理合成一个纹理。这样我们就可以只用一个材质来代替之前定的2个材质了。而DrawCall Batching本身，也还细分2种。
- Static Batching 静态批处理
  只要这些物体不移动，并且拥有相同的材质，静态批处理就允许引擎对任意大小的几何物体进行批处理操作来降低描绘调用
  那么如何使用静态批处理来减少DrawCall？你只需要明确指出哪些物体是静止的，并且在游戏中永远不会移动、旋转和缩放。想完成这一步，你只需要在检测器中将stat复选框打勾即可
  ![](https://pic3.zhimg.com/80/cc180d1f8e664410f0aefb8ac4cab2f6_hd.png)
  至于效果如何呢？
  举个例子:新建4个物体，分别是Cube,Sphere,Capsule,Cylinder，它们有不同的网络模型，但是也有相同的材质(Default-Diffuse)。
  首先，我们不指定它们是static的，DrawCall是4次，再讲它们设置为static，DrawCall次数变成1了，而Saved By batching的次数变成了3。

  静态批处理的好处很多，其中之一就是与下面要说的动态批处理相比，约束要少很多，所以一般推荐的是drawcall的静态批处理来减少draw call的次数。
- Dynamic Batching 动态批处理
  首先要明确一点，Unity3D的draw call动态批处理机制是引擎自动进行的，无需像静态批处理那样手动设置static。我们举一个动态实例化prefab的例子，如果动态物体共享相同的材质，则引擎会自动对draw call优化，也就是使用批处理。首先，我们将一个cube做成prefab,然后再实例化500次，看看draw call的数量。
  ~~~
  for(int i = 0; i < 500; i++)
  {
      GameObject cube;
      cube = GameObject.Instantiate(prefab) as GameObject;
  }
  ~~~
  draw call的数量:
  ![](https://pic2.zhimg.com/80/e9b473c0ffded069e0ea1eaa87e5577d_hd.png)
  可以看到draw call的数量为1，而saved by batching的数量是499。而这个过程中，我们除了实例化创建物体之外什么都没做。不错，unity3d引擎为我们自动处理了这种情况。
  但是有很多童鞋也遇到这种情况，就是我也是从prefab实例化创建的物体，为何我的draw call依然很高呢？draw call的动态批处理存在着很多约束。下面就演示一下，针对cube这样一个简单的物体创建，如果过稍有不慎就会造成draw call飞涨的情况吧。
  我们同样是创建500个物体，不同的是其中的100个物体，每个物体的大小都不同，也就是Scale不同。
  ~~~
  for(int i = 0; i < 500; i++)
  {
      GameObject cube;
      cube = GameObject.Instantiate(prefab) as GameObject;
      if(i/100 == 0)
      {
          cube.transform.localScale = new Vector3(2 + i, 2 + i,2 + i);
      }
  }
  ~~~
  我们看到draw call的数量上升到了101次，而saved by batching的数量也下降到了399。各位可以看到，仅仅是一个简单的cube的创建，如果scale不同，竟然也不会去做批处理优化。这仅仅是动态批处理机制的一种约束，那我们总结一下动态批处理的约束，各位也许也能从中找到为何动态批处理在自己的项目中不起作用的原因：
  1. 批处理动态物体需要在每个顶点上进行一定的开销，所以动态批处理仅支持小于900顶点的网格物体
  2. 如果你的着色器使用顶点位置，发现和UC值三种属性，那么你只能批处理300顶点以下的物体；如果你的着色器需要使用顶点位置，发现，UV0，UV1和切向量，那么你只能批处理180顶点以下的物体。
  3. 不要使用缩放。分别拥有缩放大小(1,1,1)和(2,2,2)的两个物体将不会进行批处理。
  4. 统一缩放的物体不会与非同一所放的物体进行批处理。
  5. 使用缩放尺度(1,1,1)和(1,2,1)的两个物体将不会进行批处理，但是使用缩放尺度(1,2,1)和(1,3,1)的两个物体将可以进行批处理。
  6. 使用不同材质的实例化物体(instance)将会导致批处理失败。
  7. 拥有lightmap的物体含有额外(隐藏)的材质属性，比如lightmap的偏移和缩放系数等。所以，拥有lightmap的物体将不会进行批处理(除非他们指向lightmap的同一部分)。
  8. 多通道的Shader会翻盖批处理操作。比如，几乎unity中所有的着色器在前向渲染中都支持多个光源，并为它们有效地开辟多个通道。
  9. 预设体的实例会自动地使用相同的网络模型和材质。
## 2.物理组件(Physics)
曾几何时，匹夫在做一个策略类游戏的时候需要在单元格上排兵布阵，而要侦测到哪个兵站在哪个格子匹夫选择使用了射线，由于士兵单位很多，而且为了精准每一帧都会执行检测，那时候cpu的负担叫一个惨不忍睹。后来匹夫果断放弃了这种做法，并且对物理组件产生了心里阴影。
这里匹夫只提2点比较重要的优化措施：
  1. 设置一个合适的Fixed Timestep:
  ![](https://pic2.zhimg.com/80/1fb448a3b46c904490a1f0894bb34075_hd.png)
  那何谓“合适”呢？首先我们要搞明白Fixed Timestep和物理组件的关系。物理组件，或者说游戏中模拟各种物理效果的组件，最重要的是什么呢？计算啊。对，需要通过计算才能将真实的物理效果展现在虚拟的游戏中。那么Fixed Timestep这货就是和物理计算有关的啦。所以，若计算的频率太高，自然会影响到cpu的开销。同时，若计算频率达不到游戏设计时的要求，有会影响到功能的实现，所以如何抉择需要各位具体分析，选择一个合适的值。
  2. 就是不要使用网格碰撞器(mesh collider)：为啥？因为实在是太复杂了。网格碰撞器利用一个网格资源并在其上构建碰撞器。对于复杂网状模型上的碰撞检测，它要比应用原型碰撞器精确的多。标记为凸起的(Convex)的网格碰撞器才能够和其他网格碰撞器发生碰撞。各位上网搜一下mesh collider的图片，自然就会明白了。我们的手机游戏自然无需这种性价比不高的东西。

## 3.GC(什么？GC不是处理内存问题的嘛？CPU使用GC去处理内存)
处理内存，却让CPU受伤的GC
在CPU部分聊GC，感觉是不是怪怪的？虽然gc是用来处理内存的，但的确增加的是CPU的开销。因此它的确能达到释放内存的效果，但代价更加沉重，会加重CPU的负担，因此对于gc的优化目标就是尽量少的触发gc。
首先我们要明确所谓的gc是mono运行时的机制，而非Unity3D游戏引擎的机制，所以GC也主要是针对Mono的对象来说的，而它管理的也是Mono的托管堆。搞清楚这一点，你也就明白了GC不是用来处理引擎assets(纹理、音效等等)的内存释放，因为Unity3D引擎也有自己的内存堆而不是和Mono一起使用所谓的托管堆。

其次我们要搞清楚什么东西会被分配到托管堆上？不错咯，就是引用类型咯。比如类的实例，字符串，数组等等。而作为int,float，包括结构体struct其实都是值类型，它们会被分配在堆栈上而非堆上。所以我们关注的对象无外乎就是类实例，字符串，数组这些了。

那么GC什么时候会触发呢?两种情况：
1. 首先当然是我们的堆的内存不足时，会自动调用GC。
2. 其次呢，作为编程人员，我们自己也可以手动的调用GC。

所以为了达到优化CPU的目的，我们就不能频繁的触发GC。而上问也说了GC处理的是托管堆，而不是Unity3D引擎的那些资源，所以GC的优化说白了也就是代码的优化。那么匹夫觉得有以下几点是需要注意的：
1. 字符串连接的处理。因为将两个字符串连接的过程，其实是生成一个新的字符串的过程。而之前的旧的字符串自然而然就成为了垃圾。而作为引用类型的字符串，其空间是在堆上分配的，被弃置的旧的字符串的空间会被GC当作垃圾回收。
2. 尽量不要使用foreach，而是使用for。foreach其实会涉及到迭代器的使用，而据传说每一次循环所产生的迭代器会带来24Bytes的垃圾。那么循环10次就是240Bytes。
3. 不要直接访问gameobject的tag属性。比如if(go.tag == "human")最好换成if(go.CompareTag("human"))。因为访问物体的tag属性会在对上额外的分配空间。如果在循环中这么处理，留下的垃圾就可想而知了。
4. 使用“池”,以实现空间的重复利用。
5. 最好不用LINQ的命令，因为它们会分配临时的空间，同样也是GC收集的目标。而且我很讨厌LINQ的一点就是它有可能在某些情况下无法很好的进行AOT编译。比如“OrderBy”会生成内部的泛型类“OrderedEnumerable”。这在AOT编辑时是无法进行的，因为它只是在OrderBy的方法中才使用。所以如果你使用了OrderBy,那么在IOS平台上也许会报错。

## 4.代码质量
聊到代码这个话题，也许有人会觉得匹夫多此一举。因为代码质量因人而异，很难像上面提到的几点，有一个明确的评判标准。也是，公写公有理，婆写婆有理。但是匹夫这里要提到的所谓代码质量是基于一个前提的：Unity3D是用C++写的，而我们的代码是用C#脚本来写的，那么问题就来了~脚本和底层的交互开销是否需要考虑呢？也就是说，我们用Unity3D写游戏的“游戏脚本语言”，也就是C#是由mono运行时托管的。而功能是底层引擎的C++实现的，“游戏脚本”中的功能实现都离不开对底层代码的调用。那么这部分的开销，我们应该如何优化呢？

1. 以物体的Transform组件为例，我们应该只访问一次，之后就将他们的引用保存，而非每次使用都去访问。这里有人做过一个小实验，就是对比通过方法GetComponent<Transform>()获取Transform组件，通过MonoBehaviour的tranform属性去取，以及保留引用之后再去访问所需要的时间：
    - GetComponent = 619ms
    - Monobehaviour = 60 ms
    - CachedMB = 8ms
    - Manual Cache = 3ms
2. 如上所述，最好不要频繁使用GetComponent,尤其是在循环中。
3. 善于使用OnBecameVisiable()和OnBecameVisable()来控制物体的update()函数的执行以减少开销。
4. 使用内建的数组，比如用Vector3.zero而不是new Vector(0,0,0);
5. 对于方法的参数的优化：善于使用ref关键字。值类型的参数，是通过将实参的值复制到形参，来实现按值传递的方法，也就是我们通常说的按值传递。复制嘛，总会让人感觉很笨重。比如Matrix4x4这样比较复杂的值类型，如果直接复制一份新的，反而不如将值类型的引用传递给方法作为参数。

# GPU的优化
GPU与CPU不同，所以侧重点自然也不一样。GPU的瓶颈主要存在在如下的方面：
1. 填充率，可以简单的理解为图形处理单元每秒渲染的像素数量。
2. 像素的复杂度，比如动态阴影，光照，复杂的shader等等
3. 几何体的复杂度(顶点数量)
4. 当然还有GPU的显存带宽

那么针对以上4点，其实仔细分析我们就可以发现，影响GPU性能的无法就是2大方面，一方面顶点数量过多，像素计算过于复杂。另一方面就是GPU的显存带宽。那么针锋相对的两方面举措也就十分明显了。
1. 减少顶点数量，简化计算复杂度。
2. 压缩图片，以适应显存带宽

##减少绘制的数目
那么第一个方面的优化也就是减少顶点数量，简化复杂度，具体的举措就总结如下了：
  - 保持材质的数目尽可能少。这使得Unity更容易进行批处理
  - 使用纹理图集(一张大贴图里包含了很多子贴图)来代替一系列单独的小贴图。它们可以更快地被加载，具有很少的状态转换，而且批处理更友好。
  - 如果使用了纹理图集和共享材质，使用Renderer.sharedMaterial来代替Renderer.material。
  - 使用光照纹理(lightmap)而非实时灯光。
  - 使用LOD，好处就是对那些离得远，看不清的物体的细节可以忽略。
  - 遮挡剔除(Occlusion culling)
  - 使用mobile版的shader

##优化显存带宽
第二个方向呢？压缩图片，减小显存带宽的压力。
  - OpenGL ES 2.0使用ETC1格式压缩等等，在打包设置那里都有
  - 使用mipmap

###使用MipMap
这里匹夫着重介绍一下MipMap到底是啥。因为有人说过MipMap会占用内存呀，但为何又会优化显存带宽呢？那就不得不从MipMap是什么开始聊起。一张图其实就能解决这个疑问。
![](https://pic4.zhimg.com/80/ee1bf6293d39daf6df58343be1e7ad7f_hd.jpg)
上面是一个mipmap如何存储的例子，左边的主图伴有一系列逐层缩小的备份小图

是不是很一目了然呢？Mipmap中每一个层级的小图都是主图一个特定比例的缩小细节的复制品。因为存了主图和它的那些缩小的复制品，所以内存占用会比之前大。但是为何又优化了显存带宽呢？因为可以根据实际情况，选择适合的小图来渲染。所以，虽然会消耗一些内存，但是为了图片渲染的质量(比压缩要好),这种方式也是推荐的。

#内存的优化
既然要聊Unity3D运行时候的内存优化，那我们自然首先要知道Unity3D游戏引擎是如何分配内存的。大概可以分为三大部分：
1. Unity3D内部的内存
2. Mono的托管内存
3. 若干我们自己引入的DLL或者第三方DLL所需的内存

第三类不是我们关注的重点，所以接下来我们会分别看一下Unity3D内部内存和Mono托管内存，最后还将分析一个官网上Assetbundle的案例来说明内存的管理。

##Unity3D内部内存
Unity3D的内部内存都会存放一些什么呢？各位想一想，除了用代码来驱动逻辑，一个游戏还需要什么呢？对，各种资源。所以简单总结一下Unity3D内部内存存放的东西吧：
- 资源:纹理、网络、音频等等
- GameObject和各种组件
- 引擎内部逻辑需要的内存：渲染器，物理系统，粒子系统等等

##Mono托管内存
因为我们的游戏脚本是用C#写的，同时还要跨平台，所以带着一个Mono的托管环境显然必须的。那么Mono的托管内存自然就不得不放到内存的优化范畴中进行考虑。那么我们所说的Mono托管内存中存放的东西和Unity3D内部内存中存放的东西究竟有何不同呢？其实Mono的内存分配就是很传统的。运行时内存的分配了：
  - 值类型：int、float、struct、bool。它们都存放在堆栈上(不是堆所以不涉及GC)
  - 引用类型：其实可以狭义的理解为各种类的实例。比如游戏脚本中对游戏引擎各种控件的封装。其实很好理解，C#中肯定要有对应的类去对应游戏引擎中的空间。那么这部分就是C#中的封装。由于是在堆上分配，所以会涉及到GC。

而Mono托管堆中的那些封装的对象，除了在在Mono托管堆上分配封装类实例化之后所需要的内存之外，还会牵扯到其背后对应的游戏引擎内部空间在Unity3D内部内存上的分配。
举一个例子：
  一个在.cs脚本中声明www类型的对象www,Mono会在Mono托管堆上为www分配它所需要的内存。同时，这个实例对象背后的所代表的引擎资源所需要的内存也需要被分配。
  一个www实例背后的资源：
    - 压缩的文件
    - 解压缩所需的缓存
    - 解压缩之后的文件
  如图：
  ![](https://pic1.zhimg.com/80/a977a65a7d5cc1b88f67753cc0d38dec_hd.png)

那么下面就举一个AssetBundle的例子:
##Assetbundle的内存处理
以下载Assetbundle为例子，聊一下内存的分配。匹夫从官网的手册上找到了一个使用Assetbundle的情景如下:
~~~
IEnumerator DownloadAndCache (){
        // Wait for the Caching system to be ready
        while (!Caching.ready)
            yield return null;

        // Load the AssetBundle file from Cache if it exists with the same version or download and store it in the cache
        using(WWW www = WWW.LoadFromCacheOrDownload (BundleURL, version)){
            yield return www; //WWW是第1部分
            if (www.error != null)
                throw new Exception("WWW download had an error:" + www.error);
            AssetBundle bundle = www.assetBundle;//AssetBundle是第2部分
            if (AssetName == "")
                Instantiate(bundle.mainAsset);//实例化是第3部分
            else
                Instantiate(bundle.Load(AssetName));
                    // Unload the AssetBundles compressed contents to conserve memory
                    bundle.Unload(false);

        } // memory is freed from the web stream (www.Dispose() gets called implicitly)
    }
}
~~~
内存分配的三个部分已经在代码中标识了出来
1. Web Stream:包括了压缩的文件，解压所需的缓存，以及解压后的文件
2. AssetBundle:Web Stream中的文件的映射，或者说引用
3. 实例化之后的对象:就是引擎的各种资源文件了，会在内存中创建出来。

那就分别解析一下:
~~~
WWW www = WWW.LoadFromCacheOrDownload (BundleURL, version)
~~~
1. 将压缩的文件读入内存中
2. 创建解压所需的缓存
3. 将文件解压，解压后的文件进入内存
4. 关闭掉为解压创建的缓存

~~~
AssetBundle bundle = www.assetBundle;
~~~
1. AssetBundle此时相当于一个桥梁，从Web Stream解压后的文件到最后实例化创建的对象之间的桥梁
2. 所以AssetBundle实质上是Web Stream解压后的文件中各个对象的映射。而非真实的对象。
3. 实际的资源还存在WebStream中，所以此时要保留Web Stream。

~~~
Instantiate(bundle.mainAsset);
~~~

1. 通过AssetBundle获取资源，实例化对象
最后各位可能看到了官网中的这个例子使用了:
~~~
using(WWW www = WWW.LoadFromCacheOrDownload (BundleURL, version)){
}
~~~
这种using的用法。这种用法其实就是为了在使用完Web Stream之后，将内存释放掉的。因为www也继承了IDispose接口，所以可以使用using的这种用法。其实相当于最后执行了：
~~~
www.Dispose();
~~~
ok,Web Stream被删除掉了。那还有谁呢？对Assetbundle。那么使用
~~~
bundle.Unload(false);
~~~

#使用Unity Profiler工具检测内存
这篇文章当时写的时候略显仓促，因此并没有特别介绍Unity Profiler工具，也更太不上用Unity Profiler工具来检测内存的使用状态了。但是使用Unity Profiler工具来检测还是十分必要的，下面就简单补充一下这方面的知识。
![](https://pic1.zhimg.com/80/3ab206cc4ffd8ed9f302c259a46a2828_hd.png)
在Profiler工具中提供了两种模式供我们检测内存的使用情况，即简易模式和详细模式。在简易模式中，我们可以看到总的内存列出了两列，即Used Total(使用总内存)和Reserved Total(预定总内存)。Used Total和Reserved均是物理内存，其中Reserved是Unity向系统申请的总内存，Unity底层为了不经常向系统申请开辟内存，开启了较大一块内存作为缓存，即所谓的Reserved内存，而运行时，unity所使用的内存首先是向Reserved中来申请内存，当不使用时也是向Reserved中释放内存，从而来保证有运行流畅性。一般来说，Used Total越大，则Reserved Total越大，而当Used Total降下去后，Reserved Total也是会随之下降的(但并不一定与Used Total同步)。
Unity3D的内存从答题上可以分为以下几个部分:
1. Unity: 位unity3D的底层代码所分配的内存。
2. Mono:即托管堆。Mono运行时在运行游戏脚本时所需要的内存，换句话说托管堆的大小与我们的GameObject数量、资源量无关，仅是脚本代码造成的。这部分内存是有垃圾回收机制的。
3. GfxDriver:可以理解为GPU显存开销，主要由Texture,Vertex buffer以及index buffer组成。所以尽可能地减少或释放Texture和mesh等资源，即可降低GfxDriver内存。
4. FMOD: 音频的内存开销。
5. Profiler
在简易模式下的监视器最下方，则列出了常见的一些资源以及它们所消耗的内存。
1. 纹理
2. 网格
3. 材质
4. 动作
5. 音频
6. 游戏对象数量







# 内存
