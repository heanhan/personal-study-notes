### Flowable 的核心介绍

flowable 全局所有的服务围绕一个 ProcessEngine 为核心的，然后通过 RrocessEngine 去构建三个服务 RepositoryService、RuntimeService、TaskService。

- RepositoryService：这个主要在部署流程的时候会使用到，在项目启动的时候会在此加载配置好的BPMN文档，从而构建整个工作流。
- RuntimeService: 用于启动流程定义的新流程实例,可以通过startProcessInstanceByKey方法来创建一个工作流实例。
- TaskService: 这是和任务相关的，也就是需要人员执行的任务，一个流程是对应这个多个任务节点。