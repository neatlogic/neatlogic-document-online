# API管理

API管理用于查看和维护系统内部定义的接口，支持导出接口信息、设置访问频率、启用审计、查看帮助和接口测试。

当需要对外提供接口调用、排查接口访问异常，或确认接口入参和认证规则时，可以从 API管理 开始。

## 阅读路径

可根据当前要解决的问题选择阅读入口。

- 如果需要查看或导出接口信息，阅读“导出”。
- 如果需要限制接口访问频率，阅读“接口访问频率”。
- 如果需要记录接口调用日志，阅读“启用审计”。
- 如果需要查看接口说明，阅读“查看帮助”。
- 如果第三方系统需要调用接口，阅读“接口认证”。
- 如果需要模拟接口调用，阅读“测试”。

## 权限

**操作权限**
- 配置：系统配置-[用户管理](../1.用户和权限/用户管理.md)-授权-接口管理权限
- 包含操作：查看接口、导出接口、设置访问频率、启用审计、查看帮助和测试接口
- 配置人员：系统超级管理员

## 导出

导出用于将系统内部接口详情导出为 PDF 文档。导出内容包括接口 URL、名称、描述和输入参数等信息。

![](images/接口管理_导出.png)

![](images/接口管理_接口信息.png)

## 接口访问频率

接口访问频率用于限制接口每秒可访问次数，适合控制高频调用对系统造成的压力。

![](images/接口管理_访问频率.png)

## 启用审计

启用审计后，系统会记录接口访问日志。访问记录可在调用记录中查看，也可在[操作审计](操作审计.md)页面查看详情。

![](images/接口管理_启用审计.png)

API管理支持设置访问记录保留期限。超过保留期限的访问记录会被系统自动删除；未设置期限时，系统会保留所有访问记录。

![](images/接口管理_访问记录保留期限.png)

## 查看帮助

查看帮助用于查看接口的说明信息，便于确认接口用途、请求参数和返回内容。

![](images/接口管理_帮助.png)

## 接口认证

第三方系统调用 NeatLogic 接口时，需要按规则组织请求数据，并使用用户 Token 生成 HMAC 签名。

### 1. 获取 Token

每个用户都有一个 Token。Token 仅用户本人可见，用户可以随时刷新 Token。第三方系统使用该用户身份调用接口时，也需要同步更新 Token。

![](images/接口管理_获取token.png)

### 2. 设置 Header

访问 NeatLogic 系统接口时，请求需要额外携带 `Tenant`、`AuthType`、`Authorization` 和 `x-access-key` Header。
    
  <table style="width:100%">
    <thead>
      <tr>
        <td>Header</td>
        <td>描述</td>
        <td>例子</td>
      </tr>
    </thead>
    <tbody>
      <tr>
        <td>Tenant</td>
        <td>租户</td>
        <td>demo</td>
      </tr>
      <tr>
        <td>Authorization</td>
        <td>认证信息</td>
        <td>Hmac<br>87d83f9d39c87d35f6925f712f7f28660e1f0637</td>
      </tr>
      <tr>
        <td>AuthType</td>
        <td>认证类型</td>
        <td>hmac</td>
      </tr>
      <tr>
        <td>x-access-key</td>
        <td>用户</td>
        <td>admin</td>
      </tr>
    </tbody>
  </table>

#### Authorization 生成规则

##### 1. 生成 sign

```text
sign = x-access-key + # + requestUri + "?" + queryString + # + base64(post body)
```

GET 请求没有 `post body` 时，也需要拼接 `#`。如果没有 `queryString`，不需要提供 `?`。

##### 2. 生成 authorization

```text
authorization = "Hmac " + HmacSHA256签名加密(token, sign)
```

范例：

请求地址：

```text
/neatlogic/api/rest/inspect/report/get
```

post body:

```json
{"test":"ddddd"}
```

拼接签名数据：

```text
sign=admin#/neatlogic/api/rest/inspect/report/get#ewogICAgInRlc3QiOiJkZGRkZCIKfQ==
```

使用 SHA256 算法进行签名：

```text
authorization="Hmac " + HmacSHA256签名加密(用户token, sign)
```

```java
InputStream input = request.getInputStream();
StringBuilder sb = new StringBuilder();
BufferedReader reader;
if (input != null) {
    reader = new BufferedReader(new InputStreamReader(input));
    char[] charBuffer = new char[2048];
    int bytesRead = -1;
    while ((bytesRead = reader.read(charBuffer)) > 0) {
        sb.append(charBuffer, 0, bytesRead);
    }
}

String queryString = StringUtils.isNotBlank(request.getQueryString()) ? "?" + request.getQueryString() : StringUtils.EMPTY;
String sign = user + "#" + request.getRequestURI() + queryString + "#" + Base64.encodeBase64StringUnChunked(sb.toString().getBytes(StandardCharsets.UTF_8));
String authorization = SHA256Util.encrypt(token, sign);
```
签名程序：
```java
    public static String encrypt(String secret, String sign) {
        try {
            SecretKeySpec signingKey = new SecretKeySpec(secret.getBytes(), "HmacSHA256");
            Mac mac = Mac.getInstance("HmacSHA256");
            mac.init(signingKey);
            byte[] rawHmac = mac.doFinal(sign.getBytes());
            StringBuilder hexString = new StringBuilder();
            for (byte b : rawHmac) {
                String shaHex = Integer.toHexString(b & 0xFF);
                if (shaHex.length() < 2) {
                    hexString.append(0);
                }
                hexString.append(shaHex);
            }
            return hexString.toString();
        } catch (NoSuchAlgorithmException | InvalidKeyException e) {
            logger.error(e.getMessage(), e);
        }
        return "0000000000000000000000000000000000000000000000000000000000000000";
    }
```
完整调用范例：
```java
package neatlogic.framework.apiparam.validator;

import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;
import okhttp3.*;
import org.springframework.http.MediaType;
import org.springframework.http.*;
import org.springframework.util.Base64Utils;
import org.springframework.web.client.RestTemplate;

import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;
import java.io.File;
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.Map;

public class HmacDemo {

    public static String encrypt(String secret, String sign) {
        try {
            SecretKeySpec signingKey = new SecretKeySpec(secret.getBytes(), "HmacSHA256");
            Mac mac = Mac.getInstance("HmacSHA256");
            mac.init(signingKey);
            byte[] rawHmac = mac.doFinal(sign.getBytes());
            StringBuilder hexString = new StringBuilder();
            for (byte b : rawHmac) {
                String shaHex = Integer.toHexString(b & 0xFF);
                if (shaHex.length() < 2) {
                    hexString.append(0);
                }
                hexString.append(shaHex);
            }
            return hexString.toString();
        } catch (Exception e) {
            e.printStackTrace();
        }
        return "0000000000000000000000000000000000000000000000000000000000000000";
    }

    public static void main(String[] args) throws IOException {
        testRestApi();


    }

    /**
     * 上传 binary
     */
    private static void testUploadBinaryApi() {
        JSONObject postDataObj = new JSONObject();
//        postDataObj.put("name", "name");
//        postDataObj.put("label", "name1");
//        postDataObj.put("icon", "name1");
//        postDataObj.put("typeId", "441079140720640");
        System.out.println(postDataObj);//打印请求body
        String cutUrl = "/neatlogic/api/binary/autoexec/script/import/forautoexec";
        String url = "http://localhost:8080" + cutUrl;
        String tenant = "develop";//租户
        String userId = "admin";//登录UserId
        String token = "eb40625d3464db3b4ddf361969524bf7";//用户token

        OkHttpClient client = new OkHttpClient();
        File file = new File("/Users/cocokong/Desktop/scripts_Demo_demo-local.pl.json");
        okhttp3.MediaType mediaType = okhttp3.MediaType.parse("multipart/form-data;");
        MultipartBody.Builder builder = new MultipartBody.Builder().setType(MultipartBody.FORM)
                .addFormDataPart("scriptInfo.json", "scripts_Demo_demo-local.pl",
                        RequestBody.create(mediaType,
                                file));
        for(Map.Entry<String,Object> entry : postDataObj.entrySet()){
            builder.addFormDataPart(entry.getKey(), entry.getValue().toString());
        }
        RequestBody body = builder.build();

        String postDataBase64 = Base64Utils.encodeToString(postDataObj.toString().getBytes());
        String sign = userId + "#" + cutUrl + "#" + postDataBase64;
        System.out.println("sign:" + sign);
        String authorization = encrypt(token, sign);//认证
        System.out.println("authorization:" + authorization);

        Request request = new Request.Builder()
                .url(url)
                .post(body)
                .addHeader("Tenant", tenant)
                .addHeader("Authorization", "Hmac " + authorization)
                .addHeader("AuthType", "hmac")
                .addHeader("x-access-key", userId)
                .addHeader("Content-Type", "multipart/form-data")
                .build();

        try {
            Response response = client.newCall(request).execute();
            System.out.println(response.body().string());
        } catch (IOException e) {
            e.printStackTrace();
        }
    }


    /**
     * 下载 binary
     */
    private static void testDownloadBinaryApi() throws IOException {
        //请求url
        String cutUrl = "/neatlogic/api/binary/file/download";
        String url = "http://localhost:8080" + cutUrl;
        String tenant = "develop";//租户
        String userId = "admin";//登录UserId
        String token = "eb40625d3464db3b4ddf361969524bf7";//用户token 从”页面右上角头像-》个人设置-〉基础信息“可以获取
        JSONObject postDataObj = JSONObject.parseObject("{\"id\":1084815183822848}");//请求post data
        System.out.println(postDataObj.toString());//打印请求body

        //将postdata加密 进而获取authorization
        String postDataBase64 = Base64Utils.encodeToString(JSON.toJSONString(postDataObj, false).getBytes());
        String sign = userId + "#" + cutUrl + "#" + postDataBase64;
        System.out.println("sign:" + sign);
        String authorization = encrypt(token, sign);//认证
        System.out.println("authorization:" + authorization);

        //设置请求头
        HttpHeaders requestHeaders = new HttpHeaders();
        requestHeaders.setContentType(MediaType.APPLICATION_JSON);
        requestHeaders.add("Tenant", tenant);
        requestHeaders.add("Authorization", "Hmac " + authorization);
        requestHeaders.add("AuthType", "hmac");
        requestHeaders.add("x-access-key", userId);
        HttpEntity<JSONObject> requestEntity = new HttpEntity<>(postDataObj, requestHeaders);
        RestTemplate restTemplate = new RestTemplate();

        //调接口
        ResponseEntity<byte[]> response = restTemplate.exchange(url, HttpMethod.POST, requestEntity, byte[].class);
        byte[] fileContent = response.getBody();
        Path destination = Paths.get("/Users/cocokong/Desktop/test.png");
        Files.write(destination, fileContent); // 将文件内容写入到目标路径中
    }

    /**
     * 普通接口
     */
    private static void testRestApi() {
        String cutUrl = "/neatlogic/api/rest/cmdb/cientity/search";
        String url = "http://localhost:8080" + cutUrl;
        String tenant = "develop";//租户
        String userId = "admin";//登录UserId
        String token = "eb40625d3464db3b4ddf361969524bf7";//用户token 从”页面右上角头像-》个人设置-〉基础信息“可以获取
        JSONObject postDataObj = JSONObject.parseObject("{\"ciId\":479609502048256,\"keyword\":\"Test\"}");//请求post data
        System.out.println(postDataObj.toString());//打印请求body

        //将postdata加密 进而获取authorization
        String postDataBase64 = Base64Utils.encodeToString(JSON.toJSONString(postDataObj, false).getBytes());
        String sign = userId + "#" + cutUrl + "#" + postDataBase64;
        System.out.println("sign:" + sign);
        String authorization = encrypt(token, sign);//认证
        System.out.println("authorization:" + authorization);

        //设置请求头
        HttpHeaders requestHeaders = new HttpHeaders();
        requestHeaders.setContentType(MediaType.APPLICATION_JSON);
        requestHeaders.add("Tenant", tenant);
        requestHeaders.add("Authorization", "Hmac " + authorization);
        requestHeaders.add("AuthType", "hmac");
        requestHeaders.add("x-access-key", userId);
        HttpEntity<JSONObject> requestEntity = new HttpEntity<>(postDataObj, requestHeaders);
        RestTemplate restTemplate = new RestTemplate();

        //调接口
        JSONObject resultObj = restTemplate.postForObject(url, requestEntity, JSONObject.class);
        System.out.println(resultObj);
    }
}

```
## 测试

测试用于模拟发送请求，检查接口调用情况。

![](images/接口管理_测试.png)
