软件系统中模块模型：
f(state_{pre},input)=(state_{post},output_{observable})

系统状态由 5 维间接输入组成:
State = (State_{data}, State_{env}, State_{time}, State_{execution}, State_{dependency})

1)state_{data}数据状态: 内存可变数据、持久化存储、运行时缓存、文件系统(路径/内容/权限)、SUT加载配置  
2)state_{env}运行环境持有的数据状态: env环境变量、env配置、系统资源配额、系统进程状态(pid,env,工作目录,user,FDs,…)、runtime GC/调度状态
3)state_{time}: 系统时钟、定时器规则及状态、deadline规则及状态, 其他状态有可能是state_{time}的函数
4)state_{execution}执行状态: 线程/Goroutine 调度顺序、竞态时序、锁与通道同步状态  
5)state_{dependency}外部依赖持有的数据状态: RPC/HTTP/数据库等 外部服务的系统状态、网络状态; 数据响应是state_{dependency}的函数
6)input: 包含使state_{pre}变化的外部信号

LLD Core Principles:

Domain State定义: 每个业务环节的输入/输出/副作用/业务拒绝 的Domain State/类型 都要定义
1)不可变输入状态Plain Struct:
Domain State使用无方法的公有字段结构体表示, 状态实例通过状态提取器 或Parser或构造守卫生产, 状态创建后不可原地修改
2)跨时间不变的身份标识与 状态快照解耦:
业务数据Entity中, identity字段适合作为主键; 业务字段是 时间序列函数; 一个identity实例对应一个Entity, 一个包含所有业务字段的实例是Entity在特定时间的状态快照
3)组合优于继承:
>不使用继承建模业务领域类型层级
>使用组合表达对象结构关系
>使用接口表达可替换的行为契约
>使用显式委托实现行为复用
4)通用接口设计:
接口名称和参数 基于自身机制定义, 只反映当前接口能做什么; 不用上层业务词汇, 不反映上层调用方打算用它做什么

状态构造:
1)状态提取器 或Parser或构造守卫 组装新的不可变状态快照时, 通过结构共享与写时复制降低复制开销:
>Structural Sharing: 新旧快照共享不在修改路径上的节点内存指针, 变化节点修改路径上的节点按顺序进行浅拷贝并重新连接
>Copy-On-Write:
2)构造守卫 方便构造收窄积类型状态空间的Result:
>func Accept(val NextState, cmds []SideEffectCmd){ return Result{Val: val; Cmd: cmds; RejectReason: nil;}}
>func Reject(val NextState, reason Rejection){ return Result{Val: val; Cmd: nil; RejectReason: reason;} }

状态读取: 状态读取与状态变换pure core解耦
1)依赖显式化: 收集state_{pre}中的所有隐式输入通过参数显式传递, 禁止在pure core中隐式读取
2)用状态读取器隔离隐式副作用操作
3)状态读取器用 Plain Struct约束 组装出的不可变状态快照: 对于提取的内存引用, 深拷贝组装出新的内存快照再输入pure core

Parser:
1)含Validate逻辑的input Parser: 若领域状态存在非法状态, 使用带校验逻辑的解析器构造领域状态类型
2)代数类型建模使非法状态不可表示: 声明一个领域状态和类型; 同时声明所有要表达的领域状态子类型; 写一个parser判断传入状态是哪种要表达的领域状态类型, 并创建快照
3)集合类型(引用类型)都支持表达空集(如nil); 泛型非空断言 在编译期 保证领域状态为非空集: type NonEmpty[T any] struct { Head T, Tail []T }
4)聚合解析而非短路解析: Parser中的校验逻辑写成收集全部字段失败信息到[]errs后一次性返回

纯函数式状态变换pure core:
1)决策与副作用执行解耦: pure core只负责业务决策, 副作用业务执行由返回的外部调用方执行, 或在事件驱动模型中通知调度器执行
2)给定输入得到确定性输出
3)显式副作用: 返回值必须包含 (NextState, []SideEffectCmd, Rejection); 决议结果类型Type Rejection struct{ Kind RejectionKind; Reason string; Payload any}用于调用方直接判断结果, RejectionKind是业务拒绝枚举类型, 拒绝上下文Payload传递业务拒绝参数; 输出副作用Type SideEffectCmd struct{ Kind CommandKind; Payload any; IdempotencyKey string; Compensate *SideEffectCmd; Guarantee ExecGuarantee }的指令类型Kind最好是枚举值, 而非可执行代码/闭包/回调; Payload是指令执行上下文; Guarantee标志命令失败时，Imperative Shell应采取的措施; IdempotencyKey是SideEffectCmd实例唯一身份凭证; Compensate是失败时的回滚指令
4)显式输出状态的包装类型: Type Result[NextState any, SideEffectCmd any, Rejection any]{ Val NextState; Cmd []SideEffectCmd; RejectReason Rejection}; Result不包含错误字段, pure core输出业务拒绝RejectReason字段, 运行期错误在pure core外抛出并由Imperative Shell处理

SideEffect Executor:
1)定义副作用执行守卫, 副作用类型Type SideEffectCmd struct{ Kind CommandKind; Payload any; IdempotencyKey string; Compensate *SideEffectCmd; Guarantee ExecGuarantee }定义失败策略, 回滚命令, 幂等性等字段; 前置逻辑校验幂等性; 后置逻辑进行失败处理; 校验幂等性时保存副作用状态快照

编排执行模块Imperative Shell设计:
1)深模块: Imperative Shell层尽量只提供 仅暴露功能接口 的深模块, 隐藏 状态读取和pure core生产的中间状态; 将 读算写 编排、异常状态守卫、缺省前置配置等逻辑沉入模块中
2)编译期保证的 前置状态时序约束: StepB函数签名声明StepA才能生产的前置状态类型, 由此 保证时序
3)状态防重用或幂等性: 缓存StepA生产的旧快照, 用于多次StepB消费; 非线性类型系统语言无法编译期保证; 只能把 StepA与StepB的编排都藏在深模块中, 让外界无法获得中间状态, 同时中间状态也不导出包外; 对于StepA, StepB跨进程调用的情况, 设置version字段判断是否收到旧 Version 的消费请求
4)不可变快照 + 纯函数的并发模型: 并发读→pure core计算→原子写
5)Imperative Shell层尽量提供严格遵循(读→算→写)三段编排的深模块功能函数: 按时序进行过程式的功能逻辑编排; 收敛回严格三段式:
>写成一个三段式循环
>预读取: 对于条件读取, 可忽略判断，在首轮统一并发读取所有可能用到的状态
>指令缓冲与批处理: 所有写操作在最后一轮循环统一并发执行

横向Principles:
状态机尽量剪枝, 降低复杂度; 业务状态空间外的异常状态不一定是业务上不允许的状态; 可定义为正常状态:
1)输入相同状态 重复执行: no-op; 幂等化
2)Normalization: 某些异常状态可映射到合法状态空间
3)Clamping边界收敛: 越界状态映射为边界状态min(right, max(a, left))
4)Defaulting: 所有未定义异常状态用Default branch