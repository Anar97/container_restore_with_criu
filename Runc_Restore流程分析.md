# runc调用lazy-page restore的流程

## （1）
`/runc/restore.go`

```golang
var restoreCommand = cli.Command{
    Name:  "restore",
	Usage: "restore a container from a previous checkpoint",
	ArgsUsage: `<container-id>
Where "<container-id>" is the name for the instance of the container to be
restored.`,
	Description: `Restores the saved state of the container instance that was previously saved
using the runc checkpoint command.`,
	Flags: []cli.Flag{
        ……
    }
    //上面这些都不重要
    Action: func(context *cli.Context) error{
        ……
        spec, err := setupSpec(context)
        ……
        options := criuOptions(context)
        ……
        status, err := startContainer(context, spec, CT_ACT_RESTORE, options)
        
    }
}
```

`func criuOptions(context *cli.Context) *libcontainer.CriuOpts `函数的定义就在同一个代码文件中
而 *CriuOpts* 的定义在`runc/libcontainer/criu_opts_linux.go`中
```golang
type CriuOpts struct {
	ImagesDirectory         string             // directory for storing image files
	WorkDirectory           string             // directory to cd and write logs/pidfiles/stats to
	ParentImage             string             // directory for storing parent image files in pre-dump and dump
	LeaveRunning            bool               // leave container in running state after checkpoint
	TcpEstablished          bool               // checkpoint/restore established TCP connections
	ExternalUnixConnections bool               // allow external unix connections
	ShellJob                bool               // allow to dump and restore shell jobs
	FileLocks               bool               // handle file locks, for safety
	PreDump                 bool               // call criu predump to perform iterative checkpoint
	PageServer              CriuPageServerInfo // allow to dump to criu page server
	VethPairs               []VethPairName     // pass the veth to criu when restore
	ManageCgroupsMode       cgMode             // dump or restore cgroup mode
	EmptyNs                 uint32             // don't c/r properties for namespace from this mask
	AutoDedup               bool               // auto deduplication for incremental dumps
	LazyPages               bool               // restore memory pages lazily using userfaultfd
	StatusFd                string             // fd for feedback when lazy server is ready
}

```
## (2)
`runc/utils_linux.go`
```golang
func startContainer(context *cli.Context, spec *specs.Spec, action CtAct, criuOpts *libcontainer.CriuOpts) (int, error) {
    ……
    container, err := createContainer(context, id, spec)
    ……
    r := &runner{
		……
	}
	return r.run(spec.Process)
}
```
runner的定义在同一个文件中，调用run函数
```golang
func (r *runner) run(config *specs.Process) (int, error) {
    ……
    switch r.action {
	case CT_ACT_CREATE:
		err = r.container.Start(process)
	case CT_ACT_RESTORE:
		err = r.container.Restore(process, r.criuOpts)
	case CT_ACT_RUN:
		err = r.container.Run(process)
	default:
		panic("Unknown action")
	}
	if err != nil {
		return -1, err
	}

    ……
}
```
*r.container.Restore*中*container*是一个*linuxContainer*,其定义在`runc/libcontainer/container_linux.go`中

## (3)
`runc/libcontainer/container_linux.go`
```golang
func (c *linuxContainer) Restore(process *Process, criuOpts *CriuOpts) error {
    var extraFiles []*os.File
    ……
    req := &criurpc.CriuReq{
		Type: &t,
		Opts: &criurpc.CriuOpts{
			……
		},
	}
    ……
    return c.criuSwrk(process, req, criuOpts, true, extraFiles)
}
```

*Checkpoint*、*Restore*、*checkCriuVersion*、*checkCriuFeatures*都会调用此函数
```golang
func (c *linuxContainer) criuSwrk(process *Process, req *criurpc.CriuReq, opts *CriuOpts, applyCgroups bool, extraFiles []*os.File) error {
    fds, err := unix.Socketpair(unix.AF_LOCAL, unix.SOCK_SEQPACKET|unix.SOCK_CLOEXEC, 0)
    ……
    cmd := exec.Command(c.criuPath, args...)
	if process != nil {
		cmd.Stdin = process.Stdin
		cmd.Stdout = process.Stdout
		cmd.Stderr = process.Stderr
	}


}
```
其中的*criuPath*在`runc/utils_linux.go`中，通过*startContainer*函数中的`container, err := createContainer(context, id, spec)`创建出来的，而在*createContainer*中：
```golang
func createContainer(context *cli.Context, id string, spec *specs.Spec) (libcontainer.Container, error) {
    ……
	factory, err := loadFactory(context)
	……
	return factory.Create(id, config)
}
```
而*loadFactory*返回的是一个*libcontainer.Factory*类型，
在`runc/libcontainer/factory_linux.go`中：
```golang
type LinuxFactory struct {
    ……
    // CriuPath is the path to the criu binary used for checkpoint and restore of
	// containers.
	CriuPath string
    ……
}
```
通过上面的代码可以看出，CriuPath是criu程序的地址

所以criuSwrk其实也是通过执行指令的方式来创建criu进程的？
