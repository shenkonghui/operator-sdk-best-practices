# api设计与operator数据结构
先来参考一下deployment的设计
在目录 k8s.io/api/apps/v1k8s.io/api/apps/v1下面
```
// DeploymentSpec is the specification of the desired behavior of the Deployment.
type DeploymentSpec struct {
	
	Replicas *int32 `json:"replicas,omitempty" protobuf:"varint,1,opt,name=replicas"`
	
	Selector *metav1.LabelSelector `json:"selector" protobuf:"bytes,2,opt,name=selector"`

	Template v1.PodTemplateSpec `json:"template" protobuf:"bytes,3,opt,name=template"`

	Strategy DeploymentStrategy `json:"strategy,omitempty" patchStrategy:"retainKeys" protobuf:"bytes,4,opt,name=strategy"`

	MinReadySeconds int32 `json:"minReadySeconds,omitempty" protobuf:"varint,5,opt,name=minReadySeconds"`

	RevisionHistoryLimit *int32 `json:"revisionHistoryLimit,omitempty" protobuf:"varint,6,opt,name=revisionHistoryLimit"`

	Paused bool `json:"paused,omitempty" protobuf:"varint,7,opt,name=paused"`

	ProgressDeadlineSeconds *int32 `json:"progressDeadlineSeconds,omitempty" protobuf:"varint,9,opt,name=progressDeadlineSeconds"`
}
```

## 数据类型
- string
- int32
- 指针
- 结构体
- *intstr.IntOrString
- metav1.Time
...

不支持并且不建议使用浮点类型
https://github.com/kubernetes-sigs/controller-tools/issues/245

#### 整形字符串复合型 Intstr.IntOrString
来自包 k8s.io/apimachinery/pkg/util/intstr
结构如下
```
type IntOrString struct {
	Type   Type   `protobuf:"varint,1,opt,name=type,casttype=Type"`
	IntVal int32  `protobuf:"varint,2,opt,name=intVal"`
	StrVal string `protobuf:"bytes,3,opt,name=strVal"`
}
```

如果type = 0 ，在转json时候，取取整形，
如果type = 1， 那么取字符串类型

使用场景：适用某些既可以使用整形也可以使用字符串的场景
比如service中的targetPort，即可以写具体的端口号，也可以写字符串进行自动配置

我这里实现了将资源类型转化
```
type ServicePort struct {
...
	TargetPort intstr.IntOrString `json:"targetPort,omitempty" protobuf:"bytes,4,opt,name=targetPort"`
	..
}
```

我这里用它来记录时间，比如2m、600等等
```
func GeIntStrTimeToInt(input intstr.IntOrString)  time.Duration {
	if input.Type == 0 {
		return time.Duration(input.IntValue()) * time.Second
	}else{
		str := strings.ToUpper(input.String())
		output := 2 * 60
		var err error
		if len(str) > 0{
			switch str[len(str)-1] {
			case 'S':
				output, err = strconv.Atoi(str[0 :len(str)-1])
				break
			case 'M':
				output, err = strconv.Atoi(str[0 :len(str)-1])
				output = output * 60
				break
			case 'H':
				output, err = strconv.Atoi(str[0 :len(str)-1])
				output = output * 60 * 60
				break
			default:
				output = 2 * 60
			}
		}
		if err != nil {
			output = 2 * 60
		}
		return time.Duration(output) * time.Second
	}
}
```

#### 资源类型 resource.Quantity  
来自包k8s.io/apimachinery/pkg/api/resource
结构如下
```
type Quantity struct {
	// i is the quantity in int64 scaled form, if d.Dec == nil
	i int64Amount
	// d is the quantity in inf.Dec form if d.Dec != nil
	d infDecAmount
	// s is the generated value of this quantity to avoid recalculation
	s string

	// Change Format at will. See the comment for Canonicalize for
	// more details.
	Format
}
```
使用场景：适合某些使用资源的场景，需要定义字符串、整形、浮点型等等
常见方法
- 字符串转化成Quantity类型
resource.MustParse(val)
- 

#### json protobuf
protocol buffers 是一种语言无关、平台无关、可扩展的序列化结构数据的方法，它可用于（数据）通信协议、数据存储等
```
RevisionHistoryLimit *int32 `json:"revisionHistoryLimit,omitempty" protobuf:"varint,6,opt,name=revisionHistoryLimit"`
```
以上的意思是
- omitempty
忽略掉空的
- protobuf:"varint,6,opt,name=revisionHistoryLimit"
初始化变量为6,Varints代表4个字节


## 问题
#### 时间使用了metav1.Time类型，  更新错误
以下代码，提示nil，但是debug看了一下是有值的
```
	err = r.Client.Update(context.TODO(),&rqa,&client.UpdateOptions{})
	if err != nil {
		log.Error(err, "update rqa fail")
	}
```
2020-06-01 11:11:44 ERROR: ResourceQuotaAutoscaler.harmonycloud.cn "example-resourcequotaautoscaler" is invalid: [spec.status.lastExpanTimeScale: Invalid value: "null": spec.status.lastExpanTimeScale in body must be of type string: "null", spec.status.lastShrinkTimeScale: Invalid value: "null": spec.status.lastShrinkTimeScale in body must be of type string: "null"]update rqa fail




解答：

https://github.com/kubernetes/kube-openapi/tree/master/pkg/generators
需要使用openapi，定义自定义格式转化 ，具体不知道如何实现，暂时先使用string类型代替