### 通过feign实现文件上传

项目为springcloud 服务，项目中对各个业务做了业务能力的拆分，将公共的服务能力放在一个模块里，通过Feign的方式进行调用，feign调用的本质还是http的内部通信，实现模块之间的调用。

通过feign进行文件上传时，feign的示例代码：

```java
@FeignClient
public interface UploadClent{
  
  /**
  * 文件上传
  */
  @PostMapping(value = "/uploadFile", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
  String uploadFile(@RequestPart(value="/updateFile",MultipartFile file,, @RequestParam String uploadPath));
}

```

文件属性使用@RequestPart ，而且在请求中添加consumers= MediaType.MULTIPART_FORM_DATA_VALUE ,

文件表单上传请求通常使用的 ContentType为multipart/from-data,通过以上直接调用feign的方式即可实现feign的文件上传。

以下为 @RequestPart 与 @RequestParam 的区别：

```text
1、@RequestPart 这个注解在multipart/from-data 表单提交请求的方法上。
2、支持的请求方法的方式MultipartFile，属于spring 的MultipartResolver类，这个请求是通过http协议传输的。
3、@RequestParam也同样支持multipart/form-data请求。
4. 他们最大的不同是，当请求方法的请求参数类型不再是String类型的时候，@RequestParam适用于name-valueString类型的请求域，@RequestPart适用于复杂的请求域（像JSON，XML）




@RequestParam批注还可用于将“ multipart/form-data”请求的一部分与支持相同方法参数类型的方法参数相关联。主要区别在于，当方法参数不是String时，@ RequestParam依赖于通过注册的Converter或PropertyEditor进行的类型转换，而@RequestPart依赖于HttpMessageConverters，并考虑了请求部分的“ Content-Type”标头。@RequestParam可能与名称-值表单字段一起使用，而@RequestPart可能与包含更复杂内容（例如JSON，XML）的部分一起使用
```

