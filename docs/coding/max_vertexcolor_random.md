---
layout: default
---

```python
//修改顶点色有meshOp和polyOp两种操作，区别是meshOp如果进入到mesh编辑将会丢失模型的modifiers（修改器）
//为了保留诸如蒙皮等信息，选用polyOp
function VertexColorRandom pMin pMax =
(
	undo on
	--"--"为注释符号
	--切换到modify模式（模型修改模式）
	if (getCommandPanelTaskMode() != #modify) do
	setCommandPanelTaskMode #modify
	--获得模型选择队列（selection获取的是动态选择队列，不符合这里）
	sl = getcurrentSelection()
	for i in sl do
	(
		--选择队列中模型
		select i
		--选择到基础模型（绕开modifiers)
		modPanel.setCurrentObject i.baseObject
		i.vertexColorType = #alpha --alpha 类型
		i.showVertexColors = true	--顶点色显示
		mesh_verts = i.numverts		--当前模型顶点数
		for v=1 to mesh_vert-1 do
		(
			val = random pMin pMax		--随机数这里是0-255
			c = color val val val
			polyop.setVertColor i -2 #(v) c		--i=设置颜色物体；-2=alpha；#(start,end)；c=color
		)
		modPanel.setCurrentObject i.modifiers[1]		--选回最上层的modifiers
	)
	select sl		--遍历完选回之前选择的模型队列
)

```



[back](../../coding-page.html)