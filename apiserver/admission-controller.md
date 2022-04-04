## AdmissionController 源码解析 (InProgress)

    Kubernetes Version: v1.22@9a1d90165d6466
    Date: 2021.11.20

## 1. 开篇
`Admission Controller`(下面简称 `AC`) 是一段代码，它会在请求通过认证和授权之后、对象被持久化之前拦截到达 API 服务器的请求。准入控制器可以执行 “验证（Validating）” 和/或 “变更（Mutating）” 操作。 变更（mutating）控制器可以修改被其接受的对象。今天我们就从源码入手了解它是如何工作的。

## 2. 启动
`AC` 是随着 `API Server` 一起启动的，入口位于 `cmd/kube-apiserver/app/server.go`，调用链如下：

    NewAPIServerCommand() ->
    options.NewServerRunOptions() ->
    kubeoptions.NewAdmissionOptions()

我们一起看一下 `NewAdmissionOptions()` 方法：

func NewAdmissionOptions() *AdmissionOptions {
    // 生成配置信息
	options := genericoptions.NewAdmissionOptions()

    // 注册所有的 plugin
	RegisterAllAdmissionPlugins(options.Plugins)

    // 对 plugins 排序
	options.RecommendedPluginOrder = AllOrderedPlugins

    // 设置默认关闭 plugin 列表
	options.DefaultOffPlugins = DefaultOffAdmissionPlugins()

	return &AdmissionOptions{
		GenericAdmission: options,
	}
}

## 3. 运行
运行的入口命令同样位于 `cmd/kube-apiserver/app/server.go` 中，调用链如下：

    NewAPIServerCommand() -> completedOptions.Validate()

我们一起看一下 `Validata()` 方法：

    func (s *ServerRunOptions) Validate() []error {
        var errs []error
        if s.MasterCount <= 0 {
            errs = append(errs, fmt.Errorf("--apiserver-count should be a positive number, but value '%d' provided", s.MasterCount))
        }
        errs = append(errs, s.Etcd.Validate()...)
        errs = append(errs, validateClusterIPFlags(s)...)
        errs = append(errs, validateServiceNodePort(s)...)
        errs = append(errs, validateAPIPriorityAndFairness(s)...)
        errs = append(errs, s.SecureServing.Validate()...)
        errs = append(errs, s.Authentication.Validate()...)
        errs = append(errs, s.Authorization.Validate()...)
        errs = append(errs, s.Audit.Validate()...)
        errs = append(errs, s.Admission.Validate()...)
        errs = append(errs, s.APIEnablement.Validate(legacyscheme.Scheme, apiextensionsapiserver.Scheme, aggregatorscheme.Scheme)...)
        errs = append(errs, validateTokenRequest(s)...)
        errs = append(errs, s.Metrics.Validate()...)
        errs = append(errs, validateAPIServerIdentity(s)...)

        return errs
    }

这里包含了所有的参数验证相关逻辑，比如 `AuthZ`，`AuthN`，`Admission`，以 `Admission` 为例，我们一起看一下 `s.Admission.Validate()` 方法：

    func (a *AdmissionOptions) Validate() []error {

        var errs []error

        // admission-control 已经 deprecated，plugin 只能在注册到一个地方，他们是互斥的
        if a.PluginNames != nil &&
            (a.GenericAdmission.EnablePlugins != nil || a.GenericAdmission.DisablePlugins != nil) {
            errs = append(errs, fmt.Errorf("admission-control and enable-admission-plugins/disable-admission-plugins flags are mutually exclusive"))
        }

        // registeredPlugins 是所有注册的 plugin，这里进行存在性判断
        registeredPlugins := sets.NewString(a.GenericAdmission.Plugins.Registered()...)
        for _, name := range a.PluginNames {
            if !registeredPlugins.Has(name) {
                errs = append(errs, fmt.Errorf("admission-control plugin %q is unknown", name))
            }
        }

        // 对所有的 Plugin 是否存在进行验证，比较简单，就不展开了
        errs = append(errs, a.GenericAdmission.Validate()...)

        return errs
    }
