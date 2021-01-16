&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;OGG能在多样而复杂的平台的事务级别操作数据，模块化的架构便于抽取和复制抽取的记录、事务的改变、DDL语句，可以多种拓扑结构实现。  
良好的过滤、传送、OGG客户端进程特性可支持多种需求：

- 业务支持上的高可用性
- 初始化和数据库迁移
- 数据集成
- 决策支持和数据仓库  



OGG支持多种拓扑结构：  
![OGG拓扑结构图](http://cdn.lifemini.cn/dbblog/20210115/14fdfe4d2fb64dff95fe7809ef07a786.png)


OGG工作过程：  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;当匹配玩同步进程后，EXTRAXT进程捕获对配置过的对象的DML和DDL操作。EXTRACT进程保存操作直到接收到commit或rollback。当接收到rollback后，不记录此事务操作。当接收到commit后，EXTRSACT持久化事务到硬盘上，以一系列的trail文件命名，并有序的从源端传送到目的端。 

- 所有的事务中,每一个事务的所有操作被以一个有序的事务组织单元写入到trail文件中。   
- 这种设计确保了速度和数据的完整性。多样化的捕获进程能够在同一时间并发操作不同的对象。  
  + 如：数据库比较大为了达到最小延迟目标时，可以两个进程操作一个些对象，另两个进程操另一些对象。还可以给操作EXTRACT进程分组吗，并分配组名。  

##### 注意：EXTRACT进程忽略没有在配置文件中配置对象的相关操作。即使事务中存在部分配置的对象操作，那么未配置的对象的DML操作将不会被捕获到。  



#### OGG架构概览   

##### OGG能够被用来配置以下用途：  

- 抽取一个静态数据记录从一个数据库，和加载这些记录到另一个数据库。
- 持续捕获和复制DML事务操作及DDL语句变化以使两端数据库一致。  
- 从一个数据库捕获并导出到一个数据库之外的数据文件。  

##### OGG包含以下组件：  

![OGG--architecture](http://cdn.lifemini.cn/dbblog/20210115/b3007594202f40e5a66c3d2ad917fabd.png)

- EXTRACT进程
- DATA pump进程
- replicat复制进程
- Trails或extract文件  
- Checkpoints检查点
- Manager管理进程
- Collector收集器   

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;OGG是用来初始化数据和同步DML和DDL数据的逻辑架构。   
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;图片中是OGG的基础配置，可根据业务需求更改相关配置，在此配置上变体。



#### EXTRACT进程概览  

&nbsp;&nbsp;&nbsp;&nbsp;Extract进程运行在源端系统，是OGG的抽取机制。   

- 可通过如下方式配置此进程。  
    - 初始化：Extract直接从源对象捕获当前的静态的数据集。
    - 配置同步机制：保持源数据与另一个数据集合同步，捕获进程捕获DML和DDL操作在初始化同步之后。 
- 捕获进程可以如下方式捕获信息：
    + 如果是初始化加载，捕获的信息来源于原表。
    + 可以来自数据库返回日志或者事务日志（例如Oracle redo logs或SQL/MX审计追踪）。
        + 不同日志文件中捕获的方式依赖于数据库的类型。 
    + 可以来自第三方捕获模块。此种方式提供一个从外部API到抽取API的解析数据和元数据的交流层。  

- 当改变同步配置在捕获配置中，EXTRCT捕获DML和DDL等在对象上执行的操作。  
    + EATRACT接收rollback时，不记录事务中的操作，接收commit时，EXTRACT以一系列被称为trail文件的方式持久化事务到硬盘，trail文件有序的被准备传送到target库。  
    + 所有事务中的操作被组织成事务单元写到trail文件中。这个设计确保了高效性和完成性。  



#### PUMP进程概览  

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;数据泵是源端OGG在EXTRACT配置之后的第二步。如果不使用数据泵进程，EXTRACT进程一定要传送操作数据到目标端的trail文件。在经典的数据泵配置中，建议Extract进程将数据写到源端本地trail文件。数据泵读取并传送trail文件中的数据操作通过网络传送到远程的目的端trail文件。   
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;数据泵增加了存储弹性、隔离了TCP/IP与主捕获进程之间的服务。  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;通常数据泵用来执行数据抽取、映射、转换，或以原模式(pass-through)配置，但pass-through模式的数据是被消极的传送，不掺杂任何对数据的操作。pass-through模式增加了数据泵(PUMP进程)传递的数据量，因为此模式要求target对象和source对象定义一致。

##### 在大多数公司应用场景中，建议使用数据泵。原因如下：  

- ##### 预防网络和目标端崩溃：  
    + 基础OGG配置中仅在目标端有trail文件，源端没有位置存放数据，Extract进程持续捕获事务操作存于内存，一旦网络或目标端不可用，Extract进程可能造成内存溢出或进程崩溃。然而，在源端配置trail和data pump后，捕捉的数据被写入磁盘，防止主要Extract进程崩溃。当连接被重新建立，数据泵从源端的trail文件中抓取并传送数据到目标端系统。
- ##### 能执行数据筛选或事务的少量解析：  
    + 当使用复杂的筛选或数据传送配置时，你可以配置数据泵去执行第一部分传送，既不在源端也不再目标端系统，及时实在一个实时系统上，之后由其他数据泵（Extract DUMP进程）或复制组(Replicat进程)去执行第二阶段传送。
- ##### 从不同源端固化数据到目标中心：  
    + 当在多种源数据库和中心目标库时，你可以在没个源系统上执行捕获数据操作，并使用各系统上的数据泵传送数据到目标端trail文件中。切分源端和目标端的存储负荷，并解决目标端大量空间如何容纳多种数据源数据的需求。
- ##### 同步一个数据源到多个目标端：
    + 当传送数据到目标端系统的时候，你可为每个目标端配置数据泵在对应的源端数据泵。



#### replicat进程概览  

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;replicat进程运行在目标端，负责在系统上读取trail文件，及重构DML或DDL操作并应用到目标端数据库。   

可以配置replicat进程用于如下场景：  

- 初始化：
    + 用于数据初始化，复制进程能应用静态数据复制到目标对象，或以高速、批量加载工具传递数据(impdp/expdp)。  
- 改为同步模式：
    + 更改同步模式配置时，replicat支持那些被复制的源端操作到目标对象使用原生数据库接口或ODBC，这依赖于数据库的类型。  
    + 为了保护数据的完整性，Replicat将已复制的操作以于源端同样顺序应用到目标端，直至源端每个事物单元的commit位置。

你可以使用复杂的Replicat进程同Extract进程以并行的方式增大数据生产量。为了保护数据完整性，每个进程集合提交不同的对象集合。



#### trails文件概览  

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;为了支撑持续捕获与复制数据库的改变，OGG存储捕获的改变临时记录，在磁盘上以一系列trail文件的形式。trail文件能存在源端系统、中间系统、目标端系统或是任何这些系统的子系统，这取决于你如何配置它、在本地系统它被命名为extract trail或是local trail，远端称为remote trail。  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;通过使用trail文件存储，OGG提供准确的数据和数据容错功能。trail的使用也支持对于每一个依赖捕获发生的复制活动。伴随着这些进程被抽离，对于数据是如何被处理和分发的你会有更多的选择。例如，如果一旦捕获和辅助无法持续工作，你可以持续捕获改变但是存储他们在trail文件中。当目标端需要他们时传送到目标端。  

- ##### trail的读和写处理  
    + ##### 写处理trail：
主捕获进程Extract和数据泵Extract向trail中进行写操作。
        - <font color="red">一个Extract进程仅能写到一个trail</font>  
        - <font color="red">每个Extract必须被连接到trail。</font>
    + ##### 读处理trail：
        * 数据泵Extract：从先前被连接捕获进程中捕获DML和DDL操作，如果需要可执行更多操作，并传送数据到被下一个OGG进程读取的下游进程（replicat进程就是最具代表性的，如果需要也能被其他数据泵处理）。  
        * <font color="red">可以使用EXTTRAIL或RMTEXTRAIL参数在extract文件中，配置trail文件存放位置，默认情况下存放在OGG根目录下的./dirdat文件下。</font>
        * Replicat：读取trail文件并应用DML和DDL操作到目标数据库。

- ##### trail文件的创建及保持  
    + trail文件被需要时自动创建，<font color="red">但是当在配置文件中加入ADD RMTTRAIL或ADD EXTTRAIL两个参数时，需要为trail文件手动指定名称。</font>默认情况下，trail文件存储在dirdat的OGG子目录中。  
全部trail文件自动过期，对于文件保持允许没有中断。当新trail文件被创建时，内置两个字符以生成唯一的文件名（文件名+序号，序号在000000~999999之间）。  


    + 与一个相比你能创建更多trail文件去分离不同的对象或应用的数据。你可指定执行表或序列的对象链接到在参数文件中配置过EXTTRAIL或RMTTRAIL的trail文件。过期的trail文件能通过管理参数PURGEOLDEXTRACTS删除。　
        * 例子：PURGEOLDEXTRACTS ./dirdat/*,usecheckpoints, minkeepdays 7   指定删除超过七天trail文件。
    + 系统最大产出，最小Ｉ/Ｏ负荷，捕获进程以大块容量（事物单元）送入trail文件，事务的顺序被保护。默认OGG向trail中以标准的形式写，在大量复杂数据库中使用专用形式能快速准确的交互。数据通过写入trail文件兼容其他不同应用的格式。  



#### extract files概览  

- 某些配置中，OGG存储捕获的数据在一个捕获文件中extrail文件。  
- extract文件是一个单项文件，或者他能被配置转存到不同的数据文件，以不被操作系统限制文件大小的形式。
- 某种角度看，extract文件要比trail文件小。
    - 除了checkpoint不被记录这点不同之外。  
    - extract文件为运行时自动创建。
    - 同版本OGG中既支持trail，有支持extract。



#### checkpoints概览  

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;为了达到恢复的目，checkpoint进程在硬盘上存储进程读写的位置。检查点确保同步被源端extract捕获和目标端replicat复制造成的数据改变，被准确的捕获和阻止冗余处理。  

- 检车点checkpoints提供容错机制阻止本应该由操作系统、网络或OGG进程处理的数据丢失所造成的重启。  
- 对于复杂的同步配置，检查点支持多样化Extract和replicat进程读取相同trails文件。  
- 检查点在内部处理掉确认并阻止网络中造成的消息丢失，OGG有专用个的数据分发保护技术。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Extract为他在数据源和本地trail文件中创建检查点的处理位置。  
因为Extract仅捕获提交的数据，所以检查点必须追踪所有事务中已提交的操作。这就需要Extract在事务日志中准确的读取检查点，追加到最旧事务开始时的位置，这能在当前(current log file)或任何日志(redo log file)中实现。  

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;为了控制大量必须处理的过期事务日志，在确定的时间间隔Extract持久化当前状态和处理的数据到硬盘，包括状态和长时间运行的事务数据。如果Extract进程停止达一个时间间隔，他现在能恢复到以之前的或更旧的检查点位置，而不是返回到最老开始事务时，第一次的日志位置。  


&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;复制进程创建检查点在trail文件中，Replicat存储检查点在一个目标端数据库的<font color="red">checkpoint表中</font>，联系以提交的事务在trail文件中的位置。即使Replicat进程或数据库进程出问题的情况下，检查点表通过确保一个事务仅被应用一次来保护数据的一致性。出于报告的目的，Replicat也有一个检查点文件在硬盘上，存放于<font color="red">OGG的dirchk子目录</font>。  

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;检查点不调用非持续类型的配置，如果需要时可以在开始点配置，如:初始化数据时配置的初始化Replicat进程。



#### Manager概览  

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Manager是OGG的进程控制进程。Extract和Replicat能被开始之前，Manager必须运行在每一个OGG系统，Manager必须保持进程持续运行，以便于源端管理方法能被执行。  

Manager职能如下：  

- 开启OGG进程
- 开启动态进程
- 占用进程端口号
- 执行trail文件管理
- 创建事件、错误和阈值报告

一个Manager能够管理所有Extract和Replicat进程，在Windows系统上，Manager能作为一个服务运行。   



#### Collector概览

当连续的在线更改同步进程时，Collector是一个在目标端后台运行的进程。  

Collector处理如下工作：   

- 为Manager和Extract建立连连接，在目标端扫描和绑定到一个可用的端口，并传送端口给Manager计划去提取进程。  
- 能接收捕获分为Extract捕获改变和写入磁盘trail文件中的数据库改变。当网络连接被调用时Manager自动开启Collector，所以OGG用户不能与它交互。Collector仅能从一个Extract进程接收信息，所以每个使用的Extract进程都有一个Collector。 
- Collector如果需要时，也可以手动的方式运行，被称作静态Collector（与常规的动态Collector相对应）。几个进程可共享一个静态Collector。静态Collector可以运行在一个指定的端口。  
- 默认，Extract初始化TCP/IP连接从源端到目标端Collector，但是OGG能在目标端被配置便于初始化连接。
    - 从目标端初始化连接需要条件，例如：目标端在可信的时区，但源端在不可信时区。



#### 进程类型  

OGG配置如下进程类型以满足不同需求：  

- ##### online process  
    +  在线运行直至用户停止Extract或Replicat进程。在线处理持有在trail中的检查点，以便处理能中断后重用。使用online process持续捕获和复制DML及DDL操作保持源端和目标端一致。
    +  使用EXTRACT和REPLICAT参数配置在命令行或配置文件中配置这种进程类型。
- ##### source-is_table
    + source-is_table进程直接从源对象捕获当前静态数据集合为另一个数据库初始化，此进程类型不使用检查点；
    + SOURCEISTABLE参数配置之中进程类型。
- ##### special-run
    + special-run复制进程应用数据从开始点到结束点。可以使用Special复制进程初始化数据，他能被用在线抽取进程一起使用，例如：replicat应用一天而不是持续应用。
    + 此进程类型不使用checkpoint，因为完全运行从开始点运行到结束点。
    + SPECIALRUN参数配置这种进程类型。
- ##### remote task
    + remote task是一个特别的初始化进程类型，用于直接以TCP/IP同Replicat进程捕获交流。
    + 既不是Collector进程也不是临时硬盘存储在trail或被使用的文件中。
    + 配置方式是在Extract参数文件中配置RMTTASK参数。



#### groups组  

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在一个系统上为了区分复杂的extract和replicat进程，可以组的方式定义进程。   
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<font color="red">例如：并行的方式复制不同的数据集合，你应该创建两个replicat组。</font>     
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;进程组由既不是extract也不是replicat的进程组成，通过检查点文件和其他文件以支撑进程运行。<font color="red">对于Replicat进程组也包含检查点表。</font>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;可以定义groups使用OGG交互界面使用ADD EXTRACT和ADD REPLICAT命令。  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;可以将所有文件和检查点联系到一个groups，共享一个name。  



#### Commit Sequence Number（CSN）  

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;当OGG工作时，可能需要参考CSN。CSN被OGG进程构造用来确定一个事务，以保持事务的一致和数据完整性。它唯一的定义一个数据库事务提交的时间点。CSN能在事务日志中被调用捕获，用于在trail文件中复位Replicat或完成其他目的。CSN被一些交互函数、报告或ggsci输出返回。



### 参考文章

- [参考文章1--trail优化部分](https://blog.csdn.net/fly43108622/article/details/47151099)