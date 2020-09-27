HashMap与ConcurrentHashMap——jdk8
===
以下所有内容并非与jdk源码完全相同，属于个人对jdk的理解

HashMap
---
* 内部数据结构
```
List<Map.Entry<K, V>>	//存储结构

/**
  * Map.Entry里的东西
  */	
class Node<K, V> implement Map.Entry<K, V> {

	final private int hash;
	
	final private K key;
	
	private V value;
	
	private Node<K, V> next;
	
}

/**
 * 链表超过8位长度转为红黑树
 */
class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
        
        TreeNode<K,V> parent;  // red-black tree links
        
        TreeNode<K,V> left;
        
        TreeNode<K,V> right;
        
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
        
        boolean red;
        
 }
```
* 初始化容量capacity默认16、负载因子loadFactor默认0.75、扩容时机是当前容量超过总容量乘以负载因子、容量总是2^n
* 计算hash
```
//位运算
h = hash & (cap);
```
* put过程：算hash找到位置，null就new Node，next=null就next=new Node，超过8位转TreeNode。全方位不支持多线程操作
* get：算hash找到位置，遍历Node或者TreeNode，红黑树复杂度O(logn)
* resize：容量size超过阈值threshold（总容量capacity乘以负载因子loadFactor）就扩容，遍历一遍一个个放
* 可以看出到处都是多线程不安全的问题

ConcurrentHashMap
---
* jdk7实现方式：默认16个分段锁加HashMap，put和resize需要加锁，gei无锁，保证不读到脏数据即可
* 数据结构
```
```
* jdk8数据结构：同HashMap
