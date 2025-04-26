## 一、pipeline概念

一个大任务被拆分为多个小任务，像流水线一样，上一级处理后将数据传递给下一级。



## 1、1 基本组成

一个成熟的pipeline构成如下：

- 1.**输入源(Source)**

网络流量、文件数据、消息队列数据、实时流数据等

- **2.处理阶段(Stage)**

多个组合，每个stage可能存在多个并发处理器(Processor)

- **3.输出(Sink)**

可指定不同的输出位置(命令行/日志/kakfa队列等) 也称为消费者



## 1、2 常见使用场景

![Golang pipeline](https://www.druva.com/adobe/dynamicmedia/deliver/dm-aid--771d4fef-ee2a-4129-9120-5dae89c66299/blog-image1-2.png?preferwebp=true&quality=85)

所有的stage都在同一个go 协程中运行。

![Golang stages](https://www.druva.com/adobe/dynamicmedia/deliver/dm-aid--dab9cfd7-d1b6-4974-98f1-f53c8dd39b38/blog-image2-1.png?preferwebp=true&quality=85)

为了增加处理效率，可能会多个协程来处理数据，每个协程中都包含一个pipeline，pipeline中包含完整的Input,Stage1,Stage2,Stage3,Stage4,Output。

如果我们能让每个阶段都有多个实例，并且这些实例能够独立运行且相互协作，会怎么样？

![Simultaneous stage interaction](https://www.druva.com/adobe/dynamicmedia/deliver/dm-aid--e2fdd1eb-8b91-452f-935c-87bc327ef08c/blog-image3.png?preferwebp=true&quality=85)



传统模式： 一个人从头到尾负责一份面 

**烧水->下面->煮面->调味** 

新模式： 专人专岗，每道工序都有专门的人 

- 烧水师傅专门负责烧水
- 下面师傅专门负责下面
- 煮面师傅专门负责煮面
- 调味师傅专门负责调味

好处：

- 提升并行度，多个阶段的任务各不相同，可同时处理；

- 每个阶段无需等待整个流程完毕，就可以开始下一轮任务；

- 避免资源空闲，提升资源利用率
- 能够对各个阶段进行独立的扩展、配置和执行



参考链接：

https://www.druva.com/blog/concurrent-and-efficient-pipelines-using-golang-channels



# 二、同步pipeline vs 异步pipeline 使用场景举例

根据不同的任务类型分为两种pipeline：

- **同步pipeline**
  - 场景：多个stage阶段必须在**同一个协程中有序进行**，前一个stage阶段处理器的输出结果，是作为输入参数传递给下一个stage阶段的处理器的
- **异步pipeline**
  - 多个stage阶段可以在不同协程中并行执行，互不阻塞；
  - 前一个stage的结果通过channel异步传递给下一个stage；
  - stage间处理速度不一致时，需要进行解耦



对于异步pipeline中处理速度不一致时的情况，举个例子：

**想象一个视频处理Pipeline：**

1. Stage1: 读取视频文件 (很快，1秒读一个文件)
2. Stage2: 视频转码 (很慢，10秒处理一个文件)

**同步处理（性能不好）：**

- Stage1读完要等Stage2处理完才能继续
- 读取能力被浪费了
- 整体处理变慢

**使用异步Pipeline后：**

- Stage1可以持续读取，把文件放入channel缓冲区
- Stage2按自己节奏处理（对于耗时操作，可考虑多开启协程加快处理速度），不会阻塞Stage1
- channel起到"蓄水池"作用，平衡两边速度差



# 三、同步pipeline最佳实践

## 3、0 同步pipeline背景

![image-20250218090827654](https://gitee.com/codergeek/picgo-image/raw/master/image/202502180908498.png)

对输入的文章，如下处理

1. 按行分割后按照空格分割成每个单词（对应处理器1）
2. 统计每个单词出现的次数（对应处理器2）
3. 按照次数排序（对应处理器3）
4. 输出出现次数最多的前三单词（对应Sink）



## 3、1 面向对象封装

使用接口来标准化，更容易模块化，整个程序完全解耦，输入/输出/错误处理可以实现多种，各个Processor都可以替换。

### **1）数据源接口**

```
type ISource interface {
	Process(ctx context.Context) (<-chan any, error)
}
```



### 2）输入源的实现代码

```
type TimerSource struct {
    data   string
    ticker *time.Ticker
    buffer int
}

func NewTimerSource(data string, bufferSize int) ISource {
    return &TimerSource{
        data:   data,
        buffer: bufferSize,
    }
}

func (s *TimerSource) Process(ctx context.Context) (<-chan any, error) {
    if s.data == "" {
        return nil, errors.New("empty data source")
    }
    
    out := make(chan any, s.buffer)
    s.ticker = time.NewTicker(time.Second)//开启定时器，一秒钟读取一次数据
    
    go func() {
        defer func() {
            s.ticker.Stop()
            close(out)
        }()
        
        r := bufio.NewReader(strings.NewReader(s.data))
        
        for {
            select {
            case <-s.ticker.C:
                input, err := io.ReadAll(r)
                if err != nil {
                    log.Printf("Read error: %v", err)
                    return
                }
                
                // 内层select添加default，当channel满时直接丢弃数据
                select {
                case out <- string(input):
                    // 发送成功
                case <-ctx.Done():
                    return
                default:
                    log.Printf("Channel is full, dropping message")
                }
                
            case <-ctx.Done():
                log.Println("Source received exit signal")
                return
            }
        }
    }()
    
    return out, nil
}
```



### **3）输出接口**

```
type ISink interface {
	Process(ctx context.Context, params any) error
}
```

### 4）输出源的实现代码

```
type ConsoleSink struct {
}

func NewConsoleSink() ISink {
	return &ConsoleSink{}
}

func (s *ConsoleSink) Process(ctx context.Context, params any) error {
	if v, ok := params.([]struct {
		cnt  int
		word string
	}); !ok {
		return errors.New("Sink Input type error ")
	} else {
		for i := range v {
			fmt.Printf("单词:%s - 出现次数:%d\n", v[i].word, v[i].cnt)
		}

		return nil
	}
}
```



### 5）**处理器接口**

```
type IProcessor interface {
	Process(ctx context.Context, params any) (any, error)
}
```

**小伙伴可能有疑问：**

ISource接口定义的Process函数的输出参数类型为<-chan any?

ISink和IProcessor接口定义的Process函数的输入参数类型为params any？

答：将输入参数和输出参数都设置为any，是因为输入数据的类型是多变的，输出数据的类型也是多变的，所以类型设置为any，是为了更好的适配不同类型的数据。

注意这里输入输出使用any来完成一定的泛化，下游接受上游数据时，最好做下类型断言



### 6）处理器的实现代码

**按行分割处理器**

```
type SplitProcessor struct{}

//此处返回值是个[]string
func (p *SplitProcessor) Process(ctx context.Context, params any) (any, error) {
	//校验是否为字符串类型
	if v, ok := params.(string); !ok {
		return nil, errors.New("Split Input type error ")
	} else {
		return strings.Split(v, "\n"), nil
	}
}
```



**统计每个单词数量处理器**

```
// CountProcessor 拆分单词并统计词频
type CountProcessor struct{}

func (p *CountProcessor) Process(ctx context.Context, params any) (any, error) {
	if v, ok := params.([]string); !ok {
		return nil, errors.New("Count Input type error ")
	} else {
		wordStat := make(map[string]int)

		//遍历切片中元素，使用空格为分隔符来拆分单词，并统计每个单词的频次
		for _, l := range v {
			words := strings.Split(string(l), " ")
			for _, word := range words {
				if v, ok := wordStat[word]; ok {
					wordStat[word] = v + 1
				} else {
					wordStat[word] = 1
				}
			}
		}

		return wordStat, nil
	}
}
```



**倒序排列处理器**

```
// Process 处理单词统计结果，返回出现频率最高的3个单词
// params: map[string]int 单词及其出现次数的映射
// return: []WordCount 按频率排序的前3个单词，error 错误信息
func (p *SortProcessor) Process(ctx context.Context, params any) (any, error) {
    // 类型断言确保输入是 map[string]int
    wordStat, ok := params.(map[string]int)
    if !ok {
        return nil, errors.New("Sort Input type error")
    }
    
    // 预分配切片避免动态扩容带来的性能开销
    wordStatSlice := make([]WordCount, 0, len(wordStat))
    
    // 将map转换为切片以便排序
    for k, v := range wordStat {
        wordStatSlice = append(wordStatSlice, WordCount{cnt: v, word: k})
    }
    
    // 按照单词出现次数降序排序
    sort.Slice(wordStatSlice, func(i, j int) bool {
        return wordStatSlice[i].cnt > wordStatSlice[j].cnt
    })
    
    // 安全返回TOP 3结果
    // 如果结果少于3个，返回全部结果
    if len(wordStatSlice) > 3 {
        return wordStatSlice[:3], nil
    }
    return wordStatSlice, nil
}
```





### 7）pipeline流水线的实现

#### 1、单个并发度

![img](https://gitee.com/codergeek/picgo-image/raw/master/image/202502181012203.png)

上述所有流程都在一个golang goroutine协程中依次执行。

```
// Run 执行整个处理管道，包含数据源读取、处理器链处理、数据落地三个阶段
// 单个并发度
func (pm *ProcessorManager) Run(ctx context.Context) error {
	var err error

	// 第一阶段：从数据源获取数据流
	// in 为一个channel，用于异步接收数据
	in, err := pm.source.Process(ctx)
	if err != nil {
		return err
	}

	// 第二阶段：遍历数据流，执行处理管道
	for data := range in {
		// 依次执行所有处理器
		// 每个处理器的输出作为下一个处理器的输入
		for _, p := range pm.ps {
			data, err = p.Process(ctx, data)
			if err != nil {
				log.Printf("process err %s\n", err)
				//不返回错误，直接处理下一条数据
				//return err
			}
		}

		// 第三阶段：数据落地
		// 将处理后的数据交给sink处理器进行持久化
		err = pm.sink.Process(ctx, data)
		if err != nil {
			log.Printf("Sink err %s\n", err)
			return err
		}
	}
	return nil
}
```



#### 2、多个并发度

![img](https://gitee.com/codergeek/picgo-image/raw/master/image/202502181014133.jpg)

```
// RunN 并发执行处理管道，支持多个goroutine同时处理数据
// ctx: 上下文，用于控制处理流程的生命周期
// maxCnt: 并发处理的goroutine数量
// return: error 初始化过程中的错误信息
func (pm *ProcessorManager) RunN(ctx context.Context, maxCnt int) error {
    // 第一阶段：初始化数据源
    // 获取数据流channel
    in, err := pm.source.Process(ctx)
    if err != nil {
        return err
    }

    // 定义单条数据的处理函数（很推荐这种做法）
    // 包含处理器链执行和数据落地两个步骤
    syncProcess := func(data any) {
        // 依次执行所有处理器
        // 处理器的输出作为下一个处理器的输入
        for _, v := range pm.ps {
            data, err = v.Process(ctx, data)
            if err != nil {
                log.Printf("process err %s\n", err)
                return
            }
        }

        // 数据落地操作
        err := pm.sink.Process(ctx, data)
        if err != nil {
            log.Printf("sink err %s\n", err)
            return
        }
    }

    // 使用WaitGroup控制并发goroutine的生命周期
    wg := sync.WaitGroup{}
    wg.Add(maxCnt)

    // 启动多个goroutine并发处理数据
    // 每个goroutine独立消费数据源channel
    for i := 0; i < maxCnt; i++ {
        go func() {
            defer wg.Done() // 确保goroutine退出时减少计数

            // 循环处理数据，直到channel关闭
            for data := range in {
                syncProcess(data)
            }
        }()
    }

    // 等待所有goroutine完成处理
    wg.Wait()
    return nil
}
```



## 3、2 高阶函数封装

这块还是要好好处理的。



# 四、异步pipeline最佳实践

## 4、0 异步pipeline背景

一个典型的计算任务需求如下图，输入一串数字，先分别计算平方然后计算累加值

1. 生成数据源，投递到channel通道中；
2. 从channel通道中取出数据，计算平方后，投递到下一个channel通道中；
3. 从channel通道中取出数据，计算累加值，然后投递到下一个channel通道中；
4. 输出Sink从channel通道中读取数据，然后进行输出。



**同步pipeline vs 异步pipeline 的不同点是什么？**

不同于同步pipeline，异步pipeline输入输出和各个Stage算子之间的数据传输都是通过channel，每个stage都可以独立控制自己的并发度。

![img](https://gitee.com/codergeek/picgo-image/raw/master/image/202502181509405.jpg)



## 4、1 面向对象封装

### 1）数据源接口

```
type ISource interface {
    Process(ctx context.Context, wg *sync.WaitGroup, errChan chan error) <-chan int
}

```

### 2）数据源的实现代码

```
// TimerSource 定时数据生成器
// Nums 存储要依次输出的数据
type TimerSource struct {
	Nums []int
}

func NewTimerSource(nums ...int) *TimerSource {
	return &TimerSource{Nums: nums}
}

// Process 实现数据源接口
// 每隔1秒生成一个数据并发送到输出通道
// 如果数据小于0，则产生错误并跳过该数据
// 当所有数据都已发送或收到取消信号时退出
func (t *TimerSource) Process(ctx context.Context, wg *sync.WaitGroup, errChan chan error) <-chan int {
	//细节：在go协程的for循环外调用wg.Done()
	defer wg.Done()
	outChannel := make(chan int, 10)

	go func() {
		defer close(outChannel)
		i := 0

		for i < len(t.Nums) {
			select {
			case <-time.After(1 * time.Second):
				s := t.Nums[i]
				i++
				outChannel <- s					// 数据发送成功

			case <-ctx.Done():
				log.Println("Timer source received cancel signal")
				return
			}
		}
		log.Println("Timer source completed normally")
	}()

	return outChannel
}
```



### **3）输出的接口**

```
type ISink interface {
    Process(ctx context.Context, wg *sync.WaitGroup, dataChan <-chan int, errChan chan error)
}
```



### 4）输出Sink的实现代码

```
// ConsoleSink 控制台输出接收器
// 将接收到的数据打印到控制台
type ConsoleSink struct {
}

func NewConsoleSink() *ConsoleSink {
	return &ConsoleSink{}
}

// Process 实现输出器接口
// 从输入通道读取数据并打印到控制台
// 当输入通道关闭或收到取消信号时退出
func (s *ConsoleSink) Process(ctx context.Context, wg *sync.WaitGroup, dataChan <-chan int, errChan chan error) {
	go func() {
		//细节：在go协程的for循环内部调用wg.Done()
		defer wg.Done()

		for {
			select {
			case val, ok := <-dataChan:
				if !ok {
					log.Println("sink data channel closed!")
					return
				}
				fmt.Printf("sink value: %v\n", val)

			case <-ctx.Done():
				// 继续处理通道中的剩余数据
				log.Println("Sink draining remaining data")
				for val := range dataChan {
					fmt.Printf("sink value (draining): %v\n", val)
				}
				log.Println("Sink completed")
				return
			}
		}
	}()
}

```



### 5）**处理器接口**

```
type IProcessor interface {
    Process(ctx context.Context, wg *sync.WaitGroup, dataChan <-chan int, errChan chan error) <-chan int
}
```



### 6） 处理器的实现代码

```
// SqProcessor 平方处理器
// 对输入的数据计算平方值
type SqProcessor struct {
}

// Process 实现处理器接口
// 从输入通道读取数据，计算平方后发送到输出通道
// 当输入通道关闭或收到取消信号时退出
func (s *SqProcessor) Process(ctx context.Context, wg *sync.WaitGroup, dataChan <-chan int, errChan chan error) <-chan int {
	defer wg.Done()
	outChannel := make(chan int)

	go func() {
		defer close(outChannel)
		
		for {
			select {
			// 从输入通道读取数据
			case s, ok := <-dataChan:
				if !ok {
					log.Println("sq data channel closed!")
					return
				}
				// 处理数据并确保发送成功
				result := s * s
				outChannel <- result	// 数据发送成功
				
			case <-ctx.Done():
				// 继续处理输入通道中的剩余数据
				log.Println("Sq processor draining remaining data")
				for s := range dataChan {
					outChannel <- s * s
				}
				log.Println("Sq processor completed")
				return
			}
		}
	}()

	return outChannel
}

// SumProcessor 累加处理器
// 对输入的数据进行累加计算
type SumProcessor struct {
}

// Process 实现处理器接口
// 从输入通道读取数据，与之前的累加和相加后发送到输出通道
// 当输入通道关闭或收到取消信号时退出
func (s *SumProcessor) Process(ctx context.Context, wg *sync.WaitGroup, dataChan <-chan int, errChan chan error) <-chan int {
	defer wg.Done()
	outChannel := make(chan int)

	go func() {
		defer close(outChannel)
		var sum = 0

		for {
			select {
			case s, ok := <-dataChan:
				if !ok {
					log.Println("sum data channel closed!")
					return
				}
				// 处理数据并确保发送成功
				sum += s
				outChannel <- sum // 数据发送成功

			case <-ctx.Done():
				// 继续处理输入通道中的剩余数据
				log.Println("Sum processor draining remaining data")
				for s := range dataChan {
					sum += s
					outChannel <- sum
				}
				log.Println("Sum processor completed")
				return
			}
		}
	}()

	return outChannel
}

```



### 7）**错误处理接口**

```
type IError interface {
    Process(ctx context.Context, wg *sync.WaitGroup, errChan chan error, cancel context.CancelFunc)
}
```

**错误处理实现**

```
func (p *ErrorPolicyExit) Process(ctx context.Context, wg *sync.WaitGroup, errChan chan error, cancel context.CancelFunc) {
    for {
        select {
        case err, ok := <-errChan:
            if !ok {
                log.Println("error channel closed and exit!")
                return
            }
			
			//在error channel上一旦发现错误，cancel通知整个处理pipeline停止
            log.Printf("Receive error %v\n", err)
            cancel()
        }
    }
}
```

### 8） pipeline流水线实现

**对象的定义和方法**

```
type ProcessorManager struct {
	source  ISource
	sink    ISink
	ps      []IProcessor
	err     IError
	errChan chan error
}

func NewProcessorManager() *ProcessorManager {
	return &ProcessorManager{errChan: make(chan error, 2)}
}

func (m *ProcessorManager) AddProcessor(processor IProcessor) {
	m.ps = append(m.ps, processor)
}

func (m *ProcessorManager) AddSource(source ISource) {
	m.source = source
}

func (m *ProcessorManager) AddSink(sink ISink) {
	m.sink = sink
}

func (m *ProcessorManager) AddError(err IError) {
	m.err = err
}
```

**流程编排**

```
func (m *ProcessorManager) Run(ctx context.Context) {
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	var wg = sync.WaitGroup{}

	// 组装pipeline,如何组装这个pipeline的呢
	wg.Add(1)
	dataChan := m.source.Process(ctx, &wg, m.errChan)

	//遍历所有注册的处理器，为每个处理器的函数执行开启一个协程
	for _, v := range m.ps {
		wg.Add(1)
		dataChan = v.Process(ctx, &wg, dataChan, m.errChan)
	}

	wg.Add(1)
	m.sink.Process(ctx, &wg, dataChan, m.errChan)

	go func() {
		wg.Wait()
		//所有处理完成后，关闭err错误通道
		close(m.errChan)
	}()

	// 错误通道内部逻辑是for循环，阻塞，错误处理集中处理
	// 出现错误则调用cancel函数通知退出，可灵活定制处理策略
	m.err.Process(ctx, &wg, m.errChan, cancel)
}
```

注意到一个细节，processor、sink的Process函数中关于wait group的处理是在直接调用Done

# 五、参考文档

上述代码示例在我的github上

https://github.com/haolipeng/go_study/tree/master/go-design-patterns/pipeline



https://zhuanlan.zhihu.com/p/598761593

https://github.com/google/go-pipeline.git

https://medium.com/@gopinathr143/go-concurrency-patterns-a-deep-dive-a2750f98a102

https://gitee.com/wenzhou1219/go-in-prod/blob/master/pipeline/sync_pipe/sync_test.go