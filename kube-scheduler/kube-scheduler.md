# kube-scheduler 详细分析

* 总入口 cmd/kube-scheduler/scheduler.go
入口函数为 app.NewSchedulerCommand()，进入到app/server.go runCommand()是主体函数，先开始主要看下这个函数实现以及参数

先大概来看下runCommand函数(精简了下)
```
// runCommand runs the scheduler.
func runCommand(cmd *cobra.Command, args []string, opts *options.Options, registryOptions ...Option) error {
    ...
    .......

	c, err := opts.Config()
	if err != nil {
		fmt.Fprintf(os.Stderr, "%v\n", err)
		os.Exit(1)
	}

	stopCh := make(chan struct{})
	// Get the completed config
	cc := c.Complete()

	algorithmprovider.ApplyFeatureGates()

	if cz, err := configz.New("componentconfig"); err == nil {
		cz.Set(cc.ComponentConfig)
	} else {
		return fmt.Errorf("unable to register configz: %s", err)
	}

	return Run(cc, stopCh, registryOptions...)
}
```
从头往下看，可以看到比较重要的几个方法
> c, err := opts.Config()
> cc := c.Complete()

我们先从第一个看起,在看这个方法前，我们需要先看下方法的主人是谁,也就是这个 Options 这个结构体,Options 结构体如下:
```
// Options has all the params needed to run a Scheduler
type Options struct {
	// The default values. These are overridden if ConfigFile is set or by values in InsecureServing.
	ComponentConfig kubeschedulerconfig.KubeSchedulerConfiguration

	SecureServing           *apiserveroptions.SecureServingOptionsWithLoopback
	CombinedInsecureServing *CombinedInsecureServingOptions
	Authentication          *apiserveroptions.DelegatingAuthenticationOptions
	Authorization           *apiserveroptions.DelegatingAuthorizationOptions
	Deprecated              *DeprecatedOptions

	// ConfigFile is the location of the scheduler server's configuration file.
	ConfigFile string

	// WriteConfigTo is the path where the default configuration will be written.
	WriteConfigTo string

	Master string
}
```
可以看到Option这个结构体里面就定义了kube-scheduler启动参数，最明显的就是指定master url参数以及配置文件ConfigFile path地址
，我们可以很明显的看到比较重要的参数设定应该就是ComponentConfig以及Deprecated(现在社区推荐使用configfile的配置方式，所以传统的
命名行参数就是会被遗弃的)

```
// Config return a scheduler config object
func (o *Options) Config() (*schedulerappconfig.Config, error) {
    ...
    ......

	c := &schedulerappconfig.Config{}
	if err := o.ApplyTo(c); err != nil {
		return nil, err
	}

	// Prepare kube clients.
	client, leaderElectionClient, eventClient, err := createClients(c.ComponentConfig.ClientConnection, o.Master, c.ComponentConfig.LeaderElection.RenewDeadline.Duration)
	if err != nil {
		return nil, err
	}

	coreBroadcaster := record.NewBroadcaster()
	coreRecorder := coreBroadcaster.NewRecorder(scheme.Scheme, corev1.EventSource{Component: c.ComponentConfig.SchedulerName})

	c.Client = client
	c.InformerFactory = informers.NewSharedInformerFactory(client, 0)
	c.PodInformer = factory.NewPodInformer(client, 0)
	c.EventClient = eventClient.EventsV1beta1()
	c.CoreEventClient = eventClient.CoreV1()
	c.CoreBroadcaster = coreBroadcaster
	c.LeaderElection = leaderElectionConfig

	return c, nil
}
```
我们再接着看下opt.Config()这个方法，可以看到里面有个关键的方法 o.ApplyTo(c) 调用来填冲　Config 这个结构体，之后把各种必需的 Client 配置给
Config,而 Config 这个结构体有着 kube-scheduer 启动所需的必要配置(例如:informer ... )，再来看下这个ApplyTo的方法，很简单如果有ConfigFile，
就从ConfigFile读取配置,不然调用老的遗弃配置来填冲
```
// ApplyTo applies the scheduler options to the given scheduler app configuration.
func (o *Options) ApplyTo(c *schedulerappconfig.Config) error {
	if len(o.ConfigFile) == 0 {
		c.ComponentConfig = o.ComponentConfig
        ...
	} else {
		cfg, err := loadConfigFromFile(o.ConfigFile)
        ...
    }
    ...
```

接着再来看下　cc := c.Complete()　这个方法看起来就是重新包装了下　Config 这个结构体





runCommand() 主要参数有(*options.Options, registryOptions ...Option), 其中的registryOptions ...Option这个应该是给scheduler framework plugins注册用的，后面再分析，先看下options.Options

