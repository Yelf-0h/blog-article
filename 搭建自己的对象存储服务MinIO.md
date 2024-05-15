# 1、MinIO简介

MinIO是高性能的对象存储，单个对象最大可达5TB。适合存储图片、视频、文档、备份数据、安装包等一系列文件。是一款主要采用Golang语言实现发开的高性能、分布式的对象存储系统。

客户端支持Java,Net,Python,Javacript,Golang语言。客户端与服务器之间采用http/https通信协议。

# 2、MinIO的优点

常见的云存储例如：七牛云，阿里云等。缺点是要钱

私有的存储系统：fastdfs（安装部署超级蛋疼，要安装hadoop那一套，且没有界面...）、mongodb自带的GridFS（在使用上也有诸多不利），所以对照MinIO优点如下：

## 1）开发文档全面

MinIO作为一款基于Golang 编程语言开发的一款高性能的分布式式存储方案的开源项目，有十分完善的官方文档。

## 2）高性能

MinIO号称是目前速度最快的对象存储服务器。在标准硬件上，对象存储的读/写速度最高可以高达183 GB/s和171 GB/s。对象存储可以作为主存储层，用来处理Spark、Presto、TensorFlow、H2O.ai等各种复杂工作负载以及成为Hadoop HDFS的替代品。

MinIO用作云原生应用程序的主要存储，和传统对象存储相比，云原生应用程序需要更高的吞吐量和更低的延迟。而这些都是MinIO能够达成的性能指标。

## 3）SDK支持全面

目前MinIO支持市面主流的开发语言并且可以通过SDK快速集成快速集成使用。
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/4eb85ef62faf768f95bf306e3572c4b0.png)

## 4）安装部署简单

Linux环境下只需下载一个二进制文件然后执行，即可在几分钟内完成安装和配置MinIO。配置选项和变体的数量保持在最低限度，这样让失败的配置概率降低到几乎接近于0的水平。MinIO升级是通过一个简单命令完成的，这个命令可以无中断的完成MinIO的升级工作，并且不需要停机即可完成升级操作，大大降低总使用和运维成本。

## 5） 管理界面的支持

MinIO服务安装后，可以直接通过浏览器登录系统，完成文件夹、文件的管理。非常方便使用。

# 3、安装使用

## 1）docker拉取镜像

```powershell
#拉取镜像
docker pull minio/minio

```

![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/27418a018d6d169d8a71cc091dcc507f.png)

## 2）docker运行

```powershell
#运行容器
docker run -d --restart=always \
-p 9000:9000 \
-p 9001:9001 \
--name minio -v /home/minio/data:/data \
-e "MINIO_ROOT_USER=root" -e "MINIO_ROOT_PASSWORD=15359050865" \
minio/minio server /data --console-address ":9001"

```

![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/28ec4cccfe33ff80fbe9cf9a321635f6.png)

观察docker ps 可以看到运行成功，这时候记得去开放相关端口

## 3）进入客户端管理后台

```html
#通过服务器ip:9001进入管理后台
http://cloud.warframe.top:9001/
```

[管理后台](http://cloud.warframe.top:9001/)

![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/99abae843df6041a8026b108417a7bd0.png)
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/d52009c76d1ba96a4e81651855c20b5b.png)

## 4）小小试玩一下

### 创建Bucket

![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/4debed2025f69712e5ecb09e1aebc659.png)

### 修改成公开权限

![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/ef35a91777f60f377ae5da79c1c0961f.png)

### 然后上传文件

![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/5d2a7a4a8d4eaf7bfc4fb2e2824eb166.png)
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/a6b5163aae85c1c412ff7a50afa0f604.png)

### 访问测试

![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/a0d0f9fb8ecf7ad107cb14a9e1be9e39.png)

### 对比七牛

![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/a1eb2b583898d7821ea64018c72dae3f.png)

## 5) SDK整合-代码实现

```xml

//需要注意的是  minio 依赖于 okhttp 且版本较高。注意，spring-boot-dependencies 中的不够高
<dependency>
      <groupId>io.minio</groupId>
      <artifactId>minio</artifactId>
      <version>8.4.1</version>
</dependency>
<dependency>
      <groupId>com.squareup.okhttp3</groupId>
      <artifactId>okhttp</artifactId>
      <version>4.8.1</version>
</dependency>

```

```java
/**
 * Minio上传策略
 *
 * @author Yelf
 * @date 2024/01/18
 */
@Service("minioUploadStrategyImpl")
public class MinioUploadStrategyImpl extends AbstractUploadStrategyImpl {

    @Autowired
    private MinioProperties minioProperties;

    @Override
    public Boolean exists(String filePath) {
        boolean exist = true;
        try {
            getMinioClient()
                    .statObject(StatObjectArgs.builder().bucket(minioProperties.getBucketName()).object(filePath).build());
        } catch (Exception e) {
            exist = false;
        }
        return exist;
    }

    @SneakyThrows
    @Override
    public void upload(String path, String fileName, InputStream inputStream) {
        getMinioClient().putObject(
                PutObjectArgs.builder().bucket(minioProperties.getBucketName()).object(path + fileName).stream(
                                inputStream, inputStream.available(), -1)
                        .build());
    }

    @Override
    public String getFileAccessUrl(String filePath) {
        return minioProperties.getUrl() + minioProperties.getBucketName() + "/" + filePath;
    }

    private MinioClient getMinioClient() {
        return MinioClient.builder()
                .endpoint(minioProperties.getEndpoint())
                .credentials(minioProperties.getAccessKey(), minioProperties.getSecretKey())
                .build();
    }

}

// 策略调用方法
@Service
public class UploadStrategyContext {

    @Value("${upload.mode}")
    private String uploadMode;

    @Autowired
    private Map<String, UploadStrategy> uploadStrategyMap;

    public String executeUploadStrategy(MultipartFile file, String path) {
        return uploadStrategyMap.get(getStrategy(uploadMode)).uploadFile(file, path);
    }

    public String executeUploadStrategy(String fileName, InputStream inputStream, String path) {
        return uploadStrategyMap.get(getStrategy(uploadMode)).uploadFile(fileName, inputStream, path);
    }

}

//测试方法
@Test
public void name1() throws Exception {
        File file = new File("C:\\Users\\Administrator\\Pictures\\f065408be0151aecbf5a9443e0272407.jpg");
        FileInputStream input = new FileInputStream(file);
        MockMultipartFile multipartFile = new MockMultipartFile("file", file.getName(), "image/jpeg", input);
        String s = uploadStrategyContext.executeUploadStrategy(multipartFile, "cs/");
        System.out.println(s);
    }

```

结果
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/5066f6aebf4fb4cb0966dde6f0e244a3.png)
![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/408c830c4912304969f1d068a8b2f9e9.png)
