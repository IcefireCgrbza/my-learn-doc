ControllerAdvice
===
* 增强所有@RequestMapping方法

通过@ExceptionHandler处理异常
---
* 举例
```
@RestControllerAdvice
public class ApiControllerAdvice {

    @ExceptionHandler(BusinessException.class)
    public BaseResponse handleBussinessException(BusinessException businessException) {
        System.out.println("business exception");
        return new BaseResponse().setErrorCode(businessException.getCode()).setErrorMessage(businessException.getMessage());
    }

    @ExceptionHandler(Exception.class)
    public BaseResponse handleException(Exception e) {
        System.out.println("runtime exception");
        return new BaseResponse().setErrorMessage(e.getMessage());
    }

}
```

通过@ModelAttribute传递属性
---
* 感觉比较像mvc的东西

