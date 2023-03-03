# Karmada优雅故障迁移设计与实现原理

Karmada在v1.3版本引入了新的`Taint-Manager-Controller`和`Graceful-Eviction-Controller`，用打`Taint`的方式实现了优雅的故障迁移。

> 本文基于[Karmada release-v1.5.0](https://github.com/karmada-io/karmada/tree/v1.5.0)源码来分析其实现原理。

## 基本原理

> 在Karmada V1.4之后，优雅故障迁移被默认开启，`karmada-controller-manager`组件启动参数添加了`--feature-gates=Failover=true,GracefulEviction=true`

Karmada通过给成员集群打`Taint`的方式，来标识一个集群是否可以接受工作负载。在成员集群故障时（控制面监听不到成员集群状态），会给这个成员集群打上`Taint`。用户也可以手动给成员集群打上`Taint`，来驱逐成员集群上的工作负载。



### 为故障集群添加污点

Karmada设置了四种污点类型，以描述成员集群发生故障时的状态。

1. 当成员集群的`Ready`字段为`False`时

```
key: cluster.karmada.io/not-ready
effect: NoSchedule
```

2. 当成员集群的`Ready`字段为`Unknow`时

```
key: cluster.karmada.io/unreachable
effect: NoSchedule
```

如果不健康的集群在一段时间后没有恢复的话（默认为5分钟，可以通过`karmada-controller-manager`组件的启动参数`--failover-eviction-timeout`来设置），会被打上`Effect: NoExecute`的污点，以表达需要迁移该集群上的工作负载了。

3. 当成员集群的`Ready`字段为`False`时

```
key: cluster.karmada.io/not-ready
effect: NoExecute
```

4. 当成员集群的`Ready`字段为`Unknow`时

```
key: cluster.karmada.io/unreachable
effect: NoExecute
```

### 容忍集群的污点

当用户创建`PropragationPolicy/ClusterPropagationPolicy`时，Karmada会通过Webhook自动为该`PP/CPP`添加污点容忍，**这样可以控制在集群被打上`effect: NoExecute`污点后，多久开始触发迁移当前集群的工作负载。**

```
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: nginx-propagation
  namespace: default
spec:
  placement:
    clusterTolerations: # Karmada自动添加的
    - effect: NoExecute
      key: cluster.karmada.io/not-ready
      operator: Exists
      tolerationSeconds: 300
    - effect: NoExecute
      key: cluster.karmada.io/unreachable
      operator: Exists
      tolerationSeconds: 300
  resourceSelectors:
  - apiVersion: apps/v1
    kind: Deployment
    name: nginx
    namespace: default
```

这个`tolerationSeconds`默认为300s，可以通过`karmada-controller-manager`组件的参数`--default-not-ready-toleration-seconds`和`default-unreachable-toleration-seconds`来设置。

### 触发故障转移

当Karmada发现`PP/CPP`的容忍故障集群的时间已经到了之后，会将该集群从`调度结果`中移除，并触发重调度。

- 在重调度的过程中，也需要满足`ClusterAffinity`和`SpreadConstraints`等条件。
- 之前被调度到健康集群的工作负载会被保留。

对于不同调度类型（`Duplicated`和`Divided`），Karmada尽力将副本调度到健康的集群，具体可以参考[官方示例](https://karmada.io/docs/userguide/failover/failover-analysis#duplicated-schedule-type)。

### 优雅驱逐

为了避免在驱逐工作负载时对外服务不可用，Karmada会在启动新的工作负载之后，再驱逐现有的工作负载。

该功能通过在`ResourceBinding/ClusterResourceBinding`添加`GracefulEvictionTasks`字段表示**优雅驱逐任务队列**.

`Graceful-Eviction-Controller` 在确定了重新调度结果后，会 检查资源的健康状态。在超过间隔时间后才将故障集群资源移除。这个间隔时间可以通过 `--graceful-eviction-timeout` 时间设置，默认 10 分钟。最后将这个 task 从队列移除。

## 源码分析

Karmada优雅故障迁移功能是由多个组件配合使用实现的，其中最主要有`Cluster-Controller`，`Taint-Manager-Controller`和`Graceful-Eviction-Controller`，接下来主要分析这几个组件。

### Cluster-Controller

> 代码在`pkg/controllers/cluster/cluster_controller.go`

这个Controller负责所有集群的调协，在此只分析其为集群自动打污点的功能。其中主要有五种污点类型。

1. cluster.karmada.io/not-ready:NoSchedule
2. cluster.karmada.io/not-ready:NoExecute
3. cluster.karmada.io/unreachable:NoSchedule
4. cluster.karmada.io/unreachable:NoExecute
5. cluster.karmada.io/terminating:NoExecute

**工作流程如下：**
- 当新的成员集群注册后还未Ready时，给这个集群打上`cluster.karmada.io/not-ready:NoSchedule`污点，防止在集群还未就绪的时候将工作负载调度到该集群。
- 当删除集群时，给这个集群打上`cluster.karmada.io/terminating:NoExecute`污点，驱逐该集群上的工作负载。
- 当集群状态为`NotReady`时，给集群打上`cluster.karmada.io/not-ready:NoSchedule`污点；当集群状态为`Unknow`时，给集群打上`cluster.karmada.io/unreachable:NoSchedule`，防止将后续工作负载调度到该集群上。
- 持续监听集群的健康状态，如果集群的故障时间超过了`--failover-eviction-timeout`参数设置的时间，如果集群状态为`NotReady`，给集群打上`cluster.karmada.io/not-ready:NoExecute`污点；如果集群状态为`Unknown`，则给集群打上`cluster.karmada.io/unreachable:NoExecute`污点，驱逐该集群上的工作负载。
- 如果集群恢复正常，则移除所有这些自动添加的污点。



调协对象是`Cluster`。

```golang
func (c *Controller) Reconcile(ctx context.Context, req controllerruntime.Request) (controllerruntime.Result, error) {
	klog.V(4).Infof("Reconciling cluster %s", req.NamespacedName.Name)

	cluster := &clusterv1alpha1.Cluster{}
	......

	if !cluster.DeletionTimestamp.IsZero() {
		return c.removeCluster(ctx, cluster)
	}

	return c.syncCluster(ctx, cluster)
}

// 在移除集群的时候，会给集群打cluster.karmada.io/terminating:NoExecute污点
func (c *Controller) removeCluster(ctx context.Context, cluster *clusterv1alpha1.Cluster) (controllerruntime.Result, error) {
	// add terminating taint before cluster is deleted
	if err := utilhelper.UpdateClusterControllerTaint(ctx, c.Client, []*corev1.Taint{TerminatingTaintTemplate}, nil, cluster); err != nil {
		c.EventRecorder.Event(cluster, corev1.EventTypeWarning, events.EventReasonRemoveTargetClusterFailed, err.Error())
		klog.ErrorS(err, "Failed to update terminating taint", "cluster", cluster.Name)
		return controllerruntime.Result{Requeue: true}, err
	}
}

func (c *Controller) syncCluster(ctx context.Context, cluster *clusterv1alpha1.Cluster) (controllerruntime.Result, error) {
 ......
	// taint cluster by condition
	// 根据集群的状态打taint
	err = c.taintClusterByCondition(ctx, cluster)
	if err != nil {
		c.EventRecorder.Event(cluster, corev1.EventTypeWarning, events.EventReasonTaintClusterByConditionFailed, err.Error())
		return controllerruntime.Result{Requeue: true}, err
	}
  ...
}

func (c *Controller) taintClusterByCondition(ctx context.Context, cluster *clusterv1alpha1.Cluster) error {
	currentReadyCondition := meta.FindStatusCondition(cluster.Status.Conditions, clusterv1alpha1.ClusterConditionReady)
	var err error
	if currentReadyCondition != nil {
		switch currentReadyCondition.Status {
		case metav1.ConditionFalse:
			// Add NotReadyTaintTemplateForSched taint immediately.
			// 集群状态为False，立即打上cluster.karmada.io/not-ready:NoSchedule污点，并移除cluster.karmada.io/unreachable:NoSchedule污点
			if err = utilhelper.UpdateClusterControllerTaint(ctx, c.Client, []*corev1.Taint{NotReadyTaintTemplateForSched}, []*corev1.Taint{UnreachableTaintTemplateForSched}, cluster); err != nil {
				klog.ErrorS(err, "Failed to instantly update UnreachableTaintForSched to NotReadyTaintForSched, will try again in the next cycle.", "cluster", cluster.Name)
			}
		case metav1.ConditionUnknown:
			// Add UnreachableTaintTemplateForSched taint immediately.
			// 集群状态为Unknown，立即打上cluster.karmada.io/unreachable:NoSchedule污点，并移除cluster.karmada.io/not-ready:NoSchedule污点
			if err = utilhelper.UpdateClusterControllerTaint(ctx, c.Client, []*corev1.Taint{UnreachableTaintTemplateForSched}, []*corev1.Taint{NotReadyTaintTemplateForSched}, cluster); err != nil {
				klog.ErrorS(err, "Failed to instantly swap NotReadyTaintForSched to UnreachableTaintForSched, will try again in the next cycle.", "cluster", cluster.Name)
			}
		case metav1.ConditionTrue:
			// 集群健康，则移除相关污点
			if err = utilhelper.UpdateClusterControllerTaint(ctx, c.Client, nil, []*corev1.Taint{NotReadyTaintTemplateForSched, UnreachableTaintTemplateForSched}, cluster); err != nil {
				klog.ErrorS(err, "Failed to remove schedule taints from cluster, will retry in next iteration.", "cluster", cluster.Name)
			}
		}
	} else {
		// Add NotReadyTaintTemplateForSched taint immediately.
		// 对于新添加的集群，在未Ready前，立即打上cluster.karmada.io/not-ready:NoSchedule污点
		if err = utilhelper.UpdateClusterControllerTaint(ctx, c.Client, []*corev1.Taint{NotReadyTaintTemplateForSched}, nil, cluster); err != nil {
			klog.ErrorS(err, "Failed to add a NotReady taint to the newly added cluster, will try again in the next cycle.", "cluster", cluster.Name)
		}
	}
	return err
}

// Start starts an asynchronous loop that monitors the status of cluster.
// 启动一个协程来异步监听所有集群的健康状态
func (c *Controller) Start(ctx context.Context) error {
	klog.Infof("Starting cluster health monitor")
	defer klog.Infof("Shutting cluster health monitor")

	// Incorporate the results of cluster health signal pushed from cluster-status-controller to master.
	go wait.UntilWithContext(ctx, func(ctx context.Context) {
		if err := c.monitorClusterHealth(ctx); err != nil {
			klog.Errorf("Error monitoring cluster health: %v", err)
		}
	}, c.ClusterMonitorPeriod)
	<-ctx.Done()

	return nil
}

func (c *Controller) monitorClusterHealth(ctx context.Context) (err error) {
	clusterList := &clusterv1alpha1.ClusterList{}
	if err = c.Client.List(ctx, clusterList); err != nil {
		return err
	}

	clusters := clusterList.Items
	for i := range clusters {
		cluster := &clusters[i]
		var observedReadyCondition, currentReadyCondition *metav1.Condition
		if err = wait.PollImmediate(MonitorRetrySleepTime, MonitorRetrySleepTime*HealthUpdateRetry, func() (bool, error) {
			// Cluster object may be changed in this function.
			observedReadyCondition, currentReadyCondition, err = c.tryUpdateClusterHealth(ctx, cluster)
			......
		}); err != nil {
			klog.Errorf("Update health of Cluster '%v' from Controller error: %v. Skipping.", cluster.Name, err)
			continue
		}

		if currentReadyCondition != nil {
			// 根据集群当前状态来处理集群上的Taint
			if err = c.processTaintBaseEviction(ctx, cluster, observedReadyCondition); err != nil {
				klog.Errorf("Failed to process taint base eviction error: %v. Skipping.", err)
				continue
			}
		}
	}

	return nil
}

func (c *Controller) processTaintBaseEviction(ctx context.Context, cluster *clusterv1alpha1.Cluster, observedReadyCondition *metav1.Condition) error {
	decisionTimestamp := metav1.Now()
	clusterHealth := c.clusterHealthMap.getDeepCopy(cluster.Name)
	if clusterHealth == nil {
		return fmt.Errorf("health data doesn't exist for cluster %q", cluster.Name)
	}
	// Check eviction timeout against decisionTimestamp
	switch observedReadyCondition.Status {
	case metav1.ConditionFalse:
		if features.FeatureGate.Enabled(features.Failover) && decisionTimestamp.After(clusterHealth.readyTransitionTimestamp.Add(c.FailoverEvictionTimeout)) {
			// We want to update the taint straight away if Cluster is already tainted with the UnreachableTaint
			// 如果开启了Failover，并且检测集群故障时间已经超过了FailoverEvictionTimeout，则打上cluster.karmada.io/not-ready:NoExecute污点
			// 并移除cluster.karmada.io/unreachable:NoExecute污点，这么做的原因是用哪种污点，取决于集群的状态。
			taintToAdd := *NotReadyTaintTemplate
			if err := utilhelper.UpdateClusterControllerTaint(ctx, c.Client, []*corev1.Taint{&taintToAdd}, []*corev1.Taint{UnreachableTaintTemplate}, cluster); err != nil {
				klog.ErrorS(err, "Failed to instantly update UnreachableTaint to NotReadyTaint, will try again in the next cycle.", "cluster", cluster.Name)
			}
		}
	case metav1.ConditionUnknown:
		if features.FeatureGate.Enabled(features.Failover) && decisionTimestamp.After(clusterHealth.probeTimestamp.Add(c.FailoverEvictionTimeout)) {
			// We want to update the taint straight away if Cluster is already tainted with the UnreachableTaint
			// 如果开启了Failover，并且检测集群故障时间已经超过了FailoverEvictionTimeout，则打上cluster.karmada.io/unreachable:NoExecute污点
			// 并移除cluster.karmada.io/not-ready:NoExecute污点
			taintToAdd := *UnreachableTaintTemplate
			if err := utilhelper.UpdateClusterControllerTaint(ctx, c.Client, []*corev1.Taint{&taintToAdd}, []*corev1.Taint{NotReadyTaintTemplate}, cluster); err != nil {
				klog.ErrorS(err, "Failed to instantly swap NotReadyTaint to UnreachableTaint, will try again in the next cycle.", "cluster", cluster.Name)
			}
		}
	case metav1.ConditionTrue:
		// 如果集群健康了，就移除这两种用于故障转移的taint
		if err := utilhelper.UpdateClusterControllerTaint(ctx, c.Client, nil, []*corev1.Taint{NotReadyTaintTemplate, UnreachableTaintTemplate}, cluster); err != nil {
			klog.ErrorS(err, "Failed to remove taints from cluster, will retry in next iteration.", "cluster", cluster.Name)
		}
	}
	return nil
}
```

### Taint-Manager-Controller

> 代码在`pkg/controllers/cluster/taint_manager.go`

这个Controller主要负责从`ResourceBinding`和`ClusterResourceBinding`对象上移除打了`NoExecute`污点且不再被`PP/CPP`容忍的集群。如果开启优雅驱逐功能，还会将其移动到`PP/CPP`的`优雅驱逐任务队列`中

**工作流程如下：**

- 持续监听Cluster对象上是否出现`NoExecute`类型的污点，会处理有这类污点的集群。
- 找到所有指定了该Cluster的`RB/CRB`，检查对应`PP/CPP`上是否会容忍该集群上的污点。
- 只要有一个污点不被容忍或容忍时间到期，就需要从`RB/CRB`指派集群中删除该Cluster；如果容忍所有污点，则计算容忍剩余的最短时间，将该`RB/CRB`重新入队，等待下次处理。
- 如果开启了优雅迁移，会在`RB/CRB`中删除该Cluster的同时，将这次驱逐事件加入到`优雅驱逐任务队列中`，交给`Graceful-Eviction-Controller`执行最终的驱逐。

调协对象是`Cluster`。

```golang
func (tc *NoExecuteTaintManager) Reconcile(ctx context.Context, req reconcile.Request) (reconcile.Result, error) {
	klog.V(4).Infof("Reconciling cluster %s for taint manager", req.NamespacedName.Name)

	cluster := &clusterv1alpha1.Cluster{}
	if err := tc.Client.Get(ctx, req.NamespacedName, cluster); err != nil {
		// The resource may no longer exist, in which case we stop processing.
		if apierrors.IsNotFound(err) {
			return controllerruntime.Result{}, nil
		}
		return controllerruntime.Result{Requeue: true}, err
	}

	// Check whether the target cluster has no execute taints.
	// 检查当前集群是否有NoExecute类型的Taint，如果没有则不需要处理该集群
	if !helper.HasNoExecuteTaints(cluster.Spec.Taints) {
		return controllerruntime.Result{}, nil
	}
    
 // 集群上有污点，则进行同步
	return tc.syncCluster(ctx, cluster)
}

func (tc *NoExecuteTaintManager) syncCluster(ctx context.Context, cluster *clusterv1alpha1.Cluster) (reconcile.Result, error) {
	// List all ResourceBindings which are assigned to this cluster.
	// 找出所有将工作负载分发给了这个集群的ResourceBinding
	rbList := &workv1alpha2.ResourceBindingList{}
	if err := tc.List(ctx, rbList, client.MatchingFieldsSelector{
	 // 注意此处的索引机制是在cmd\controller-manager\app\controllermanager.go中
	 // startClusterController时注册的，只有开启taint-manager-controller时启用
		Selector: fields.OneTermEqualSelector(rbClusterKeyIndex, cluster.Name),
	}); err != nil {
		klog.ErrorS(err, "Failed to list ResourceBindings", "cluster", cluster.Name)
		return controllerruntime.Result{Requeue: true}, err
	}
	for i := range rbList.Items {
		key, err := keys.FederatedKeyFunc(cluster.Name, &rbList.Items[i])
		if err != nil {
			klog.Warningf("Failed to generate key for obj: %s", rbList.Items[i].GetObjectKind().GroupVersionKind())
			continue
		}
		// 将对应的rb交给驱逐worker，进行异步处理
		tc.bindingEvictionWorker.Add(key)
	}

	// List all ClusterResourceBindings which are assigned to this cluster.
	......
	return controllerruntime.Result{RequeueAfter: tc.ClusterTaintEvictionRetryFrequency}, nil
}

// Start starts an asynchronous loop that handle evictions.
func (tc *NoExecuteTaintManager) Start(ctx context.Context) error {
	bindingEvictionWorkerOptions := util.Options{
		Name:          "binding-eviction",
		KeyFunc:       nil,
		ReconcileFunc: tc.syncBindingEviction,
	}
	// 此处创建并启动了异步work，默认并发数为3
	tc.bindingEvictionWorker = util.NewAsyncWorker(bindingEvictionWorkerOptions)
	tc.bindingEvictionWorker.Run(tc.ConcurrentReconciles, ctx.Done())
	......

	<-ctx.Done()
	return nil
}

func (tc *NoExecuteTaintManager) syncBindingEviction(key util.QueueKey) error {
	// 此处的FederatedKey中，Cluster与ResourceBinding是一对一的关系
	fedKey, ok := key.(keys.FederatedKey)
	if !ok {
		klog.Errorf("Failed to sync binding eviction as invalid key: %v", key)
		return fmt.Errorf("invalid key")
	}
	cluster := fedKey.Cluster

	binding := &workv1alpha2.ResourceBinding{}
	if err := tc.Client.Get(context.TODO(), types.NamespacedName{Namespace: fedKey.Namespace, Name: fedKey.Name}, binding); err != nil {
		// The resource no longer exist, in which case we stop processing.
		if apierrors.IsNotFound(err) {
			return nil
		}
		return fmt.Errorf("failed to get binding %s: %v", fedKey.NamespaceKey(), err)
	}

	if !binding.DeletionTimestamp.IsZero() || !binding.Spec.TargetContains(cluster) {
		return nil
	}
	// 检查需要从该ResourceBinding中驱逐该集群，主要通过ResourceBinding的Annotation中存有相关联的PP
	// 如果集群上的污点都不被容忍，则needEviction为True，否则计算该PP容忍集群上污点的最短时间，
	// 因为打NoExecute污点时，会附带打污点的时间，所以可以计算出容忍还剩下的最短时间
	needEviction, tolerationTime, err := tc.needEviction(cluster, binding.Annotations)
	if err != nil {
		klog.ErrorS(err, "Failed to check if binding needs eviction", "binding", fedKey.ClusterWideKey.NamespaceKey())
		return err
	}

	// Case 1: Need eviction now.
	// Case 2: Need eviction after toleration time. If time is up, do eviction right now.
	// Case 3: Tolerate forever, we do nothing.
	if needEviction || tolerationTime == 0 {
		// update final result to evict the target cluster
		if features.FeatureGate.Enabled(features.GracefulEviction) {
			// 如果开启了优雅驱逐，将该驱逐任务加入到RB的驱逐任务队列中，等待被优雅驱逐
			binding.Spec.GracefulEvictCluster(cluster, workv1alpha2.EvictionProducerTaintManager, workv1alpha2.EvictionReasonTaintUntolerated, "")
		} else {
			// 否则，直接进行驱逐
			binding.Spec.RemoveCluster(cluster)
		}
		// 更新RB，使其生效
		if err = tc.Update(context.TODO(), binding); err != nil {
			helper.EmitClusterEvictionEventForResourceBinding(binding, cluster, tc.EventRecorder, err)
			klog.ErrorS(err, "Failed to update binding", "binding", klog.KObj(binding))
			return err
		}
		// 如果没有开启优雅驱逐，驱逐会立即生效
		if !features.FeatureGate.Enabled(features.GracefulEviction) {
			helper.EmitClusterEvictionEventForResourceBinding(binding, cluster, tc.EventRecorder, nil)
		}
	} else if tolerationTime > 0 {
		// 如果还可以继续容忍，则将其加入到work延迟队列中，等待下一次处理
		tc.bindingEvictionWorker.AddAfter(fedKey, tolerationTime)
	}

	return nil
}
```
以上代码主要分析了`Taint-Manager-Controller`对`ResourceBinding`资源的操作，对于`ClusterResourceBinding`也有类似的流程。

### Graceful-Eviction-Controller

> 代码在`pkg/controllers/gracefuleviction/rb_graceful_eviction_controller.go`和`pkg/controllers/gracefuleviction/crb_graceful_eviction_controller.go`，以下主要分析RB的优雅驱逐，CRB具有相似的逻辑。

这个Controller主要根据`ResourceBinding`和`ClusterResourceBinding`对象上的`GracefulEvictionTasks`字段，优雅地驱逐集群（**集群不一定是故障，也可能是用户手动打污点执行的驱逐操作**）上的工作负载，避免立即删除原先的工作负载而新的工作负载还未就绪，导致服务不可用。

**工作流程如下：**

- 扫描`RB`对象的`GracefulEvictionTasks`队列，如果没有任务已超过`GracefulEvictionTimeout`并且这些任务的重调度工作负载都还未正常就绪，则直接不处理，等待下一次重试（重试间隔时间的计算公式为：**min(GracefulEvictionTimeout/10, 最短任务过期的时间)**）
- 有驱逐任务需要处理，将该任务从`GracefulEvictionTasks`列表里删除，发送驱逐成功的Event，这个Event会被其他组件使用，以真正地驱逐对应集群上的工作负载。

调协对象是`ResourceBinding`

```golang
func (c *RBGracefulEvictionController) Reconcile(ctx context.Context, req controllerruntime.Request) (controllerruntime.Result, error) {
	klog.V(4).Infof("Reconciling ResourceBinding %s.", req.NamespacedName.String())

	binding := &workv1alpha2.ResourceBinding{}
	......
	// 同步RB，并记录下一次重试的时间
	retryDuration, err := c.syncBinding(binding)
	if err != nil {
		return controllerruntime.Result{Requeue: true}, err
	}
	// 如果优雅驱逐任务的时间还没到，会在下一次进行重试
	if retryDuration > 0 {
		klog.V(4).Infof("Retry to evict task after %v minutes.", retryDuration.Minutes())
		return controllerruntime.Result{RequeueAfter: retryDuration}, nil
	}
	return controllerruntime.Result{}, nil
}

func (c *RBGracefulEvictionController) syncBinding(binding *workv1alpha2.ResourceBinding) (time.Duration, error) {
	// 检查驱逐任务，如果任务入队已超过GracefulEvictionTimeout，则开始驱逐该任务，
	// 如果重调度的资源全部都处于健康状态，则也开始驱逐该任务，否则该任务继续保留在队列中
	keptTask, evictedCluster := assessEvictionTasks(binding.Spec, binding.Status.AggregatedStatus, c.GracefulEvictionTimeout, metav1.Now())
	// 如果这次没有要执行的驱逐任务，则在较短的时间内进行下一次重试（计算工时为，min(GracefulEvictionTimeout/10, 离到期最短的任务)）
	if reflect.DeepEqual(binding.Spec.GracefulEvictionTasks, keptTask) {
		return nextRetry(keptTask, c.GracefulEvictionTimeout, metav1.Now().Time), nil
	}

	// 修改RB对象的GracefulEvictionTasks字段
	objPatch := client.MergeFrom(binding)
	modifiedObj := binding.DeepCopy()
	modifiedObj.Spec.GracefulEvictionTasks = keptTask
	err := c.Client.Patch(context.TODO(), modifiedObj, objPatch)
	if err != nil {
		return 0, err
	}

	for _, cluster := range evictedCluster {
		// 发出驱逐成功Event
		helper.EmitClusterEvictionEventForResourceBinding(binding, cluster, c.EventRecorder, err)
	}
	return nextRetry(keptTask, c.GracefulEvictionTimeout, metav1.Now().Time), nil
}

// 这里进行了一些事件过滤，以只处理必要的RB事件
func (c *RBGracefulEvictionController) SetupWithManager(mgr controllerruntime.Manager) error {
	resourceBindingPredicateFn := predicate.Funcs{
		CreateFunc: func(createEvent event.CreateEvent) bool { return false },
		UpdateFunc: func(updateEvent event.UpdateEvent) bool {
			newObj := updateEvent.ObjectNew.(*workv1alpha2.ResourceBinding)
			// 只监听RB更新事件，如果RB的GracefulEvictionTasks为空，则不进行处理
			if len(newObj.Spec.GracefulEvictionTasks) == 0 {
				return false
			}
			// 只监听调度器已经重新调度过的RB
			return newObj.Status.SchedulerObservedGeneration == newObj.Generation
		},
		DeleteFunc:  func(deleteEvent event.DeleteEvent) bool { return false },
		GenericFunc: func(genericEvent event.GenericEvent) bool { return false },
	}

	return controllerruntime.NewControllerManagedBy(mgr).
		For(&workv1alpha2.ResourceBinding{}, builder.WithPredicates(resourceBindingPredicateFn)).
		WithOptions(controller.Options{RateLimiter: ratelimiterflag.DefaultControllerRateLimiter(c.RateLimiterOptions)}).
		Complete(c)
}
```

### Binding-Controller

> 代码在`pkg/controllers/binding/binding_controller.go`和`pkg/controllers/binding/cluster_resource_binding_controller.go`。

这个Controller涉及到很多的工作，这里与`优雅驱逐`有关的是，这里才是真正清除工作负载的地方。

调协对象是`RB/CRB`。

**工作流程如下：**

- `ResourceBindng`对象会关联到分发到每个成员集群的`Work`对象
- 调协`ResourceBindng`时，**会清除关联了该RB但是不符合其分发策略的`Work`对象**。
- 对于`GracefulEvictionTasks`不为空的RB，会保留分发给这些Cluster的`Work`对象。

```golang
func (c *ResourceBindingController) Reconcile(ctx context.Context, req controllerruntime.Request) (controllerruntime.Result, error) {
	klog.V(4).Infof("Reconciling ResourceBinding %s.", req.NamespacedName.String())

	binding := &workv1alpha2.ResourceBinding{}
	......

	return c.syncBinding(binding)
}

func (c *ResourceBindingController) syncBinding(binding *workv1alpha2.ResourceBinding) (controllerruntime.Result, error) {
    // 移除不符合当前分发策略的Work对象
	if err := c.removeOrphanWorks(binding); err != nil {
		return controllerruntime.Result{Requeue: true}, err
	}
	......
}

func (c *ResourceBindingController) removeOrphanWorks(binding *workv1alpha2.ResourceBinding) error {
	// 筛选出来不符合在当前调度集群且不在优雅驱逐任务列表集群中的Wrok
	works, err := helper.FindOrphanWorks(c.Client, binding.Namespace, binding.Name, helper.ObtainBindingSpecExistingClusters(binding.Spec))
	......
	return nil
}
```

`pkg/util/helper/binding.go`

```golang
// ObtainBindingSpecExistingClusters will obtain the cluster slice existing in the binding's spec field.
func ObtainBindingSpecExistingClusters(bindingSpec workv1alpha2.ResourceBindingSpec) sets.Set[string] {
	clusterNames := util.ConvertToClusterNames(bindingSpec.Clusters)
	for _, binding := range bindingSpec.RequiredBy {
		for _, targetCluster := range binding.Clusters {
			clusterNames.Insert(targetCluster.Name)
		}
	}
	// 关键所在，将GracefulEvictionTasks中的集群插入到被集群名集合中
	// 后续的过滤，会保留待驱逐集群上的Work
	for _, task := range bindingSpec.GracefulEvictionTasks {
		clusterNames.Insert(task.FromCluster)
	}

	return clusterNames
}
```

对于`Cluster-Resource-Binding-Controller`有类似的逻辑，在此不赘述。

## 参考

[Karmada Failover Analysis](https://karmada.io/docs/userguide/failover/failover-analysis/)
