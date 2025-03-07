
review主要做一些再解读和再理解，好的工作反复咀嚼也不为过。

<span id="inline-blue">论文发布日期：2016.12 [CVPR]<p/span>

## 1. Introduction
<center><img src="http://chaserblog.test.upcdn.net/blogs/paper/FPN/str1.png" alt="" style="width:60%" /></center>

* 关于金字塔  
&emsp;&emsp;金字塔形具有尺度不变性的解释：物体的尺度在塔层间进行转换时都能得到很好的检测。        
&emsp;&emsp;那么存在的问题也是很直观的：没有刚好落在塔层附近的尺度，检测性能就不可避免地降低了。
<!-- more -->
* 特征金字塔和图像金字塔      
（1）图像金字塔的每一层都具有整张图片的完整信息；特征金字塔则不然，表征的是单尺度图片信息的不同尺度响应，是信息深组合的表征。两者携带的信息是有区别的。        
（2）作者认为图像金字塔只用在多尺度测试（时间和内存消耗大）造成了训练和测试的不一致。我觉得这个说法太笼统，而且结论不太对。并不是总要追求训练和测试的一致性。虽然采用图像金字塔时，模型训练没有尺度变化性，甚至可以假定只在一个尺度上预测效果好；但是检测时通过将图片缩放构造图像金字塔，能够实现搜索一定尺度范围内的物体，将其映射到检测性能好的尺度上，从而解决尺度变化性。        

* 语义鸿沟和语义特点  
&emsp;&emsp;卷积神经网络的降采样操作，天然地形成了金字塔形，但是不同深度的塔层具有**语义分歧**（这一点很重要，原来何大佬这么早就提到了这个问题）。        
&emsp;&emsp;高分辨率的图像具有的低级特征对于目标识别任务的表征能力是不好的（可见他不是像想的那样一上来就论述底层特征的定位优势，而是关注了其语义信息的缺失，大佬真的是大佬啊）。结合SSD实际用的是比较高的层进行分别的预测，也印证了上面说的——只用底层是不行的（SSD开始大家早就意识到这一点了）。之后才提到底层特征的未利用是一种缺憾，其对于小目标检测很重要。特征金字塔的一大亮眼表现是特征之间的融合，这个是之前金字塔都没有的，它使得金字塔的深度可以大大拓宽，而底层的这些弱语义信息又在定位上是很重要的。        
&emsp;&emsp;顺便，通过他的表述可以看出，作者的出发点是利用好强语义信息，而不是抓住定位好的底层信息，重点很明确。

## 2. Feature Pyramid Networks   
* ques:  关于何大佬说的<u>多级预测的特征独立于backbone架构</u>的说法不是很理解，反向传播一定会影响到backbone参数们甚至各个level之间也是互相影响的。这个独立是不是有其他含义？怎么理解？


* 关于结构理解  
&emsp;&emsp;检测网络可以单独作为分类网络使用，分类网络加个坐标预测分支就能检测，体现了两个任务的相关性。但是检测性能不好，所以有FPN特征融合等方法，这些方法和结构可以看做是为检测网络特化的，体现了这些结构对检测任务的适应性，这些差异性结构的提出建立在对任务本身具有相当的理解基础上的，这上面可以有很大的工作空间。

* 关于上采样  
&emsp;&emsp;这里是简化起见采用双线性插值，当然还有转置卷积等其他方法，上采样层面大的改进空间不大，稍有创新（如STDN这种）就能发表。但是可以从出发点思考一下：**有没有必要**？        

* 关于融合层设置
&emsp;&emsp;首先，通道256，后面一些其他论文来看，这个数目是比较合适的。但问题在于他统一为256是为参数共享提供便利，模仿传统使用一组参数进行多尺度预测；其次，没有非线性的加入，推测是因为想要保留原始信息的表征能力，这块只靠脑子想确实不靠谱（~~...不知道有没有可解释和量化的推理方法....~~）不过就实验结果来看，加不加影响不大。

* 自顶向下和横向连接    
&emsp;&emsp;下图是自顶向下的连接结构，将高层特征图不断上采样，作用是构造高分辨率语义信息丰富的特征图，但是这种特征图无论在哪个尺度上 ，<u>都缺乏精确的定位信息</u>，单独使用意义不大；横向连接将各层（主要是下面一点的层）的定位信息融合到上面传下来的语义信息中去。        
&emsp;&emsp;但是真正形成的FPN结构又不是单纯两个结构的叠加，而是连续不断地将高级语义融合和下传，避免语义鸿沟。
<center><img src="http://chaserblog.test.upcdn.net/blogs/paper/FPN/str2.png" alt="" style="width:50%" /></center>
      
## 3. Applications  
### 3.1 Feature Pyramid Networks for RPN
* 改动：RPN原来是单尺度特征图上采用多尺度anchor进行密集滑窗预测；采用FPN后，由于本身形成了多尺度特征图，可以直接在固定尺度特征图上分配相应尺度的anchor进行预测，即一张图对应一个尺度anchors（一个尺度即一个面积，五张图有五个不同面积的anchor，但是长宽比每个尺度三个，也就是每张图3anchor，共计15个）。对应关系是五个面积直接分配到五张图。  
* 参数共享：参数共享，也能取得较好的效果，说明检测头部确实在不同尺度下学到了类似的语义信息

### 3.2  Feature Pyramid Networks for Fast R-CNN 
* 层级分配：不像RPN直接指定，这里分配比较复杂。不细说，直接看论文公式，取的k0=4，也就是如果通过RPN得到的proposal（RoI）大小为ImageNet的224，则k=4，取当前proposal在C4的特征图进行各尺寸的anchor box 回归。其他尺寸的以此类推。
* 检测特头部：不论是哪个level，所有的参数共享  


## 4. Experiments on Object Detection

&emsp;&emsp;这个实验其实做得很好，因为FPN影响目标检测性能可以从region proposal和head两个方面下功夫，正好两个实验都涉及了。

### 4.1 Region Proposal with RPN  
<center><img src="http://chaserblog.test.upcdn.net/blogs/paper/FPN/exp1.png" alt="" style="width:80%" /></center>

* 语义鸿沟。通过对比实验，只加自顶向下结构，在不同level都放上强语义的特征图的上采样图，效果反而不好，作者归结为不同level的语义gap（但是我理解的gap是不同level之间不能直接融合，这一点值得再思考）。此外，实验证明，越是深度网络这种现象越明显，即使不同的head不共享参数也无济于事。（这是自然的，因为方法上是不对的，将强语义图直接上采样的做法就有问题）
* 自顶向下的问题。只是自顶向下的话，由于信息经过了多次的降采样上采样，丢失了精确的定位信息的，改进意义不大。
* ab对比来看，在更高的特征图上进行回归，得到的召回在小尺度效果更差，大尺度有不明显的提升。<u>印证了小特征图提高大尺度物体检测性能确实有用，但是太大就没有意义，信息被过度压缩了，所以检测任务也不能取太抽象的信息</u>。
* FPN广为诟病的速度慢其实比较明显，anchor的数目增长就能看出。
* FPN从召回率的角度来看，大物体的效果虽然也有提升的，但是提升很小。再结合实际检测时大物体的性能下降，**说明FPN结构破坏大物体检测性能是从proposal和检测头两个位置都有劣化**

### 4.2 Object Detection with Fast/Faster R-CNN        
#### 4.2.1 Fast R-CNN (on fixed proposals)            
&emsp;&emsp;选用固定的proposal进行试验，取自上一个实验的RPN得到的proposals。 
<center><img src="http://chaserblog.test.upcdn.net/blogs/paper/FPN/exp2.png" alt="" style="width:80%" /></center> 

* f只在最大特征图预测，结果并没有比全level预测差太多（TDM就是这样的），作者认为是RoIPooling具有区域（尺度）的不变性，所以能够一定程度抵御尺度变化，多level只是增强了这种不变性，这一点比较认同
* d的方式移除了自顶向下路径，只保留横向多尺度预测，效果很差。作者推测是大尺度特征图的语义信息太弱了，不能单独使用很容易出错。（SSD虽然是多尺度预测，但是别人的预测level都比较高，太低层的特征没用）

####4.2.2 Faster R-CNN (on consistent proposals)            
&emsp;&emsp;上面的实验选取现成的proposals作为对象不太现实，实际检测时RPN和检测是统一框架实现的。   
<center><img src="http://chaserblog.test.upcdn.net/blogs/paper/FPN/exp3.png" alt="" style="width:80%" /></center>

&emsp;&emsp;这里作者改进了之前的baseline获得更好的效果作为新的baseline为ab，主要是训练策略和参数调整。
* ab对照可以发现，过分往小尺度特征图上进行检测，并不会带来提升，答案是显然的因为信息过度压缩和抽象，导致单独用于检测已经不行了。另外，小尺度图像掉点特别厉害也很正常，小目标降采样都丢了，大目标相对掉点就少些。
* 使用FPN的反常大物体掉点如c。没有得到解答。
* RPN和Fast RCNN共享特征更加合适。（应该是符合MTL的原则）




<br>
<br>
<hr />