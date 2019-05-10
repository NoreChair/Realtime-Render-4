# Efficient Shading 

# Deferred Shading 
- 早期 Deferred Shading 流程：
    - 将 geometry 分离与 material & light  ,将信息写入 G-buffer (Diffuse,Specular,Normal 等)中，G-buffer 中的数据做自定义，通常使用 MultiRenderTarget 的功能将信息存入不同 Texture 中，Target 一般不超过3个。压缩数据的方法各式各样，平衡与材质与性能之间，一般都有严格的限制。
    - 每盏灯光与材质分别计算，即计算复杂度：F = geometry + material * light ;普通 Forward Shading 复杂度为： F = geometry * material * light ;最终将所有数据结合为最终图像。 
    
- 后出现的 Deferred Light 进一步解耦材质与光源，使复杂度简化为 : F = geometry + material + light。流程大致如下： G-buffer → Light pass → render 
    - G-buffer 的生成是固定的，但是相对于早期的 Deferred Shading 相比，可以去掉很多信息，因为最终还需要在 render 步骤中重新渲染几何 ，所以材质再后面具体计算。一般只需要 position 和 normal，position 可以由 depth 推导，thus 只需要一张 render texture 就足够。
    - Light pass 是 Deferred Light 的重点，将所有 light volume 一次绘制 记录到 L-buffer 中，可以由 depth 推出此 pixels 是否被遮挡 ，再判定是否有相交于表面，将 diffuse 和 specular 记录到 L-buffer 中。代价为计算 L-buffer ，光源重复覆盖时，同一个 pixels 会 多次的读取 G-buffer数据。
    - 再次渲染全部物体，光照结果直接取于 L-buffer ，材质并没有太大限制 , 增加的是 shader 的复杂度 ，并且由于最后的render 中存在几何信息，MSAA 可以被使用。
- 关于这两种方法的更多细节 ，可以参考[龚大的博客](http://www.klayge.org/2011/01/11/klayge%E4%B8%AD%E7%9A%84%E5%BB%B6%E8%BF%9F%E6%B8%B2%E6%9F%93%EF%BC%88%E4%B8%80%EF%BC%89/)。
----------
# Decal Rendering

待续

----------
# Tiled Shading
- Tiled Shading 分为 Tiled Deferred Shading 和 Tiled Forward Shading（Forward +），基础思想都是将光源进行 Tiled 划分，在着色阶段根据所在 Tile 获取 light list ：
    - 将屏幕空间按像素划分为多个 Tile : 32 x 32 piexls  per tile 。指定的 1920 x 1080 ,划分为 60 x 34 块 tile , view frustum 的 near plane 和 far plane 进行此划分，形成每个 tile 自己的 frustum 
    - 将每个 tile 与 light volume 进行相交测试，可以使用 compute shader 或 cpu 多线程进行加速计算，一般选用 compute shader ，将结果存入 imagebuffer 中，再着色时可以直接取到 imagebuffer 中该 tile 对应的 light data 。
    - 对于光源的剔除，进行一或多次的采样确定 Zmin Zmax 快速剔除与表面无交互或空 mesh 的 tile，Zmin Zmax 是否值得进行计算需依据场景类型，或只选其一 。但是对于同一个 tile , z range 过大的情况，需要多一个 Half Z 。但即使是这样也不能避免所有情况，所以有人提出了 2.5D Culling 的方法，后面会再提及。

Tiled Deferred Shading 可以在绘制 L-Buffer 时，绘制 quad ，在 fragment 中采取 tile data 。

    - 这样只需要一次的 G-buffer 采样就可以将所有该像素的光照贡献计算出，结果图像也只需写入一次，不必多个 Light 使用 blend 叠加。
    - 确保每个 warps/wavefront 的计算过程一致，在渲染 opaque 后 transparent 可以使用同样的 light list data，所有的光照都计算在同一个 pass 中。
    - 虽然未必每个 tile 中的 light volume 都覆盖整个 tile ，但是这并不影响最终结果，也节省下了很多带宽和计算成本。

Tiled Forward Shading 使用同样的方法：

    - 首先进行一次 z-prepass 避免最终的图像出现 overdraw ，并且进行光源的剔除。
    - 执行 compute shader 获取 tile light list data 。第二次几何绘制时执行 forward shading ，tile 的位置获取来源于几何的 screen space position .
    

[此处](https://mynameismjp.wordpress.com/2012/03/31/light-indexed-deferred-rendering/)给出了一些对比测试数据，可以看到 Tiled Shading 的效率在 上千的光源绘制中是占据绝对优势地位的。


## 2.5D Culling 

此方法用于精确剔除 light 是否影响 geometry ：

- 确定 Zmin Zmax 
- 从 Zmin 到 Zmax 划分 n 块区域，每块区域用一个 Bitmask 标记是否在光照影响范围或是否处于几何范围 。如下图。
- Bitmask 相与的结果即为是否写入 light list 。
- 此方法在上千的光源场景中，相比未使用性能占据主导地位，但是不明显。
![](https://paper-attachments.dropbox.com/s_C6129561BCABBEE522C5180BE9A02C36E6B77B6CB08AABA19DA76F828D891446_1557393050585_image.png)

----------
# Clustered Shading
- Clustered Shading 是在 Tiled Shading 的 3D 延伸版。在 Tiled shading 中，我们说过 Tiled Shading 所切分的 Screen Space Tile ，所形成的细长的视椎体，存在 depth disccontinuities 的问题。即：Zmax Zmin 过大，中间可能存在与表面无交互的光源存入 light list。虽然针对此问题可以使用剔除的方法，但是成本过大，z-prepass 与确定 Zmin Zmax 的深度采样是必须的。Clustered Shading 从设计的方向就根除了这个问题：
    - 首先与 Tiled Shading 一样 ，都需要划分 Tile , 但是多一步划分 Depth 。即沿着 Z 方向再进行划分。通常划分的 n 取 16 或 32 。如图下，如何进行 Depth 的划分是 clustered shading 的一个重要步骤，后续会讲到。
![](https://paper-attachments.dropbox.com/s_C6129561BCABBEE522C5180BE9A02C36E6B77B6CB08AABA19DA76F828D891446_1557457610900_image.png)

    - 得到 clusters 后同 tile 相似，使用 compute shader 计算相交性，计算相交性是 Cluster Shading 的另一个重点后面会提及。在 render 阶段，无论 opaque 或 transparent 都可以直接使用其所在 cluster 进行 light list 的获取。此方法大大减少了与表面无交的无效光源访问，不需要 z-prepass ，不需要确定 Zmin Zmax ，性能的最糟情况比 Tiled Shading 更好，独立于场景只与 light volume 和 frustum 相关，光源数量可以达到上百万。
    
## Depth 划分方法
- 划分 depth 存在一些问题，首先不能以均等距离的划分，这会很大的浪费划分 cluster 的意义。对我们而言，越近的距离我们需要更密集的 cluster ，更远的灯光，相对于图像的贡献度会更低，所以我们使用更大距离。 
- 而一个视椎体的 far plane 可能非常远，Just Cause 3 中视椎体的 near :0.1 far:50000 。这种情况下需要一个 “far plane” for light cluster 。所以他们使用的方法是：确定一个 light cluster 的 far：500 。然后在 0.1 ~ 500 中进行 depth 划分，超过此 far 的区块，使用 lod 方法将光源换为 particle 或其他。
- 而第一层级的划分可能过近，比如 default ：0.1，0.17， 0.29……..，在这种划分内，可能完全没有光源存在，为了更合理的划分，需要一个 “near plane ” for light cluster ，所以改进为：0.1, 5.0, 6.8, 9.2….. 
- 更详细的内容可以看 [这里](http://www.humus.name/Articles/PracticalClusteredShading.pdf) ，包括了如何分配数据结构和详细的 Tile 和 Cluster 的区别，以及此方法对于 GPU 执行的性能影响，如何进行相交性测试等。


## 相交性测试与数据存储
- 相交性测试的目的是为了找出每个 Cluster 所对应的 light list ，为了更快更准确的寻找到相对应的 light list ，这里示例其中一种方法。
- 首先进行 Frustum Test ，处于视椎体内的所有光源，从近到远进行编号。首先进行一次 xy 水平 z 方向上的划分，确认每个 slices 的区域，将与此区域相交的光源记录下来，我们称这个数据为 z-bin。再对每 tile 进行测试(没错，就是 Tiled Shading 中的 tile )，确认每 tile 相交的数据称为：tile lists。最后每 cluster 的 light list ，就是 z-bin & tile lists 的结果。如下图。这样做的好处是可以拆分三维数据转为一个二维的 tile list 和一个一维的 z-bin ，大大节省了带宽，多的是需要在 fragment 中确认数据的时候多出一步。这种方法会存在一些“误差”，但是 Drobot 证明此误差在人为场景中很少存在。使用 fragment 和 compute shader 能够极好的剔除。
![](https://paper-attachments.dropbox.com/s_C6129561BCABBEE522C5180BE9A02C36E6B77B6CB08AABA19DA76F828D891446_1557479601277_image.png)

- 而 [这里](http://www.diva-portal.org/smash/get/diva2:839812/FULLTEXT02.pdf) 使用了 DX12 通过 Conservative Rasterization 两次 pass 进行更为精确的 Cluster 与 light volume 测试，可以作为参考。

----------
# 最后
- [这里](https://www.3dgep.com/forward-plus/) 你可以找到完整的  Deferred Shading /Tiled Shading 方法，作者讲的非常详细完善，[这里](https://www.3dgep.com/volume-tiled-forward-shading/) 你可以找到 Clustered Shading 的完整方法，同一个作者写的，代码与方法介绍都有，非常适合学习。
- [GDC2013](https://www.gdcvault.com/browse/gdc-13/play/1017627) 的演讲上，详细分析了 Tiled Based 下 deferred 和 forward 的性能差别，在 MSAA 未开启时，forward 优势很大，开启后 deferred 完胜 forward。
- Decal Render 可以在 [这里](http://blog.wolfire.com/2009/06/how-to-project-decals/) 得到其中一种方法，而 [这里](http://martindevans.me/game-development/2015/02/27/Drawing-Stuff-On-Other-Stuff-With-Deferred-Screenspace-Decals/) 和 [这里](https://bartwronski.com/2015/03/12/fixing-screen-space-deferred-decals/) 详细的讨论了 Deferred Decal 的方法和优化。 

