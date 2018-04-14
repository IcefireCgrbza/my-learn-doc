Lombok
===
依赖
---
* compile "org.projectlombok:lombok:$lombokVersion"
* idea支持
    * 装lombok插件
    * setting开anno

提供给实体类优雅的set,get,toString,constructor等方法
---
重要注解如下：
* @Data：为所有属性设置set和get方法
* @ToString：toString方法
* @NoArgsConstructor
* @AllArgsConstructor
* @Accessor：chain属性为true时，提供链式set方法
* @Slf4j：声明log日志对象

举例
---
```
@Data
@NoArgsConstructor
@Accessors(chain = true)
public class User {

    private Long id;

    private String userName;

    private Boolean isAdmin;

}
```
