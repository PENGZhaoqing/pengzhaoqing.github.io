---
date: 2015-11-15 
title: Ruby/Java - 两种语言的二叉排序树的实现和遍历
categories: 学习笔记
tags: [ruby,java]
layout: single-blog
---


用Java和Ruby两种语言实现了二叉排序树，纪录一下

## Ruby

```ruby
class BinaryTree
  include Enumerable
 
  def initialize
    @root=TreeNode.new
    @array=[]
  end
  
  def << (data)
    @root.push(data)
  end
  
  def empty?
    if @root.content
      return false
    else 
      return true
    end
  end
  
  class TreeNode
    attr_accessor :content
    attr_accessor :right
    attr_accessor :left
    
    def push(value)
      if self.content==nil
        self.left =TreeNode.new
        self.right =TreeNode.new
        self.content =value 
        print "#{value} is set \n"
      else
        if self.content>=value
          puts "#{value} is compared with #{self.content} \n"
          self.left.push(value)
          
        else
          puts "#{value} is compared with #{self.content} \n"
          self.left.push(value)
         
        end
      end
    end 
  end
  
  
  def tranversal(node)
    if node.content==nil
      return
    else 
      @array<< node.content
      tranversal(node.left)
      tranversal(node.right)
    end
  end
  
  def each
    tranversal(@root)
    @array.each do |value|
      yield value
    end
  end 
end

Tree=BinaryTree.new
Tree<<(5)
Tree<<(4)
Tree<<(9)
Tree<<(8)
Tree<<(1)
Tree<<(2)
puts Tree.empty?

Tree.each do |value| 
    print "#{value},"
end
```

输出结果如下：

```ruby
#=> 5 is set 
#=> 4 is compared with 5 
#=> 4 is set 
#=> 9 is compared with 5 
#=> 9 is compared with 4 
#=> 9 is set 
#=> 8 is compared with 5 
#=> 8 is compared with 4 
#=> 8 is compared with 9 
#=> 8 is set 
#=> 1 is compared with 5 
#=> 1 is compared with 4 
#=> 1 is compared with 9 
#=> 1 is compared with 8 
#=> 1 is set 
#=> 2 is compared with 5 
#=> 2 is compared with 4 
#=> 2 is compared with 9 
#=> 2 is compared with 8 
#=> 2 is compared with 1 
#=> 2 is set 
#=> false
#=> 5,4,9,8,1,2,
```

将BinaryTree继承了Enumerable模块，类似于ruby标准的Array、Hash等collection类；
其中，BinaryTree中定义了内部类TreeNode用来描述数据结构和构造二叉树；
BinaryTree中还定义了其他实例方法：

1. `<<`: 添加新节点
2. `empty?`：判断树是否为空
3. `tranversal`: 对树进行前序遍历
4. `each`：为了继承Enumerable模块而覆写的方法，用来返回遍历的结果

## Java

```java
package SortBinaryTree;

import java.util.ArrayList;

class BinaryTree {

<span style="white-space:pre">	</span>private TreeNode root;
	private ArrayList<Integer> element;

	public BinaryTree() {
		root = new TreeNode();
		element = new ArrayList<Integer>();
	}

	public void add(int i) {
		root.push(i);
	}

	public boolean empty() {
		if (root == null) {
			return true;
		} else {
			return false;
		}
	}

	public void tranversal(TreeNode node) {
		if (node.getKey() == 0) {
			return;
		} else {
			element.add(node.getKey());
			tranversal(node.getLeft());
			tranversal(node.getRight());
		}
	}

	public ArrayList<Integer> getElement() {
		tranversal(root);
		return element;
	}

	class TreeNode {

		private int key;
		private TreeNode left;
		private TreeNode right;

		public int getKey() {
			return key;
		}

		public void setKey(int key) {
			this.key = key;
		}

		public TreeNode getLeft() {
			return left;
		}

		public void setLeft(TreeNode left) {
			this.left = left;
		}

		public TreeNode getRight() {
			return right;
		}

		public void setRight(TreeNode right) {
			this.right = right;
		}

		public void push(int k) {

			if (this.key == 0) {
				setRight(new TreeNode());
				setLeft(new TreeNode());
				setKey(k);
				System.out.println(k + " is set");
			} else {
				if (k >= this.key) {
					System.out.println(k + " is compared with " + this.key);
					this.right.push(k);
				} else {
					System.out.println(k + " is compared with " + this.key);
					this.left.push(k);
				}

			}

		}

	}

}

public class Test {

	public static void main(String args[]) {

		BinaryTree tree = new BinaryTree();
		tree.add(5);
		tree.add(4);
		tree.add(9);
		tree.add(8);
		tree.add(1);
		tree.add(2);

		System.out.println(tree.empty());

		ArrayList<Integer> list = tree.getElement();
		for (int i = 0; i < list.size(); i++) {
			System.out.print(list.get(i) + ",");
		}

	}

}
```

输出结果：

```java
5 is set
4 is compared with 5
4 is set
9 is compared with 5
9 is set
8 is compared with 5
8 is compared with 9
8 is set
1 is compared with 5
1 is compared with 4
1 is set
2 is compared with 5
2 is compared with 4
2 is compared with 1
2 is set
false
5,4,1,2,9,8,
```

思路上与Ruby实现一模一样，在BinaryTree中定义内部类TreeNode描述数据结构和构造树；
同样也定义了add、empty、getElement方法用来增加、判断空树、获得遍历结果。

## <text style="color:red">比较</text>

对比两个语言，可以发现java对数据类型要求比较严格，例如ArrayList定义需声明存储对象类型（泛型），而且Java中基本类型int的判断为空只能用默认值0进行判断，不过可以改申明Integer类来描述整型变量再进行null判断。

Ruby作为脚本语言就灵活多了，有attr_accessor访问器，省去了Java中getter和setter，而且对存储的对象类型没有明确要求，判断是否为空都用nil就行；可以定义each方法能更方便的返回结果。

<text style="color:cornflowerblue">最后，Ruby的self和Java的this用法相似，但self后只能接method，表示当前对象调用这个method；而this后可以接属性或者method，既可以访问和设置属性值还可以调用方法，当然this也表示当前对象。多说一句，self之所以只能调用method是因为Ruby的宗旨是“一切都是方法调用”。</text>

