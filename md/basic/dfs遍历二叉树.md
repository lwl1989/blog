CreateTime:2021-03-24 09:57:45.0

### dfs


从顶点v出发深度遍历的算法
1. 访问v
2. 依次从顶点v未被访问的邻接点出发深度遍历。

>dfs算法最大特色就在于其递归特性，使得算法代码简洁。但也由于递归使得算法难以理解，原因在于递归使得初学者难以把握程序运行到何处了！一点建议就是先学好递归，把握函数调用是的种种。


### 代码

```
type TreeNode struct {
     Val int
     Left *TreeNode
     Right *TreeNode
}

func dfs(root *TreeNode) {
	if root == nil {
		return false
	}
	fmt.Println(root.Val)
	dfs(root.Left)
	dfs(root.Right)
}
```
