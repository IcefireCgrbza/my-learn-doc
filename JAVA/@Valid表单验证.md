@Valid表单验证
===
对请求字段做验证，拒绝在业务逻辑头部写大量if else判断
===
前后端对接思考
---
null还是空对象？
* 对String来说，前端传String，在输入框输入后删除掉时一般是空对象""，用户不输入，前端初始化为null才是传null,还有可能用户输入一堆空格就提交了，就是说都有可能
* 对对象来说，可能是null，可能是对象内部都是null，可能对象内部部分是null
* 对数组来说，可能是null，也可能长度为0
* 自己写前端的时候，最好是初始化为空对象！！！，String初始化为""！！！！List为[]！！！避免在前端出现奇奇怪怪的问题

@Valid：后端统一表单验证方式
---
* 需要Spring支持
* @Valid：验证某对象
* @NotNull
* @NotBlank
* @NotEmpty
```
@Data
@Accessors(chain = true)
@NoArgsConstructor
@ToString
public class CdnConfigCreateRequest extends BaseRequest {

    @NotBlank
    private String bucketTag;

    @NotNull
    private Boolean useOss;

    @NotNull
    @Valid
    private OssBucketConfigCreateRequest oss;

    @NotEmpty
    @Valid
    private List<CosBucketConfigCreateRequest> cos;

    private Double rateLimit;

    @Data
    @Accessors(chain = true)
    @NoArgsConstructor
    @ToString
    public static class OssBucketConfigCreateRequest{
        @NotBlank
        private String uploadEndpoint;

        @NotBlank
        private String downloadEndpoint;

        @NotBlank
        private String bucketName;

        @NotBlank
        private String uploadPath;

        @NotBlank
        private String accessKeyId;

        @NotBlank
        private String accessKeySecret;

        @NotNull
        private Integer maxSize;

        private Integer uploadExpire;

        private Long maxPartSize;
    }

    @Data
    @Accessors(chain = true)
    @NoArgsConstructor
    @ToString
    public static class CosBucketConfigCreateRequest{
        @NotBlank
        private String uploadEndpoint;

        @NotBlank
        private String downloadEndpoint;

        @NotBlank
        private String bucketName;

        @NotBlank
        private String uploadPath;

        @NotBlank
        private String secretId;

        @NotBlank
        private String secretKey;

        @NotNull
        private Integer maxSize;

        @NotBlank
        private String appId;
    }

}
```
```
@RequestMapping(value = "/create", method = RequestMethod.POST)
    @ResponseBody
    public Object cdnConfigCreate(@RequestBody @Valid CdnConfigCreateRequest request){
        log.info("bucket create request: {}", request);
        BaseResponse response = commandBus.dispatch(cdnConfigCreateCommand, request);
        log.info("bucket create response: {}", JsonUtils.toJson(response));
        return response;
    }
```

通过AOP给@Valid拦截的请求打log
---
（暂未研究）
