## CNI
```cni```也就是container network interface，提出了一种通用的、基于插件的容器网络解决方案。为了实现这个解决方案，下面专有名词需要明确：
- ***container***：***容器***是一个网络单独隔离的实体，可以是```linux network namespace```实现的隔离，也可以是一个vm虚拟机
- ***network***：***网络***由一组拥有独立网络地址，并且能互相通信的网络实体组成。这些实体可以加入网络或者被移除出网络
- ***runtime***：负责执行CNI插件的程序
- ***plugin***：负责读取网络配置，并据此配置容器的网络

CNI定义了下面几个方面
- 网络管理员配置网络的具体格式
- ***runtime***向***network plugin***发送请求的协议
- ***plugin***插件根据网络配置执行的流程
- ***plugin***插件将部分功能委托给其他插件的流程
- ***plugin***插件将结果返回给***runtime***的具体数据类型

### Network configuration format：网络配置的格式
CNI为网络管理员定义了网络配置的格式，其中包含了提供给***runtime***跟***plugin***插件的指令。在插件运行的时候，配置文件的格式先由***runtime***解析并转化一定的形式传递给网络插件。\
通常来说，网络配置文件是一个静态文件，准确地说是存在磁盘上的文件，但是CNI并没有要求这一点。
#### 配置文件格式
网络配置文件是一个json对象，包含下面的keys：
- ```cniVersion```：字符串，该配置文件所属于的cni版本
- ```name```：字符串，网络名字，在同一个主机上必须唯一，由字母或数字开头，可以包含字母、数字、下划线、.跟-
- ```disableCheck```：布尔值，如果值为```true```，则***runtime***不会执行```check```
- ```plugins```：列表，列表的成员类型为***plugin configuration objects***

#### plugin configuration objects
- 必须要求的键值对
    - ```type```：字符串，必须跟主机上cni插件二进制文件的名字匹配
- 可选的键值对，提供给协议使用
    - ```capabilities```：字典数据结构
- 保留的键值对，提供给协议使用：这些键值对由***runtime***在执行的时候自动生成，所以不应该在配置文件中使用
    - ```runtimeConfig```
    - ```args```
    - 其它任何以```cni.dev/```开头的键
- 可选的键值对，比较知名常用的：
    - ```ipMasq```：布尔值，如果插件支持，这个插件决定了要不要设置ip masquerade
    - ```ipam```：字典，
        - ```type```：IPAM插件的可执行文件的名字
    - ```dns```：字典，DNS相关的具体配置
        - ```nameservers```：列表，nameserver服务器的地址
        - ```domain```：字符串
        - ```search```：列表
        - ```options```：列表
- 其他键值对：插件可能定义了其他的键值对，***runtime***需要确保这些键值对在转换的时候被保留下来
> ip masquerade是一种Linux的网络急速，可以看成一种SNAT的特例。只要设置了masquerade，不管网口设置了什么ip，masquerade都会自动读取并SNAT \
> iptables-t nat -A POSTROUTING -s 10.8.0.0/255.255.255.0 -o eth0 -j MASQUERADE
> 如果要用iptables的SNAT，那么网口的ip地址每变化一次都需要重新修改一次
> iptables-t nat -A POSTROUTING -s 10.8.0.0/255.255.255.0 -o eth0 -j SNAT --to-source 192.168.5.3

一个配置文件的例子
```json5
{
  "cniVersion": "1.0.0",
  "name": "dbnet",
  "plugins": [
    {
      "type": "bridge",
      // plugin specific parameters
      "bridge": "cni0",
      "keyA": ["some more", "plugin specific", "configuration"],
      
      "ipam": {
        "type": "host-local",
        // ipam specific
        "subnet": "10.1.0.0/16",
        "gateway": "10.1.0.1",
        "routes": [
            {"dst": "0.0.0.0/0"}
        ]
      },
      "dns": {
        "nameservers": [ "10.1.0.1" ]
      }
    },
    {
      "type": "tuning",
      "capabilities": {
        "mac": true
      },
      "sysctl": {
        "net.core.somaxconn": "500"
      }
    },
    {
        "type": "portmap",
        "capabilities": {"portMappings": true}
    }
  ]
}
```
### 执行协议（Execution Protocol）
CNI插件负责给容器配置网络接口，插件可分为下面两大类：
- "Interface"插件，给容器配置网络接口并保证接口的网络连通性
- "Chained"插件，调整创建好的接口的配置（可能需要创建更多的接口已以保证该功能）
***runtime***给插件传递配置文件并通过环境变量传递额外的```parameters```参数。***runtime***通过***stdin***标准输入提供配置文件。插件如果执行成功通过***stdout***标准输入返回结果，如果执行失败则通过***stderr***标准错误输出返回报错。配置文件跟返回结果都以json格式编码。\
```parameters```参数定义了特定的设置，反之，配置文件则对于任何网络来说都是一样的，但有一些例外。\
***runtime***需要在***runtime***的网络命名空间中执行插件，在大多数情况下，指的就是root网络命名空间。
#### Parameters参数
协议的参数通过环境变量的形式传递给插件
- ```CNI_COMMAND```：指明了请求操作的类型，```ADD```，```DEL```，```CHECK```，```VERSION```
- ```CNI_CONTAINERID```：容器ID
- ```CNI_NETNS```：容器的网络命名空间的路径。(e.g. /run/netns/[nsname])
- ```CNI_IFNAME```：容器网络接口的名字，如果插件无法使用该名字则需要返回错误
- ```CNI_ARGS```：一些额外的参数
- ```CNI_PATH```：CNI可执行二进制文件的路径
#### CNI操作类型
##### ```ADD```
CNI插件在收到```ADD```操作指令的时候，需要做下面两件事情中其中意见：
- 在容器的```CNI_NETNS```网络命令空间中创建```CNI_IFNAME```为名字的网络接口
- 修改```CNI_NETNS```网络命令空间中```CNI_IFNAME```网络接口的配置
***runtime***不应该在没有```DEL```操作的前提下为同一个```CNI_CONTAINERID```，```CNI_IFNAME```组合调用两次```ADD```
