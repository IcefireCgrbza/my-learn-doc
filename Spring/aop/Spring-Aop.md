Spring AOP
===
@Pointcut
---
* 格式：带?的是可选的，*是可选的，..表示任意多个，多个pointcut可以用&&、||隔开
```
execution(modifiers-pattern? ret-type-pattern declaring-type-pattern? name-pattern(param-pattern)throws-pattern?) 
```
* @Pointcut
```
class xxxInterceptor {

	/**
	 * @Pointcut是切入点，切入哪个方法
	 * controller这个方法是这个切入点的signature
	 */
	@Pointcut("execution(* com.feimao.aop.controller.*(..))")
	public void controller(){
	}
	
}
```
@Aspect
---
* spring-aop提供方法级别的切面
* 开启Aspect
```
@EnableAspectJAutoProxy
```
* 注册切面类
```
@Component
@Aspect
```
* 声明，这些方法都是没有返回值的
	+ @After：方法执行后，包括返回和异常
	+ @Before：方法执行之前
	+ @AfterReturning：方法和around返回后
	+ @AfterThrowing：方法和around异常后
	+ @Around：通过JoinPoint控制方法执行
* 顺序
	1. @Around的JoinPoint.proceed()之前
	2. @Before
	3. 被切方法主体
	4. @Around返回前
	5. @After
	6. @AfterReturning或@AfterThrowing
* 举例
```
@Component
@Aspect
public class LogInterceptor {

    @Pointcut("execution(* com.feimao.aoplearning.controller.*.*(..))")
    public void controller(){
    }

    @Before("controller()")
    public void logBeforeExec() {
        System.out.println("before");
    }

    @After("controller()")
    public void logAfterExec() {
        System.out.println("after");
    }

    @AfterReturning("controller()")
    public void logAfterReturning() {
        System.out.println("return");
    }

    @AfterThrowing("controller()")
    public void logAfterThrowing() {
        System.out.println("throw");
    }

    @Around("controller()")
    public Object logAround(ProceedingJoinPoint pjp) throws Throwable {
        System.out.println("around before");
        Object result = pjp.proceed();
        System.out.println("around after");
        return result;
    }

}
```
* @RestControllerAdvice：处理Controller层的异常