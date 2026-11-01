什么是OpenGL
很多人会说OpenGL是图形API，但是这样说其实是不准确的，因为OpenGL实际上是一种规范，是文字说明，并不是什么写好的函数库，是显卡厂商们，根据这个文字规范，去写好的函数接口，具体实现，这才是我们所说的OpenGL API。
OpenGL，GLFW，GLEW的关系：
简单来说，OpenGL是规定如何进行图形渲染，GLFW创建窗口，管理OpenGL上下文和输入事件，GLEW就是让你能使用现代OpenGL API。