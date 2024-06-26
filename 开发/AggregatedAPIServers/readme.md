参考文章

1. [kubernetes-sigs/apiserver-builder-alpha](https://github.com/kubernetes-sigs/apiserver-builder-alpha)
2. [【Sigma敏捷版系列文章】如何利用apiserver-builder自定义Kubernetes API](https://developer.aliyun.com/article/610357)
3. [Kubernetes Apiserver和Extension apiserver的介绍](https://www.yisu.com/zixun/9840.html)
4. [Configure the Aggregation Layer](https://kubernetes.io/docs/tasks/extend-kubernetes/configure-aggregation-layer/)
5. [设置一个扩展的 API server](https://v1-17.docs.kubernetes.io/zh/docs/tasks/access-kubernetes-api/setup-extension-api-server/)
6. [Kubernetes API Aggregation Layer](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/)
    - 官方文档

> Unless you absolutely need apiserver-aggregation, you are recommended to use [Kubebuilder](https://github.com/kubernetes-sigs/kubebuilder) instead of apiserver-builder for building Kubernetes APIs. Kubebuilder builds APIs using CRDs and addresses limitations and feedback from apiserver-builder. --参加文章1

除非你非常清楚自己需要`aggregation`的功能, 否则一般情况下都建议使用`kubebuilder`来提供 kube 的 api(`kubebuilder`就是用来创建`CRD`的工具库). 可以说, 使用`apiserver-builder`创建的服务比`kubebuilder`的服务更底层.

