+++
title = "k8sを雑に読んでいく1 (kube-scheduler1)"
date = 2024-04-17
+++

## kube-scheduler

雑に読んでいく,読んだ後に気づいたが、[自作して学ぶKubernetes Scheduler
](https://engineering.mercari.com/blog/entry/20211220-create-your-kube-scheduler/)が詳しい。


<https://github.com/kubernetes/kubernetes/blob/master/cmd/kube-scheduler/scheduler.go>

> スケジューリングの概要
スケジューラーは新規に作成されたPodで、Nodeに割り当てられていないものを監視します。スケジューラーは発見した各Podのために、稼働させるべき最適なNodeを見つけ出す責務を担っています。そのスケジューラーは下記で説明するスケジューリングの原理を考慮に入れて、NodeへのPodの割り当てを行います。Podが特定のNodeに割り当てられる理由を理解したい場合や、カスタムスケジューラーを自身で作ろうと考えている場合、このページはスケジューリングに関して学ぶのに役立ちます。

<https://kubernetes.io/ja/docs/concepts/scheduling-eviction/kube-scheduler/>




```go
func main() {
	command := app.NewSchedulerCommand()
	code := cli.Run(command)
	os.Exit(code)
}
```

> The Kubernetes scheduler is a control plane process which assigns Pods to Nodes. The scheduler determines which Nodes are valid placements for each Pod in the scheduling queue according to constraints and available resources. The scheduler then ranks each valid Node and binds the Pod to a
suitable Node. Multiple different schedulers may be used within a cluster; kube-scheduler is the reference implementation.


> Kubernetesのスケジューラーは、コントロールプレーンのプロセスであり、Podをノードに割り当てます。スケジューラーは、制約や利用可能なリソースに応じて、スケジューリングキュー内の各Podに対して妥当な配置先のノードを決定します。その後、スケジューラーは各妥当なノードをランク付けし、Podを適切なノードにバインドします。クラスター内で複数の異なるスケジューラーを使用することができますが、kube-schedulerがそのリファレンス実装です。

リファレンス実装了解。GPU向けとかに異なるスケジューラーを実装できる、という話でしょうか？


MarkFlagFilenameの文脈 config/yaml/yml/jsonに補完を引数にとりたいようだ.
```go
	if err := cmd.MarkFlagFilename("config", "yaml", "yml", "json"); err != nil {
		klog.Background().Error(err, "Failed to mark flag filename")
	}
```

LeaderElectionを行っている.分散っぽい.
```go
	if cc.ComponentConfig.LeaderElection.LeaderElect {
		checks = append(checks, cc.LeaderElection.WatchDog)
	}

	waitingForLeader := make(chan struct{})
	isLeader := func() bool {
		select {
		case _, ok := <-waitingForLeader:
			// if channel is closed, we are leading
			return !ok
		default:
			// channel is open, we are waiting for a leader
			return false
		}
	}
```

Informers: 情報提供者
InformerFactoryは各リソースタイプの変更を監視する役割、すべてのハンドラが同期するのを待つ

```go
startInformersAndWaitForSync := func(ctx context.Context) {
		// Start all informers.
		cc.InformerFactory.Start(ctx.Done())
		// DynInformerFactory can be nil in tests.
		if cc.DynInformerFactory != nil {
			cc.DynInformerFactory.Start(ctx.Done())
		}

		// Wait for all caches to sync before scheduling.
		cc.InformerFactory.WaitForCacheSync(ctx.Done())
		// DynInformerFactory can be nil in tests.
		if cc.DynInformerFactory != nil {
			cc.DynInformerFactory.WaitForCacheSync(ctx.Done())
		}

		// Wait for all handlers to sync (all items in the initial list delivered) before scheduling.
		if err := sched.WaitForHandlersSync(ctx); err != nil {
			logger.Error(err, "waiting for handlers to sync")
		}

		logger.V(3).Info("Handlers synced")
	}
```

LeaderElectにしろ、そうでないにしろ
```
startInformersAndWaitForSync(ctx)
```
を実行する

Scheduler(ControlPlaneComponent)のメソッド、WaitForHandlersSync
登録されているhandlerをすべて実行する

```go
// WaitForHandlersSync waits for EventHandlers to sync.
// It returns true if it was successful, false if the controller should shut down
func (sched *Scheduler) WaitForHandlersSync(ctx context.Context) error {
	return wait.PollUntilContextCancel(ctx, syncedPollPeriod, true, func(ctx context.Context) (done bool, err error) {
		for _, handler := range sched.registeredHandlers {
			if !handler.HasSynced() {
				return false, nil
			}
		}
		return true, nil
	})
}
```

helper functionのaddAllEventHandlersを見るとどんなものがhandlerかがわかる. どんなエンティティか、どんなフィルタかによって動作を変える.

- v1.Pod : どんなPodか / assignedPod, responsibleForPod(sched.Profilesに基づく)
- v1.Nodes
- Group/Version/Kindの上記以外の CSINode, CSIDriver, CSIStorageCapacity, PersistentVolume, PersistentVolumeClaim, PodSchedulingContext, ResourceClaim, ResourceClass, ResourceClassParameters, StorageClass

Podの動作方法として
```go
			Handler: cache.ResourceEventHandlerFuncs{
				AddFunc:    sched.addPodToSchedulingQueue,
				UpdateFunc: sched.updatePodInSchedulingQueue,
				DeleteFunc: sched.deletePodFromSchedulingQueue,
			},
```
で定義されるようにAddはaddPodToSchedulingQueue, UpdateはupdatePodInSchedulingQueueのように動作が定められている.つまり, queueに追加する.


```go
// SchedulingQueue is an interface for a queue to store pods waiting to be scheduled.
// The interface follows a pattern similar to cache.FIFO and cache.Heap and
// makes it easy to use those data structures as a SchedulingQueue.
type SchedulingQueue interface {
	framework.PodNominator
	Add(logger klog.Logger, pod *v1.Pod) error

```

ここまでがControllerの役割, queueに追加して終わり.queueはシンプルなqueueではなくて, 設定によって変更可能な実装になっている.queue自体がキモ.
```go
	podQueue := internalqueue.NewSchedulingQueue(
		profiles[options.profiles[0].SchedulerName].QueueSortFunc(),
		informerFactory,
		internalqueue.WithPodInitialBackoffDuration(time.Duration(options.podInitialBackoffSeconds)*time.Second),
		internalqueue.WithPodMaxBackoffDuration(time.Duration(options.podMaxBackoffSeconds)*time.Second),
		internalqueue.WithPodLister(podLister),
		internalqueue.WithPodMaxInUnschedulablePodsDuration(options.podMaxInUnschedulablePodsDuration),
		internalqueue.WithPreEnqueuePluginMap(preEnqueuePluginMap),
		internalqueue.WithQueueingHintMapPerProfile(queueingHintsPerProfile),
		internalqueue.WithPluginMetricsSamplePercent(pluginMetricsSamplePercent),
		internalqueue.WithMetricsRecorder(*metricsRecorder),
	)
```


ここまでScheduling Cycleといい、これから先はBinding Cycleというらしい


続き: 