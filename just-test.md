允许在不重新创建 Pod 或重新启动容器的情况下改变资源，可以直接解决这个问题。此外，**原地 Pod 垂直伸缩**功能依赖于容器运行时接口（CRI）来更新 Pod 容器的 CPU 和/或内存的 **requests/limits**。<br />当前的 CRI API 有一些需要解决的缺点：

- UpdateContainerResources CRI API 需要一个参数来描述要为 Linux 容器更新的容器资源，这在未来可能无法适用于 Windows 容器或其他潜在的非 Linux 运行时。
- 没有 CRI 机制可以让 Kubelet 从容器运行时查询和发现容器上配置的 CPU 和内存限制。
- 处理 UpdateContainerResources CRI API 的运行时的预期行为没有很好地定义或记录。
<a name="DehNE"></a>
### 目标

- 主要：允许更改容器的资源请求（requests）和限制（limits），而不必重新启动容器。
- 次要：允许参与者（用户、VPA、StatefulSet、JobController）决定在无法进行原地资源调整的情况下如何进行。
- 次要：允许用户指定哪些容器可以在不重启的情况下调整资源大小。
