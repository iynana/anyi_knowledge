+ String.contains本质是字符串暴力匹配
+ List.contains本质是equals顺序遍历
+ HashSet.contains本质是hashCode + equals

注意点：
+ 区分大小写
+ 需要要定义hashcode
+ 性能问题