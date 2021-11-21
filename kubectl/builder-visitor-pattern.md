## Kubectl Builder & Visitor 设计模式解析

    Kubernetes Version: v1.22@3b76c758317b
    Date: 2021.10.08

今天，跟大家分享下 `kubectl` 中常见的2种设计模式，`builder` 模式和 `visitor` 模式。我会结合具体的代码和大家一起重新温习下这些最基础的编程技巧。

### Builder 模式

1. 最简单的构造函数:

        // 默认构造函数
        p, _ := rocketmq.NewProducer()

        // 带有Timeout参数的构造函数
        p, _ := rocketmq.NewProducerWithTimeout(60*time.Second)

    > 缺点：扩展性不够

2. 含有配置参数的构造函数:

        type Config struct {
            Timeout *time.Time
            Retry int
        }

        p, _ := rocketmq.NewProducer(&config)

        func NewProducer(c *Config) *Producer {
            p := &Producer{}
            if c.Timeout != nil {
                p.Timeout = c.Timeout
            }

            if c.Retry > 0 {
                p.Retry = c.Retry
            }

            return p
        }

    > 缺点：不够简洁，需要判断参数是否为空

3. 通过闭包实现可变长参数的构造函数:

        sched, err := scheduler.New(
            cc.Client,
            // ...
            scheduler.WithKubeConfig(cc.KubeConfig),
            scheduler.WithProfiles(cc.ComponentConfig.Profiles...),
            // ...
        )

        // 返回一个闭包函数
        func WithKubeConfig(cfg *restclient.Config) Option {
            return func(o *schedulerOptions) {
                o.kubeConfig = cfg
            }
        }

        func New(client clientset.Interface, opts ...Option) (*Scheduler, error) {
            // ...
            options := defaultSchedulerOptions
            for _, opt := range opts {
                opt(&options) // 将 New 方法传入的参数给了options
            }
            // ...
        }

        configurator := &Configurator{
            componentConfigVersion:   options.componentConfigVersion,
            percentageOfNodesToScore: options.percentageOfNodesToScore,
            podInitialBackoffSeconds: options.podInitialBackoffSeconds,
            podMaxBackoffSeconds:     options.podMaxBackoffSeconds,
            extenders:                options.extenders,
            frameworkCapturer:        options.frameworkCapturer,
            parallellism:             options.parallelism,
        }

    > 缺点：不够优雅（相较于builder模式而言）

4. builder模式

    	r := f.NewBuilder().
            // ...
            RequestChunksOf(chunkSize).
            Latest().
		    Do()

        // 返回 Builder 实例
        NewBuilder() *resource.Builder

        // 更新 Builder 实例的 latest值
        func (b *Builder) Latest() *Builder {
            b.latest = true
            return b
        }

        // 更新 Builder 实例的 limitChunks
        func (b *Builder) RequestChunksOf(chunkSize int64) *Builder {
            b.limitChunks = chunkSize
            return b
        }

    > 优点: 优雅且易读。设置好默认参数，通过链式调用动态配置需要修改的参数。


### Visitor 函数
1. 结构体方法链式调用

        person.SetAge(10).SetName("kobe").SetGender("male")

2. 循环调用

        http.Handler("/apiserver", authn, authz, admission)

        for i := range methods {
            methods[i].call()
        }

3. 使用 channel

        for {
            select {
            case <- ctx.Done():
                return
            default:
                method <- methodsCh
                method.call()
            }
        }

4. visitor模式

    * 定义


            // Visitor 是带有 Visit 方法的接口
            type Visitor interface {
                Visit(VisitorFunc) error
            }

            // 一个实现了 Visitor 接口的结构体定义，需要有 visitor 字段
            type DecoratedVisitor struct {
                visitor    Visitor
                decorators []VisitorFunc
            }

            // Visit 方法最关键的地方在于，它会调用自己 `visitor` 的 `Visit` 方法，形成调用链。
            func (v DecoratedVisitor) Visit(fn VisitorFunc) error {
                return v.visitor.Visit(func(info *Info, err error) error {
                    if err != nil {
                        return err
                    }
                    // ...
                    return fn(info, nil)
                })
            }


    * 声明


            // builder 模式构造函数
            r := f.NewBuilder().
                // ...
                RequestChunksOf(chunkSize).
                Latest().
                Do()

            func (b *Builder) Do() *Result {
                r := b.visitorResult()

                // ...

                r.visitor = NewFlattenListVisitor(r.visitor, b.objectTyper, b.mapper)

                // ...
                r.visitor = NewDecoratedVisitor(r.visitor, helpers...)

                return r
            }

            // Result 结构体
            type Result struct {
                visitor Visitor
                // ...
            }

            // Result 就是实现 Visitor 接口的结构体
            func (r *Result) Visit(fn VisitorFunc) error {
                if r.err != nil {
                    return r.err
                }
                err := r.visitor.Visit(fn)
                return utilerrors.FilterOut(err, r.ignoreErrors...)
            }


        > 返回了一个 Visitor 多层嵌套的 Result 实例: `Result.Visitor -> NewDecoratedVisitor(NewFlattenListVisitor(Visitor))`

    * 调用

            infos, err := r.Infos()

            func (r *Result) Infos() ([]*Info, error) {
                // ...

                infos := []*Info{}
                err := r.visitor.Visit(func(info *Info, err error) error {
                    if err != nil {
                        return err
                    }
                    infos = append(infos, info)
                    return nil
                })

                // ...
                return infos, err
            }

        > 调用 `Result` 实例的 `Visit()` 方法，实现类似深度遍历搜索(`DFS`)的方法调用: `Visiotr.Visit()` -> `NewFlattenListVisitor.Visit()` -> `NewDecoratedVisitor.Visit()`

    * 注意

        假设我们的Visit方法定义如下，num 表示变量数值:

            func (v Num<num>Visitor) Visit(fn VisitorFunc) error {
                return v.visitor.Visit(func(info *Info, err error) error {
                    fmt.Println("before function call-<num>")
                    err = fn(info, err)
                    if err == nil {
                        fmt.Println(err)
                    }
                    fmt.Println("after function call-<num>")
                    return err
                })
            }

        假设我们 visitor 嵌套关系为 `Result.Visitor = Num1Visitor(Num2Visitor(Num3Visitor))`，最终输出结果应该是:

            before function call-1
            before function call-2
            before function call-3
            ...
            after function call-3
            after function call-2
            after function call-1

        其实就是一个单分支的 DFS 遍历树。

## 总结
`kubernetes` 源代码中还有很多设计模式思想，今天只是介绍了 `kubectl` 中的 `builder` 和 `visitor` 两种，这也是我最初看源码的时候经常遇到和迷惑的地方，希望可以对大家有所帮助。