By the end of this chapter, you should be able to:

- Discuss configuration management for applications in Kubernetes using ConfigMaps.
- Share sensitive data (such as passwords) using Secrets.

ConfigMap 允许我们将配置细节与容器镜像解耦。使用 ConfigMap，我们将配置数据以键值对的形式传递，这些数据可以被 Pod 或任何其他系统组件和控制器以环境变量、命令和参数集或卷的形式使用。我们可以通过字面值、配置文件、一个或多个文件或目录创建 ConfigMap。

假设我们有一个 Wordpress 博客应用程序，其中 WordPress 前端使用密码连接到 MySQL 数据库后端。在为 WordPress 创建 Deployment 时，我们可以将 MySQL 密码包含在 Deployment 的 YAML 定义清单中，但密码不会受到保护。任何有权访问定义清单的人都可以获取该密码。

在这种情况下，Secret 对象可以提供帮助，它允许我们在共享敏感信息之前对其进行 Base 64 编码。我们可以像 ConfigMap 一样以键值对的形式共享敏感信息，例如密码、令牌或密钥；因此，我们可以控制 Secret 中信息的使用方式，从而降低意外泄露的风险。在 Deployment 或其他资源中，可以引用 Secret 对象，但不会公开其内容。

需要注意的是，默认情况下，Secret 数据以纯文本形式存储在 etcd 中，因此管理员必须限制对 API 服务器和 etcd 的访问。但是，Secret 数据存储在 etcd 中时可以进行静态加密，但此功能需要由 Kubernetes 集群管理员在 API 服务器级别启用。