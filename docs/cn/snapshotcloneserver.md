[English version](../en/snapshotcloneserver_en.md)

# curve 快照克隆文档



## 1、快照

### 1.1 curve快照简介

快照是云盘数据在某个时刻完整的只读拷贝，是一种便捷高效的数据容灾手段，常用于数据备份、制作自定义镜像、应用容灾等。快照系统提供给用户快照功能接口，用户通过快照系统可以创建删除和取消快照，也可以从快照恢复数据或者创建镜像。

curve的快照独立于curve核心服务，支持多级快照，首次快照是全量转储，后面是增量快照，第一次快照时，会把快照数据全量转储到S3，之后的快照只需要转储在上一个版本快照基础上修改过的数据，节省空间。快照数据异步转储到S3对象存储服务上，快照服务通过etcd实现了选主高可用，服务重启可以自动继续未完成的快照。

### 1.2 curve快照架构

![快照架构图](../images/snap.png)

#### 快照克隆系统

快照系统收到用户请求之后，通过调用curvefs的接口创建临时快照，再将临时快照转储到对象存储系统，同时将快照的管理数据持久化到数据库中。快照系统主要的工作可以分为两部分：

1. 对外提供快照接口，供使用者调用来创建删除和查询快照信息。
2. 负责快照数据的组织管理，调用curvefs和对象存储的接口控制快照数据在整个系统中的流动。

#### curvefs

curvefs提供创建、删除和读快照的RPC接口，供快照系统调用。快照创建流程中curvefs是快照数据的源，在curvefs上生成的临时快照通过快照系统转储到对象存储上；从快照恢复卷的流程，快照系统将对象存储中的数据写入curvefs。

#### 对象存储

对象存储提供快照数据的存储能力，只提供对象文件的创建、删除和上传下载接口。供快照系统调用来存储从curvefs上读取的快照数据或者将快照数据从对象存储系统中下载下来写入curvefs中用来恢复卷文件。curve使用nos作为为对象存储。

### 1.3 快照流程

用户向快照克隆系统发起快照请求。请求在快照克隆系统在http service层生成具体的snap task，交给snapshot task manager调度处理。打快照的过程先在curvefs生成一个临时快照，然后把临时快照转储到对象存储系统中，最后删除curvefs的临时快照。

创建快照有以下几个步骤：

1. 生成快照记录，并持久化到etcd。这一步需要进行几个判断：对于同一个卷，同时只能有一个快照正在打的快照，所以需要先判断卷是否有正在处理的快照请求；快照克隆系统理论上支持无限快照，在实际的实现过程中我们对快照克隆的深度增加一个限制，一个卷最多打多少层快照可以在快照的配置文件中进行修改，所以还需要判断下快照层数是否超过限制。如果判断通过，就可以从mds读取原卷的元数据信息，生成快照记录，持久化到etcd。这一步生成的快照状态为pending。
2. 在curvefs创建临时快照，并返回快照的seqNum，更新快照记录的seqNum。
3. 创建快照映射表（见1.4.1），保存快照映射表到对象存储S3。
4. 从curvefs转储快照数据到对象存储S3。转储时，快照克隆系统先从curvefs读取快照数据，然后上传快照数据到对象存储S3。
5. 删除curvefs的临时快照。
6. 更新快照状态。这一步更新快照状态为done。

### 1.4 快照的数据组织

每个快照在快照克隆系统中有一条对应的记录，并持久化到etcd中，快照记录使用uuid作为快照的唯一标识。

在打快照的过程中，快照首先在curvefs临时生成，最终会转储到S3上。

#### 1.4.1 快照数据在S3上的组织

快照的数据以chunk为单位转储到S3上，每个chunk在s3上对应着一个Object，存储数据的object称为data object。每个快照还有一个object记录着快照的元数据信息，称为meta object。meta object记录着快照的文件信息以及快照的chunk到object的映射表的映射关系。meta object包括2部分：

1. 快照文件信息，包括原始数据卷名称，卷大小，chunk大小等
2. 快照chunk映射表，记录着快照data object的列表

下图是快照数据的示意图



![快照映射表](../images/snap-s3-format.png)

### 1.5 curve快照系统的接口

见快照克隆的接口文档。[文档链接](./snapshotcloneserver_interface.md)



## 2、克隆

### 2.1 curve克隆模块简介

第一节介绍了快照系统，用于对curvefs的文件创建快照，并将快照数据增量地转储到S3服务上；但是对于一个完整的快照系统来说，仅仅支持快照创建功能是不够的，快照的作用是文件恢复或者文件的克隆，这一节介绍一下curve的克隆（恢复也可以认为是某种程度上的克隆）。

curve的克隆按照数据来源分，可以分为从快照进行克隆、从镜像进行克隆。按照是否数据全部克隆出来之后才提供服务可以分为延迟克隆和非延迟克隆。

### 2.2 curve克隆架构图

![clone架构图](../images/clone.png)

MDS：负责文件信息和文件状态的管理，对外提供管理操作文件或者查询文件的接口。

SnapshotCloneServer：负责快照和克隆任务信息的管理，处理快照逻辑和克隆逻辑。

S3对象存储：存放快照的数据。

ChunkServer：文件数据的实际存放位置，使用读时拷贝机制支持"Lazy Clone"功能；当Client像克隆卷发起读请求时，如果读区域还未写过，从镜像（保存在curvefs）或者快照(保存在s3服务)上拷贝数据，返回给Client并异步将数据写入到chunk文件中。

### 2.3. 克隆流程

对于curvefs的克隆来说，实际上就是将数据从源位置（源位置指拷贝的对象所在的位置，如果是从快照克隆，源位置就是对象在S3上的位置；如果是从curvefs上的文件做的克隆，源位置就是对象在ChunkServer上的位置）拷贝到目标位置（目标位置是克隆生成的chunk所在的位置）。在实际的使用场景中，很多情况下并不需要同步等待所有数据都拷贝过去，而是在实际用到数据的时候再进行拷贝。因此在克隆的时候可以设置一个lazy的标记，表示是否使用lazy方式进行克隆，如果lazy为true则表示用到时再拷贝；如果lazy为false，则需要等待所有数据拷贝到ChunkServer。由于只有在读取数据时会依赖源位置的数据，所以采用读时拷贝的策略。

#### 2.3.1 创建克隆卷

1. 用户指定要克隆的文件或快照，向SnapshotCloneServer发送创建克隆卷的请求；
2. SnapshotCloneServer收到请求后，如果克隆对象是快照，在快照克隆系统的本地保存的快照信息（目前持久化到etcd）中获取快照信息；如果克隆对象是文件的话，就向MDS查询文件信息。
3. SnapshotCloneServer向MDS发送`CreateCloneFile`请求，对于克隆创建的新文件的初始版本应设为1，MDS收到请求后会创建新的文件，并将文件的状态设置为`Cloning`（文件不同状态的含义及作用会在后面章节阐述）。需要注意的是，一开始创建的文件会被放到一个临时目录下，例如用户请求克隆的文件名为“des”，那么此步骤会以"/clone/des"的名称去创建文件。
4. 文件创建成功后，SnapshotCloneServer需要查询各个chunk的位置信息，如果克隆对象为文件，只需要获取文件名称；如果克隆对象为快照，则先获取S3上的metaobject进行解析来获取Chunk的信息；然后先通过MDS为每个Segment上的Chunk分配copyset，再调用ChunkServer的`CreateCloneChunk`接口为获取到的每个Chunk创建新的Chunk文件，新建的Chunk文件的版本为1，Chunk中会记录源位置信息，如果克隆对象为文件，可以以/filename/offset@cs作为location，filename表示数据源的文件名，@cs表示文件数据在chunkserver上；如果克隆对象为快照，以url@s3作为location，url表示源数据在s3上的访问url，@s3表示源数据在s3上。
5. 当所有的Chunk都在chunkserver创建并记录源端信息后，SnapshotCloneServer通过`CompleteCloneMeta`接口将文件的状态修改为`CloneMetaInstalled`，到这一步为止，lazy的方式和非lazy的方式的流程都是一样的，两者的差别在下面的步骤中体现。
6. 如果用户指定以lazy的方式进行克隆，SnapshotCloneServer首先会将前面创建的“/clone/des”重命名为“des”，这样一来，用户就可以访问到新建的文件并进行挂载读写。lazy克隆不会继续后面的数据拷贝，除非显式调用快照克隆系统的flatten接口，然后SnapshotCloneServer会继续循环调用`RecoverChunk`异步地触发ChunkServer上Chunk数据的拷贝，当所有的Chunk都拷贝成功以后，再调用`CompleteCloneFile`将文件的状态改为`Cloned`，调用时指定的文件名为“des”，因为在前面已经将文件重命名了。
7. 而如果用户没有指定以lazy的方式进行克隆，意味着需要将所有的数据同步下来后才能对外提供服务，SnapshotCloneServer首先循环调用`RecoverChunk`触发ChunkServer上Chunk数据的拷贝，当所有Chunk数据拷贝下来后，调用`CompleteCloneFile`将文件的状态改为`Cloned`，此时调用时指定的文件名为“/clone/des”；然后再将“/clone/des”重命名为“des”供用户使用。

#### 2.3.2 恢复文件

恢复的操作本质也是利用了克隆的过程，简单描述来说就是新建一个临时文件，将新建的文件当做克隆来处理，克隆成功后，删除原文件，然后将新文件重命名为原文件名，该文件的属性和原文件是一样的（文件名、文件大小、文件版本等），在恢复过程中相对于克隆过程主要有以下几个不同点：

1. 恢复前会获取原文件的信息，然后用原文件的文件名、版本号和文件大小作为参数来创建新的文件，新建的文件同克隆一样放在/clone目录下；
2. 调用CreateCloneChunk时指定的版本号为Chunk在快照中的实际版本号；
3. 恢复成功后，在将新文件重命名为原文件名时会覆盖原文件。

#### 2.3.3 写克隆卷

克隆卷的写入过程同写普通卷一样，区别在于克隆卷的chunk会记录bitmap，数据写入时，会将对应位置的bit置位1；bitmap的作用在于读克隆卷时区分哪些区域已经被写过，哪些区域未被写过需要从克隆源读取。

#### 2.3.4 读克隆卷

1.客户端发起读请求；

2.chunkserver端接收到读请求后，会判断所写的chunk上记录的表示源对象位置的flag，然后会再判断读取的区域的在bitmap中对应的bit是否为1，如果不是1会触发拷贝；

3.chunkserver通过记录的位置从源对象上拷贝数据，拷贝以slice(默认1MB，可配置)为单位；

4.拷贝成功后，返回用户读取数据。

#### 2.3.5 文件状态

上面过程中我们提到了克隆文件拥有多种状态，这些不同状态代表了什么含义，其作用又是什么？

![clone-file-status](../images/clone-file-status.png)

- Cloning

  克隆文件的初始状态，此时表示SnapshotCloneServer正在创建Chunk，并将Chunk的数据源位置信息装载到各个Chunk上。在这一过程中，文件不可用，用户在界面无法看到文件，只能看到任务信息。

  此状态下，对于克隆来说会在/clone目录下存在一个文件；对于恢复来说/clone目录存在一份文件同时原文件也存在。

- CloneMetaInstalled

  如果文件处于这一状态，说明Chunk的源位置信息都已装载成功，如果用户指定了lazy的方式，此状态下的文件就可以对外提供挂载使用了；此状态过程中，SnapshotCloneServer会去触发各个Chunk从数据源处拷贝数据，此时文件还不允许打快照。

  此状态下，如果是lazy的方式，文件会被移到实际目录中，如果是恢复会先删除原文件；如果是非lazy的方式，文件所处位置同Cloning状态。

- Cloned

  此状态下说明所有Chunk的数据已经全部完成了拷贝，此时的文件可提供所有的功能服务。

  此状态下文件会被移到实际目录中，如果是恢复会先删除原文件。

### 2.4 curve快照系统的接口

见快照克隆的接口文档。[文档链接](./snapshotcloneserver_interface.md)

