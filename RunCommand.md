    Kubernetes Version: v1.22
    Author(Github): @kerthcet
    Datetime: 2021.10.17

# Kube-Scheduler初始化运行

## 一. 开篇
Kubernetes Scheduling是Kubernetes架构设计中的核心模块，负责整个集群的容器编排和调度策略。得益于Kubernetes整体架构设计的一致性，我们知道所有的组件启动命令都位于 `cmd` 文件夹下各个子模块的 `main` 方法内，`scheduling` 模块的具体代码位置位于 `cmd/kube-scheduler/scheduler.go` ，代码如下：

    func main() {
        command := app.NewSchedulerCommand()
        code := cli.Run(command)
        os.Exit(code)
    }


代码很简单，分为3步，第一步注册 Command，第二步运行 Command， 第三步退出程序，我们一步一步看。

## 二. 注册 Command
`app.NewSchedulerCommand()` 代码位于 `cmd/kube-scheduler/app/server.go`, 我把主要的逻辑梳理下，省略的代码用 `...` 标记。

    func NewSchedulerCommand(registryOptions ...Option) *cobra.Command {
     	opts, err := options.NewOptions()
        // ...

        namedFlagSets := opts.Flags()

        cmd := &cobra.Command{
            // ...

            RunE: func(cmd *cobra.Command, args []string) error {
                if err := runCommand(cmd, opts, registryOptions...); err != nil {
                    return err
                }
                return nil
            },

            // ...
        }

        // ...
        return cmd
    }


1. `options.NewOptions()` 主要负责补全默认的启动参数，如默认端口10259，默认 log 配置， 类似于工厂函数。

2. `namedFlagSets := opts.Flags()` 主要负责解析命令行参数，这里用到了 `spf13/cobra` 库。举个例子：

        fs.StringVar(&o.ConfigFile, "config", o.ConfigFile, "The path to the configuration file.")`

    `fs` 是 `Struct FlagSet` 实例，定义所有 `flags` 的集合。该行代码第一个参数接受命令行参数变量，第二个参数指定命令行参数名称，第三个表示默认值，第四个参数设置命令行参数提示信息。最后将该 `flag` 加入到 `fs` 集合中。

3. 定义 `cobra.Command` 对象，其中最重要的是 `RunE` 方法的定义，这是实际执行的方法。我们看一下 `Command` 数据结构，他有很多字段，我们挑几个常用的看一下，代码位于 `vendor/github.com/spf13/cobra/command.go`:

        type Command struct {
            PersistentPreRun func(cmd *Command, args []string)
            PersistentPreRunE func(cmd *Command, args []string) error
            PreRun func(cmd *Command, args []string)
            PreRunE func(cmd *Command, args []string) error
            Run func(cmd *Command, args []string)
            RunE func(cmd *Command, args []string) error
            PostRun func(cmd *Command, args []string)
            PostRunE func(cmd *Command, args []string) error
            PersistentPostRun func(cmd *Command, args []string)
            PersistentPostRunE func(cmd *Command, args []string) error


            // SilenceErrors is an option to quiet errors down stream.
	        SilenceErrors bool
            // SilenceUsage is an option to silence usage when an error occurs.
            SilenceUsage bool
        }

    在数据结构中，我们看到很多 `Run` 方法和 `RunE` 方法，他们的区别在于是否返回 `error`，而这些 `Run` 方法有一个调用顺序的区别，`PersistentPreRun -> PreRun -> Run -> PostRun ->PersistentPostRun`, `RunE` 方法也是一样的顺序。

    `SilenceErrors` 和 `SilenceUsage` 用来静默是否由 `cobra` 输出错误和使用信息，默认情况下，如果解析 `command` 出现错误，会输出错误和使用信息，代码同样位于 `vendor/github.com/spf13/cobra/command.go`：

            err = cmd.execute(flags)
            if err != nil {
                // Always show help if requested, even if SilenceErrors is in
                // effect
                if err == flag.ErrHelp {
                    cmd.HelpFunc()(cmd, args)
                    return cmd, nil
                }

                // If root command has SilenceErrors flagged,
                // all subcommands should respect it
                if !cmd.SilenceErrors && !c.SilenceErrors {
                    c.PrintErrln("Error:", err.Error())
                }

                // If root command has SilenceUsage flagged,
                // all subcommands should respect it
                if !cmd.SilenceUsage && !c.SilenceUsage {
                    c.Println(cmd.UsageString())
                }
            }

## 三. 运行 Command
`cli.Run(command)` 方法会执行 `command` 命令，其实主要是执行 上面定义的 `RunE` 方法，我们看一下:

        RunE: func(cmd *cobra.Command, args []string) error {
			if err := opts.Complete(&namedFlagSets); err != nil {
				return err
			}
			if err := runCommand(cmd, opts, registryOptions...); err != nil {
				return err
			}
			return nil
		}


1. `opts.Complete()` 用于补全初始化所需的配置，并作为参数传入 `runCommand` 方法
2. `runCommand` 包含了真正运行 `kube-scheduler` 的代码:

        func runCommand(cmd *cobra.Command, opts *options.Options, registryOptions ...Option) error {
            ...

            ctx, cancel := context.WithCancel(context.Background())
            defer cancel()
            go func() {
                stopCh := server.SetupSignalHandler()
                <-stopCh
                cancel()
            }()

            cc, sched, err := Setup(ctx, opts, registryOptions...)
            if err != nil {
                return err
            }

            return Run(ctx, cc, sched)
        }

    `Setup()` 方法主要用于初始化 `Scheduler` 实例，`Run()` 则根据配置运行 `scheduler`，具体逻辑会在后面的系列文章中说明，不作为本次文章重点。
## 四. 退出程序
`Run` 命令会根据执行情况返回对应的错误码，成功返回0，错误返回非0。调用 `os.Exit(code)` 退出程序。

## 五. 其他
### 1. 如何自定义 `flag` 启动 `kube-scheduler`
假设我们使用 `kubeadm` 来安装 `kubernetes cluster`，登陆到 `master` 节点，我们可以在 `/etc/kubernetes/manifests` 下发现几个 `yaml` 文件。

    etcd.yaml
    kube-apiserver.yaml
    kube-controller-manager.yaml
    kube-scheduler.yaml

这几个组件都是以 `static pod` 的方式运行，所谓 static pod ，他们都是由 kubelet 守护进程直接管理，不需要 API 服务器 监管。直接修改 `kube-scheduler.yaml` 文件，kubelet 会监听到文件变动直接重启 `static pod`, 配置命令如下：

    spec:
        containers:
        - command:
            - kube-scheduler
            - --authentication-kubeconfig=/etc/kubernetes/scheduler.conf
            - --authorization-kubeconfig=/etc/kubernetes/scheduler.conf
            - --bind-address=127.0.0.1
            - --kubeconfig=/etc/kubernetes/scheduler.conf
            - --leader-elect=true
            - --port=0

### 2. 如何编译 `kube-scheduler` 二进制文件
`kubernetes` 根项目直接执行 `build/run.sh make kube-scheduler`