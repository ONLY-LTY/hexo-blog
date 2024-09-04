---
title: Spring MVC Exception
date: 2016-11-14 19:04:09
tags:
  - Spring
  - MVC
---
Spring MVC 全局异常处理

&emsp;&emsp;当我们用Sping MVC的时候,可以对Controller层的接口进行全局异常处理。那样就不用每个接口try catch了。只需要继承这个类就好了。

```java
public class BaseExceptionHandler {

    private  static final Log LOGGER = LogFactory.getLog(BaseExceptionHandler.class);

    @ExceptionHandler(NoHandlerFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ResponseEntity handle404Error(HttpServletRequest request, Throwable e){
        LOGGER.error("@NoHandlerFound Path [{}]  {}",request.getRequestURI(),e);
        return new ResponseEntity<>("400",e.getMessage(),null);
    }

    @ExceptionHandler(ServerException.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ResponseEntity handleError(HttpServletRequest request,Throwable e){
        LOGGER.error("@Controller Path [{}]  Method [{}]",request.getRequestURI(),request.getMethod());
        LOGGER.error("@Error - ",e);
        return new ResponseEntity<>("4001","处理错误 | 错误原因 "+e.getMessage(),null);
    }

    @ExceptionHandler(value = {MethodArgumentNotValidException.class, BindException.class})
    ResponseEntity handleMethodArgumentNotValidException(HttpServletRequest request, Exception e) {
        String message = "参数异常,";
        if (e instanceof MethodArgumentNotValidException) {
            MethodArgumentNotValidException exception = (MethodArgumentNotValidException) e;
            message = message + exception.getBindingResult().getFieldError().getDefaultMessage();
        } else if (e instanceof BindException) {
            BindException exception = (BindException) e;
            message = message + exception.getBindingResult().getFieldError().getDefaultMessage();
        }
        LOGGER.error("Controller Check Argument ErrorMsg:[{}] Path:[{}]", message, request.getRequestURI());
        LOGGER.error("Controller Check Argument Error - ", e);
        return new ResponseEntity<>("4001", message, null);
    }

    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    ResponseEntity handleControllerException(HttpServletRequest request, Throwable th) {
        LOGGER.error("Controller Path:[{}]", request.getRequestURI());
        LOGGER.error("Controller Error - ", th);
        return new ResponseEntity<>("500", "操作错误,请稍后再试", null);
    }
}
```

---

<p  align='center'><font color='blue' >没有人不爱惜他的生命，但很少人珍视他的时间。</font></p><p align='right'>——梁实秋</p>

---
