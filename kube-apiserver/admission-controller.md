## AdmissionController 源码解析

    Kubernetes Version: v1.22@9a1d90165d6466
    Date: 2021.11.20

## 1. 开篇
`Admission Controller`(下面简称 `AC`) 是一段代码，它会在请求通过认证和授权之后、对象被持久化之前拦截到达 API 服务器的请求。准入控制器可以执行 “验证（Validating）” 和/或 “变更（Mutating）” 操作。 变更（mutating）控制器可以修改被其接受的对象。今天我们就从源码入手了解它是如何工作的。

## 2. 启动
`AC` 是随着 `API Server` 一起启动的，所以入口代码位于 `API Server` 启动处，代码位于 `cmd/kube-apiserver/app/server.go`，核心代码位于

    NewAPIServerCommand() ->
    options.NewServerRunOptions() ->
    kubeoptions.NewAdmissionOptions()

我们一起看一下 `NewAdmissionOptions()` 方法：

func NewAdmissionOptions() *AdmissionOptions {
    // 生成配置信息
	options := genericoptions.NewAdmissionOptions()

    // 注册所有的 plugin，Priority Plugin 也在其中
	RegisterAllAdmissionPlugins(options.Plugins)

    // 对 plugins 排序
	options.RecommendedPluginOrder = AllOrderedPlugins

    // 默认关闭 plugin 列表，我们的 Priority Plugin 也在其中，所以需要通过 --enable-admission-plugins 开启
	options.DefaultOffPlugins = DefaultOffAdmissionPlugins()

	return &AdmissionOptions{
		GenericAdmission: options,
	}
}

## 3. 运行