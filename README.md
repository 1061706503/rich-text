背景:由于课程/labs产出的富文本与游戏使用的富文本格式很多地方不同，但却没有一个有效的文档约定这种格式的转换，所以在迭代中出现了多次富文本不符合格式产生的风险与更改，现前端与游戏组将富文本的转化进行一个约定，所有的格式转化都根据这个约定补充扩展处理。

文本转换
1、Unity富文本转换统计(所有富文本格式以约定为准)
格式
课程产出 - 前端富文本
游戏使用 - Unity富文本
备注
粗体
<strong>粗体</strong>
<b>粗体</b> 可嵌套
初始化
课程：李富
斜体
<em>斜体</em>
<i>斜体</i> 可嵌套

字体
<span style="font-size:30px">大小</span>
<size=12>大小</size>

颜色
<span style="color:#f39c12">颜色</span>
<color=#ff000000>颜色</color>
<color=res>颜色</color>
可嵌套

下划线
<u>下划线</u>
<underline></underline>
不支持嵌套

换行
<p>换行</p>
\n

首行缩进
&nbsp;&nbsp;
<indentation></indentation>
SP90添加
游戏：凌华浩
课程：李富
文字底色
<span style="background-color:#f39c12">颜色</span>
<bgc=#ff000000>颜色</bgc>
可嵌套
SP95添加
游戏：王澍 
课程：黄辉
标签
<span style="标签样式">标签名称</span>
<jojotag data-tag="标签id" data-tag-text="标签名称"></jojotag>
标签需要特殊转换：查看标签功能使用
SP101添加
游戏: ~
课程：李富
空格
<span style="空格样式"></span>
<jojospace data-space-id="空格id"></jojospace>
SP107添加
游戏：钟松峰
课程：李富
数学公式
<formula>公式</formula>
<formula>\\sqrt{65522+5\\times988</formula>
SP116添加
游戏：董云龙
课程：黄辉
阅读提示

<prompt id='xxx'></prompt>
sp116添加
游戏：胡林
课程：王芸
块(空格)
<span style="块样式">1方框</span>
<span style="块样式">2圆圈</span>
<span style="块样式">3{Name}</span>
<span style="块样式">4{Time}</span>
<span style="块样式">5{Title}</span>
<span style="块样式">6{TotalWriteCount}</span>
<block data-style="1|2" data-block="块id"></block>
data-style
1：方框
2：圆圈
3：姓名标签
4：时间标签
5：标题标签
6：字数标签
7: 小输入框
8: 中输入框
9: 大输入框
10: 符号输入

SP117 添加
游戏：陈炜程
课程：李富、何浩
居左
<span style="text-align: left">居左</span>
<jleft>居左</jleft>

居中
<span style="text-align: center">居中</span>
<jcenter>居中</jcenter>

居右
<span style="text-align: right">居右</span>
<jcenter>居右</jcenter>

益智单位
<span style="益智单位样式">益智单位</span>
<undergroup>益智单位</undergroup>


2、颜色内容
● 字符串颜色只能使用下方图片中的
● 16进制可随意使用


维护与扩展
1、以约定文档为准课程前端进行富文本组件开发输出对应unity富文本。
2、如果后续有任何的改动或者扩展，双方需要提前同步与调研方案(方便提前对组件进行维护升级)。

富文本构建图示
富文本渲染时序
解释器(内部处理)流程
初始化解析流程
输入解析流程
待续...
前端组件
前端组件基于WangEditor简单封装，与插件扩展。
wangEditor文档
https://www.wangeditor.com/
wangEditor基于slate.js开发，只需要在wangEditor上再做简单的封装，如双向绑定，数据序列化处理即可。
slate 文档
https://docs.slatejs.org/
富文本的数据结构
wangEditor.children 产生的数据结构是继承自 slate生成的数据结构如下图所示


其中根据wangEditor 的 getHtml 方法源码分析，就是根据这个树的结构进行遍历做序列化


序列化方案
slate序列化参考文档地址
https://docs.slatejs.org/concepts/10-serializing#html

参考getHtml和文档使用数据序列化生成对应的标签提供给游戏，在数据回显的时候再逆序列化将字符串处理成slate数据结构方便回显。

正向序列化示例
import escapeHtml from 'escape-html'
import { Text } from 'slate'

const serialize = node => {
  if (Text.isText(node)) {
    let string = escapeHtml(node.text)
    // 处理对应的富文本标签 序列化之后返回的标签
    if (node.bold) {
      string = `<strong>${string}</strong>`
    }
    return string
  }

  const children = node.children.map(n => serialize(n)).join('')

  switch (node.type) {
    case 'quote':
      return `<blockquote><p>${children}</p></blockquote>`
    case 'paragraph':
      return `<p>${children}</p>`
    case 'link':
      return `<a href="${escapeHtml(node.url)}">${children}</a>`
    default:
      return children
  }
}

逆向序列化示例
import { jsx } from 'slate-hyperscript'

const deserialize = (el, markAttributes = {}) => {
  if (el.nodeType === Node.TEXT_NODE) {
    return jsx('text', markAttributes, el.textContent)
  } else if (el.nodeType !== Node.ELEMENT_NODE) {
    return null
  }

  const nodeAttributes = { ...markAttributes }


  switch (el.nodeName) {
      // 处理对应的dom标签，反序列化将属性赋值
    case 'strong':
      nodeAttributes.bold = true
  }

  const children = Array.from(el.childNodes)
    .map(node => deserialize(node, nodeAttributes))
    .flat()

  if (children.length === 0) {
    children.push(jsx('text', nodeAttributes, ''))
  }

  switch (el.nodeName) {
    case 'BODY':
      return jsx('fragment', {}, children)
    case 'BR':
      return '\n'
    case 'BLOCKQUOTE':
      return jsx('element', { type: 'quote' }, children)
      // 处理段落的逆向序列化
    case 'P':
      return jsx('element', { type: 'paragraph' }, children)
    case 'A':
      return jsx(
        'element',
        { type: 'link', url: el.getAttribute('href') },
        children
      )
    default:
      return children
  }
}

根据示例进行改造，详细处理以及使用包可查看与参考官方文档。

额外功能
首行缩进功能
方案描述：添加一个按钮自定义菜单，在点击的时候添加一个新的节点，首行缩进节点，在序列化的时候将节点 序列化为<indentation></indentation>标签。
示例图：

菜单添加参考文档
https://www.wangeditor.com/v5/development.html#buttonmenu
节点添加参考文档
https://www.wangeditor.com/v5/development.html#%E5%AE%9A%E4%B9%89%E6%96%B0%E5%85%83%E7%B4%A0




如何实现一个插件(踩坑实录)
1、理解插件的构成


第一步：数据结构定义 - editorPlugin 
必须需要了解节点的类型以及如何改变当前自定义节点类型
● 节点类型
https://www.wangeditor.com/v5/node-define.html
● 重新定义对应节点的类型、事件、等等
https://www.wangeditor.com/v5/development.html#%E5%AE%9A%E4%B9%89%E8%8A%82%E7%82%B9%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84

第二步：渲染到浏览器富文本中显示的内容 - renderElems
说明:slate结构序列化为html展示在富文本中(使用jsx进行渲染)
参考文档
https://www.wangeditor.com/v5/development.html#%E5%9C%A8%E7%BC%96%E8%BE%91%E5%99%A8%E4%B8%AD%E6%B8%B2%E6%9F%93%E6%96%B0%E5%85%83%E7%B4%A0

第三步：使用editor.getHtml() 获取的文本内容 - elemsToHtml
说明:slate结构序列化为输出为自定义需要的结构(字符串拼接)
简单示例:

data-w-e-type 属性(必须添加)，值为上方节点类型定义的type值，此属性是反序列为slate结构的唯一凭证
其他属性添加自定义值
注意:禁止将json或复杂字符串添加到标签属性中，如果添加的属性为json字符串或复杂结构必须进行编码，否则会出现反序列化失败等异常情况，建议在标签中使用唯一key标记json或复杂数据，如id或其他唯一值，在外部维护对应数据结构即可，保持输出结构单一简洁避免解析异常风险。

第四步：将editor.getHtml() 获取到的内容反序列化成slate结构 - parseElemHtml
https://www.wangeditor.com/v5/development.html#%E8%A7%A3%E6%9E%90%E6%96%B0%E5%85%83%E7%B4%A0-html-%E5%88%B0%E7%BC%96%E8%BE%91%E5%99%A8


第五步：菜单的定义
待补充

特殊内容处理
标签按钮的使用时序
1、标签功能除了富文本结构渲染之外需要暴露钩子操作外部数据
详情使用方法可以参考组件中README

富文本产品沉淀文档：
https://jojoread.yuque.com/rnmm85/rq2ypn/uglcifddpf2f14os
