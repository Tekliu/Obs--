**OpenGL**（Open Graphics Library）是一个跨平台的**图形编程接口**（API），它定义了一套函数，让你能跟 GPU 通信，告诉它"画点什么、怎么画"。
可以理解为显卡的遥控器。 一句话定位 **OpenGL 是软件（你的代码）和硬件（GPU）之间的中间层**，你调用它的函数，它帮你把**指令翻译给显卡驱动，驱动再指挥 GPU 干活**。 
顶点着色器、空间变换（MVP）、透视投影、光栅化——这些全是 OpenGL 渲染管线里的标准环节。OpenGL 规定了：
- 顶点数据怎么送进 GPU（ "glBufferData"）
- 顶点着色器和片段着色器怎么写（GLSL 语言）
- 矩阵怎么传（ "uniform"）
- 什么时候画（ "glDrawArrays"）
- 所以 OpenGL 就是把你脑子里那套"模型→世界→观察→裁剪→屏幕"的数学，变成能跑在显卡上的代码的工具。 几个关键事实 - 不是引擎：
- OpenGL 只管画图，不帮你管场景树、物理、音频——那是 Unity/Unreal 干的事 
- 状态机：OpenGL 本质上是一个巨大的[[状态机]]，你不断切换状态（当前颜色、当前纹理、当前着色器），然后下令绘制 - 版本：现在主流是 OpenGL 4.x（桌面），Web 上有 WebGL（基于 OpenGL ES），移动端用 OpenGL ES - 竞争者：Direct3D（Windows 独占，微软的）、Vulkan（新一代，更底层更高效）、Metal（苹果的） 一个最简的 OpenGL 程序骨架 // 1. 初始化窗口和上下文（GLFW / GLUT） // 2. 编译着色器（顶点 + 片段） // 3. 上传顶点数据到 GPU（VBO, VAO） // 4. 主循环： // 清屏 → 绑定着色器 → 上传 MVP 矩阵 → 绑定 VAO → glDrawArrays → 交换缓冲区 // 5. 清理 你之前在顶点着色器里写的 "gl_Position = MVP * vec4(aPos, 1.0)"，就是 OpenGL 规定的顶点着色器输出变量名——它告诉 OpenGL："这个顶点在裁剪空间里的坐标是这个"。 你现在要不要 1. 装一个 OpenGL 开发环境（GLFW + Glad + 一个窗口）跑通第一个三角形？ 2. 或者继续深挖透视投影矩阵的参数推导（fov, aspect, near, far 怎么影响画面）？ 3. 还是回到你原来的主线——用 PyTorch 搭 CNN 跑 MNIST？ 你说了算。