---
title: XXL-JOB 发布定时文章
date: 2024-08-22
draft: "false"
tags:
  - XXL-JOB
  - 定时任务
---


![](https://raw.githubusercontent.com/vankykoo/image/main/cut/009.png)

一、提交定时文章，待审核

1. （`ArticleController` ----> `publishScheduled`）接收参数 `ScheduledTask4ArticleTo`，里面包含两个属性。


1. 1. `ScheduledTaskTo`（定时任务信息）：保存发布时间、发布用户id、文章id等信息
   2. `ArticleSubmitDTO`（文章信息）：需要发布的文章具体信息。


1. （`ArticleController` ----> `publishScheduled`）调用 `ArticleService` 的 `submitArticle` 方法，提交文章，也就是存到数据库中，需要人工审核。
2. （`ArticleController` ----> `publishScheduled`）保存文章到数据库后，文章有了id，赋值给定时任务中文章id。调用 `ScheduledTaskFeignClient` 的 saveScheduledTask 方法，保存定时任务信息。

二、审核

1. （ArticleService -----> auditArticle）如果文章审核通过且为定时发布文章，就把定时任务信息放入 RabbitMQ，并把文章状态设置为定时文章待发布状态。

三、发布定时任务

案例中发现：发布定时任务到xxl-job 服务器中注意用到以下几个参数

```java
public class XxlJobInfoBo {

    private int id = 0;				// 0

    private int jobGroup;		// 2
    private String jobDesc;     //任务名

    private String author = "Vanky";		// 负责人

    private String scheduleType = "CRON";			// CRON
    private String scheduleConf;			// cron表达式
    private String misfireStrategy = "DO_NOTHING";

    private String executorRouteStrategy = "FIRST";	// FIRST
    private String executorHandler;		    // 执行器，任务Handler名称
    private String executorBlockStrategy = "SERIAL_EXECUTION";	// SERIAL_EXECUTION

    private String glueType = "BEAN";		// BEAN
    private String glueRemark = "GLUE代码初始化";		// GLUE代码初始化
}
```

需要赋值的有：

1. 1. `jobGroup` 执行任务的组id
   2. `jobDesc` 任务的描述（可不写）
   3. `scheduleConf` 采用cron表达式来表示任务的执行时间
   4. `executorHandler` 执行器，执行任务的具体方法
   5. `appname` 执行任务的服务


1. （`OtherMQService` ----> `taskSubmitArticleQueue`）MQ 的这个方法中，将定时任务信息转为xxl-job服务器需要的信息。比如把时间格式转为`cron`格式，设置任务的执行地址及方法等。
2. （`OtherMQService` ----> `taskSubmitArticleQueue`）给 `xxl-job` 服务器发请求，添加任务并启动（不是马上执行，是到时间再执行）

自己写了一个工具类（向服务器发请求），这个工具类便于向服务器发一些与任务相关的请求。

1. public String add(XxlJobInfo jobInfo,String appName)
2. public String update(int id, String cron)
3. public String remove(int id)
4. public String pause(int id)
5. public String start(int id)
6. public String addAndStart(XxlJobInfo jobInfo,String appName)

其中`addAndStart` 方法用于添加任务并启动，这个方法先根据`appname` 获取组id（必须要的参数），然后再添加任务，并启动。

四、执行定时任务

1. 到了执行任务的时间，xxl-job服务器到指定的服务的方法中执行，修改文章发布时间和状态。

```java
@XxlJob("submitArticleScheduled")
public void submitArticleScheduled(){
    // 执行发布逻辑
}
```



附：

还需要配合地 在xxl-job 服务端 写一些接收这些请求的简单逻辑

```java
@Slf4j
@Component
public class XxlJobUtil {
 
    @Value("${xxl.job.admin.addresses}")
    private String adminAddresses;
 
    @Value("${xxl.job.executor.appname}")
    private String appname;
 
    private RestTemplate restTemplate = new RestTemplate();
 
    private static final String ADD_URL = "/jobinfo/addJob";
    private static final String UPDATE_URL = "/jobinfo/updateJob";
    private static final String REMOVE_URL = "/jobinfo/removeJob";
    private static final String PAUSE_URL = "/jobinfo/pauseJob";
    private static final String START_URL = "/jobinfo/startJob";
    private static final String ADD_START_URL = "/jobinfo/addAndStart";
    private static final String GET_GROUP_ID = "/jobgroup/getGroupId";
 
 
    public String add(XxlJobInfo jobInfo,String appName){
        // 查询对应groupId:
        Map<String,Object> param = new HashMap<>();
        param.put("appname", appName);
        String json = JSON.toJSONString(param);
        String result = doPost(adminAddresses + GET_GROUP_ID, json);

        JSONObject jsonObject = JSON.parseObject(result);
        String groupId = jsonObject.getString("content");
        jobInfo.setJobGroup(Integer.parseInt(groupId));
        String json2 = JSON.toJSONString(jobInfo);
        return doPost(adminAddresses + ADD_URL, json2);
    }
 
    public String update(int id, String cron){
        Map<String,Object> param = new HashMap<>();
        param.put("id", id);
        param.put("jobCron", cron);
        String json = JSON.toJSONString(param);
        return doPost(adminAddresses + UPDATE_URL, json);
    }
 
    public String remove(int id){
        Map<String,Object> param = new HashMap<>();
        param.put("id", id);
        String json = JSON.toJSONString(param);
        return doPost(adminAddresses + REMOVE_URL, json);
    }
 
    public String pause(int id){
        Map<String,Object> param = new HashMap<>();
        param.put("id", id);
        String json = JSON.toJSONString(param);
        return doPost(adminAddresses + PAUSE_URL, json);
    }
 
    public String start(int id){
        Map<String,Object> param = new HashMap<>();
        param.put("id", id);
        String json = JSON.toJSONString(param);
        return doPost(adminAddresses + START_URL, json);
    }
 
    public String addAndStart(XxlJobInfo jobInfo,String appName){
        Map<String,Object> param = new HashMap<>();
        param.put("appname", appName);
        String json = JSON.toJSONString(param);
        String result = doPost(adminAddresses + GET_GROUP_ID, json);
 
        JSONObject jsonObject = JSON.parseObject(result);
        String groupId = jsonObject.getString("content");
        jobInfo.setJobGroup(Integer.parseInt(groupId));
        String json2 = JSON.toJSONString(jobInfo);
 
        return doPost(adminAddresses + ADD_START_URL, json2);
    }

    /**
     * 用于发送post请求
     * @param url
     * @param json
     * @return
     */
    public String doPost(String url, String json){
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        HttpEntity<String> entity = new HttpEntity<>(json ,headers);
        log.info(entity.toString());
        ResponseEntity<String> stringResponseEntity = restTemplate.postForEntity(url, entity, String.class);
        return stringResponseEntity.getBody().toString();
    }
 
}
```

