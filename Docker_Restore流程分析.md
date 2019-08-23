# 1.客户端部分

## (1)
`cli/cmd/docker/docker.go`

```golang
func main()
{
    dockerCli, err := command.NewDockerCli()
    runDocker(dockerCli)
}

……

func runDocker(dockerCli *command.DockerCli) error {
	tcmd := newDockerCommand(dockerCli)
    ……
}

func newDockerCommand(dockerCli *command.DockerCli) *cli.TopLevelCommand {
    commands.AddCommands(cmd, dockerCli)
    ……
}

```

## (2)
`cli/cli/command/commands/commands.go`

```golang
func AddCommands(cmd *cobra.Command, dockerCli command.Cli) {
    ……
    container.NewContainerCommand(dockerCli),
    ……
}
```

## (3)
`cli/cli/command/container/cmd.go`

```golang
func NewContainerCommand(dockerCli command.Cli) *cobra.Command {
    ……
    NewStartCommand(dockerCli),
    ……
}
```

## (4)
`cli/cli/command/cli.go`

```golang
func NewDockerCli(ops ...DockerCliOption) (*DockerCli, error) {

}

```

## (5)
`cli/cli/command/container/start.go`

```golang
func NewStartCommand(dockerCli command.Cli) *cobra.Command {
	var opts startOptions

	cmd := &cobra.Command{
		Use:   "start [OPTIONS] CONTAINER [CONTAINER...]",
		Short: "Start one or more stopped containers",
		Args:  cli.RequiresMinArgs(1),
		RunE: func(cmd *cobra.Command, args []string) error {
			opts.containers = args
			return runStart(dockerCli, &opts)		//注意这里
		},
	}

	flags := cmd.Flags()
	flags.BoolVarP(&opts.attach, "attach", "a", false, "Attach STDOUT/STDERR and forward signals")
	flags.BoolVarP(&opts.openStdin, "interactive", "i", false, "Attach container's STDIN")
	flags.StringVar(&opts.detachKeys, "detach-keys", "", "Override the key sequence for detaching a container")

	flags.StringVar(&opts.checkpoint, "checkpoint", "", "Restore from this checkpoint")
	flags.SetAnnotation("checkpoint", "experimental", nil)
	flags.SetAnnotation("checkpoint", "ostype", []string{"linux"})
	flags.StringVar(&opts.checkpointDir, "checkpoint-dir", "", "Use a custom checkpoint storage directory")
	flags.SetAnnotation("checkpoint-dir", "experimental", nil)
	flags.SetAnnotation("checkpoint-dir", "ostype", []string{"linux"})

	

	return cmd
}
```

应在此处加上lazy-restore的选项

```golang
+	flags.StringVar(&opts.checkpointRepo, "lazy-restore", "", "Restore container using lazy-pages with remote checkpoint image")
+	flags.SetAnnotation("lazy-restore", "experimental", nil)
+	flags.SetAnnotation("lazy-restore", "ostype", []string{"linux"})
```

应该使用指令docker start --lazy-restore (检查点镜像地址) (container)   

```golang
func runStart(dockerCli command.Cli, opts *startOptions) error {
	……
	if err := dockerCli.Client().ContainerStart(ctx, c.ID, startOptions); err != nil {
		cancelFun()
		<-cErr
		if c.HostConfig.AutoRemove {
			// wait container to be removed
			<-statusChan
		}
		return err
	}

}
```

# 2.发送请求
## (1)
moby/client/container_start.go
```golang
func (cli *Client) ContainerStart(ctx context.Context, containerID string, options types.ContainerStartOptions) error {
	query := url.Values{}
	if len(options.CheckpointID) != 0 {
		query.Set("checkpoint", options.CheckpointID)
	}
	if len(options.CheckpointDir) != 0 {
		query.Set("checkpoint-dir", options.CheckpointDir)
	}




	/*此处来讲就是发送一个query*/
	resp, err := cli.post(ctx, "/containers/"+containerID+"/start", query, nil, nil)
	ensureReaderClosed(resp)
	return err
}
```
应该加上如下内容
```golang
+	if len(options.CheckpointRepo) != 0 {
+		query.Set("lazy-Restore", options.CheckpointDir)
+	}
```

`moby/client/request.go`

```golang
func (cli *Client) post(ctx context.Context, path string, query url.Values, obj interface{}, headers map[string][]string) (serverResponse, error) {
	body, headers, err := encodeBody(obj, headers)
	if err != nil {
		return serverResponse{}, err
	}
	return cli.sendRequest(ctx, "POST", path, query, body, headers)
}
```

同上
```golang
func (cli *Client) sendRequest(ctx context.Context, method, path string, query url.Values, body io.Reader, headers headers) (serverResponse, error) {
	req, err := cli.buildRequest(method, cli.getAPIPath(ctx, path, query), body, headers)
	……
	resp, err := cli.doRequest(ctx, req)
	……
}
```

同上
```golang
func (cli *Client) doRequest(ctx context.Context, req *http.Request) (serverResponse, error) {
	serverResp := serverResponse{statusCode: -1, reqURL: req.URL}
	req = req.WithContext(ctx)
	resp, err := cli.client.Do(req)				//Do的相关定义在 net/http/client包内

	……

}
```

到此为止，发送到了服务端


## (2)
`moby/api/server/router/container/container.go`

```golang
func (r *containerRouter) initRoutes() {
	……
	router.NewPostRoute("/containers/{name:.*}/start", r.postContainersStart),
	……
}
```

`moby/api/server/router/local.go`

```golang
func NewPostRoute(path string, handler httputils.APIFunc, opts ...RouteWrapper) Route {
	return NewRoute("POST", path, handler, opts...)
}
```


`moby/api/server/router/container/container_routes.go`

```golang
func (s *containerRouter) postContainersStart(ctx context.Context, w http.ResponseWriter, r *http.Request, vars map[string]string) error {

}
```

# 3.daemon部分
## （3）
`moby/daemon/start.go`

```golang
func (daemon *Daemon) ContainerStart(name string, hostConfig *containertypes.HostConfig, checkpoint string, checkpointDir string) error {
	if checkpoint != "" && !daemon.HasExperimental() {
		return errdefs.InvalidParameter(errors.New("checkpoint is only supported in experimental mode"))
	}
	container, err := daemon.GetContainer(name)
    ……
    return daemon.containerStart(container, checkpoint, checkpointDir, true)
}
```

containerStart prepares the container to run by setting up everything the container needs, such as storage and networking, as well as links between containers. The container is left waiting for a signal to begin running.
```golang
func (daemon *Daemon) containerStart(container *container.Container, checkpoint string, checkpointDir string, resetRestartManager bool) (err error) {
	……
	if err := daemon.conditionalMountOnStart(container); err != nil {
		return err
	}

	if err := daemon.initializeNetworking(container); err != nil {
		return err
	}

	spec, err := daemon.createSpec(container)
	if err != nil {
		return errdefs.System(err)
	}
	……

	if checkpoint != "" {
		checkpointDir, err = getCheckpointDir(checkpointDir, checkpoint, container.Name, container.ID, container.CheckpointDir(), false)
		if err != nil {
			return err
		}
	}
	createOptions, err := daemon.getLibcontainerdCreateOptions(container)

	err = daemon.containerd.Create(ctx, container.ID, spec, createOptions)
	……
	pid, err := daemon.containerd.Start(……)
	……
	container.SetRunning(pid, true)
	……

}
```
其中*daemon.containerd*是*libcontainerdtypes.Client*类型，而*libcontainerdtypes "github.com/docker/docker/libcontainerd/types"

*Start*的定义为`Start(ctx context.Context, containerID, checkpointDir string, withStdin bool, attachStdio StdioCallback) (pid int, err error)`

# 4.libcontainerd


## (1) 
*libcontainerdtypes.Client*接口用*remote*包中的client结构体实现
`moby/libcontainerd/remote/client.go`

```golang
func (c *client) Start(ctx context.Context, id, checkpointDir string, withStdin bool, attachStdio libcontainerdtypes.StdioCallback) (int, error) {
	ctr, err := c.getContainer(ctx, id)
	var (
		cp             *types.Descriptor
		t              containerd.Task
		rio            cio.IO
		stdinCloseSync = make(chan struct{})
	)

	……
	if checkpointDir != ""{
		……
	}
	spec, err := ctr.Spec(ctx)

	labels, err := ctr.Labels(ctx)

	bundle := labels[DockerContainerBundlePath]

	uid, gid := getSpecUser(spec)

	t, err = ctr.NewTask(ctx,func(id string) (cio.IO, error),func(_ context.Context, _ *containerd.Client, info *containerd.TaskInfo) error)

	if err := t.Start(ctx); err != nil {

	}
}
```

`getContainer(ctx, id)`返回一个*containerd.container*
```golang
func (c *client) getContainer(ctx context.Context, id string) (containerd.Container, error){
	ctr, err := c.client.LoadContainer(ctx, id)
	//此处的client是一个containerd.client类型
	if err != nil {
		if containerderrors.IsNotFound(err) {
			return nil, errors.WithStack(errdefs.NotFound(errors.New("no such container")))
		}
		return nil, wrapError(err)
	}
	return ctr, nil
}
```

# 5.Containerd

##(1)
`containerd\container.go`

```golang
import(
	……
	"github.com/containerd/containerd/containers"
	……
)

type Container interface{
	……
}

type container struct {
	client   *Client
	id       string
	metadata containers.Container
}
```

`containerd/client.go`

*getContainer*函数中调用了*LoadContainer*返回了一个containerd.container
```golang
func (c *Client) LoadContainer(ctx context.Context, id string) (Container, error) {
	r, err := c.ContainerService().Get(ctx, id)
	if err != nil {
		return nil, err
	}
	return containerFromRecord(c, r), nil
}
```
后面的事不是很复杂了，就麻烦姜庆阳了
