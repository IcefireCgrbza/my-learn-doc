lambda表达式
===
lambda表达式让List操作和匿名内部类变得优雅
---
匿名内部类
---
* (a, b, ...) -> {}
```
    new Thread( () -> System.out.println("In Java8, Lambda expression rocks !!") ).start();
```

List操作
---
* foreach：遍历
    * list.stream.foreach(i -> {});
    ```
        int sum = 0;
        List nums = Arrays.asList(100, 200, 300, 400, 500);
        nums.stream.foreach(i -> {
            sum += i;
        });
    ```
* filter：过滤
    * list.stream.filter(i -> {return xxx ? true : false;});
    * true返回，false过滤
    ```
        List nums = Arrays.asList("aaa", "bbb", "ccc");
        nums.stream.foreach(i -> {
            return Objects.equals(i, "aaa");
        }).collect(Collectors.toLists()).size();  //1
    ```
* map：遍历后返回对象组成新list，与collect一起用效果更佳
    * list.stream.map(i -> {return xxx;});
    * 此处返回后还是stream，需要重组才是List
* collect
    * list.stream.map(i -> {return xxx;}).collect(Collectors.toLists());
    ```
        List nums = Arrays.asList("aaa", "bbb", "ccc");
        nums.stream.foreach(i -> {
            return i.toUpperCase();
        }).collect(Collectors.toLists()).size();  //1
    ```

能不能处理map需要补充，没认真看过
---