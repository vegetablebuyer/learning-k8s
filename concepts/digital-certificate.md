## k8s的认证系统
### 数字证书

- 对称加密：加密跟解密都使用同一个密钥，
    - 优点：加解密的效率高
    - 缺点：密钥不能公开，密钥泄露之后加密就失去意义
- 非对称加密：公私钥一对，公钥加密的数据只能私钥解密，同样的，私钥加密的数据只能公钥解密
    - 优点：公钥可以公开，安全
    - 缺点：加解密的效率比较低
http协议为明文传输，有极高的数据泄露的风险。于是https应运而生，https其实就是http+tls。其中tls采用的是非对称加密

#### 1. 数字证书的内容：
1. 证书持有者的公钥
2. 证书的持有者、用途、颁发机构、有效时间等信息
3. Certificate Signature = CA私钥加密(Hash(1 + 2))

#### 2. 客户端验证数字证书有效性的过程：
1. CA公钥解密(1.3 Certificate Signature) = Hash1
2. Hash(数字证书内容(1.1 + 1.2)) = Hash2
3. 对比Hash1跟Hash2，一致就认为数字证书有效，不一致就认为证书无效

这里有一个问题，我们怎么相信CA公钥呢？答案是我们只能选择无条件的相信，所以CA的机构都需要是权威的机构。\
一般CA证书到服务器证书之间还会多级的证书机构，每一个级别的证书公钥的验证原理是一样的，一直验证到最高一级的CA根证书我们选择无条件相信。

多级证书机构存在的优势：
1. 减少根证书机构的工作量
2. 中间证书机构的私钥泄露，可以快速在线吊销

https单向认证的过程：
1. 客户端发起建立 HTTPS 连接请求，将 SSL 协议版本的信息发送给服务器端；
2. 服务器端将本机的公钥证书（server.crt）发送给客户端；
3. 客户端用CA公钥 (ca.crt) 解密公钥证书 (server.crt)，验证证书的有效性，取出了服务端公钥；
4. 客户端生成一个随机数（密钥 R），用刚才得到的服务器公钥去加密这个随机数形成密文，发送给服务端；
5. 服务端用自己的私钥 (server.key) 去解密这个密文，得到了密钥 R；
6. 服务端和客户端在后续通讯过程中就使用这个密钥R进行通信了

https双向认证的过程：
1. 客户端发起建立 HTTPS 连接请求，将 SSL 协议版本的信息发送给服务端；
2. 服务器端将本机的公钥证书 (server.crt) 发送给客户端；
3. 客户端读取公钥证书 (server.crt)，验证证书的有效性，取出了服务端公钥；
4. 客户端将客户端公钥证书 (client.crt) 发送给服务器端；
5. 服务器端用CA公钥 (ca.crt) 解密客户端公钥证书 (client.crt)，验证证书的有效性，拿到客户端公钥；
6. 客户端发送自己支持的加密方案给服务器端；
7. 服务器端根据自己和客户端的能力，选择一个双方都能接受的加密方案，使用客户端的公钥加密后发送给客户端；
8. 客户端使用自己的私钥解密加密方案，生成一个随机数 R，使用服务器公钥加密后传给服务器端；
9. 服务端用自己的私钥去解密这个密文，得到了密钥 R
10. 服务端和客户端在后续通讯过程中就使用这个密钥R进行通信了。

#### 查看数字证书的内容
```shell script
root@hostname:/path/# openssl x509 -in apiserver.crt -text -noout
Certificate:
    Data:
        Version: 3 (0x2)   # 证书遵循的版本规格
        Serial Number: 3067901418675640369 (0x2a936015fa141831)  # 证书的序列号
        Signature Algorithm: sha256WithRSAEncryption   # 证书使用的签名算法
        Issuer: CN = kubernetes   # 证书的发行机构
        Validity  # 证书的有效期
            Not Before: Aug 26 08:33:38 2020 GMT
            Not After : Aug 28 08:12:19 2030 GMT
        Subject: CN = kube-apiserver   # 证书的所有者信息
        Subject Public Key Info:   # 证书的公钥信息
            Public Key Algorithm: rsaEncryption
                RSA Public-Key: (2048 bit)
                Modulus:
                    00:cb:d6:28:f6:5c:f7:c8:9a:52:85:94:e8:43:68:
                    b1:a3:0f:0b:24:b4:3a:c6:84:ef:76:42:92:a5:41:
                    af:12:dc:ff:fa:a7:27:ef:37:b1:5d:08:21:90:9f:
                    ee:66:63:76:9e:1d:fe:ce:4d:c4:75:61:80:e3:f4:
                    47:09:7b:0f:2b:5c:56:29:26:f5:33:2a:77:d5:0b:
                    4a:19:95:74:68:0a:e6:06:42:8f:da:1b:9d:be:2d:
                    28:24:55:6d:96:2c:70:b9:2e:c8:56:79:bf:00:b0:
                    a4:a5:95:55:29:b2:49:d7:1c:6d:15:e7:3c:5d:b1:
                    12:7d:20:3b:30:d2:b7:ea:fe:6e:d2:38:c2:46:a9:
                    46:1d:df:b1:08:67:95:dd:43:2e:ca:f6:12:54:4c:
                    93:5e:1b:f5:11:2c:64:f4:14:a5:97:3e:1a:df:5c:
                    e1:c5:39:e2:53:4a:c2:08:77:64:7e:d5:ab:8a:2c:
                    f6:ad:ce:18:03:6f:16:0d:c1:1d:92:e0:8a:3a:0c:
                    08:41:22:a5:03:8b:ed:48:67:c6:6d:cc:6f:5a:53:
                    54:77:7c:59:a1:7b:93:58:9f:1e:79:3d:b2:5b:29:
                    a3:b1:11:cc:47:d5:c5:0c:23:e5:0b:23:82:ba:9a:
                    ba:b0:12:e0:12:2f:5d:30:0d:dc:18:fb:ff:67:96:
                    92:f1
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage:
                TLS Web Server Authentication
            X509v3 Subject Alternative Name:
                DNS:k8s-bke-id-dev-10-162-34-10-node1, DNS:kubernetes, DNS:kubernetes.default, DNS:kubernetes.default.svc, IP Address:10.162.84.1
    Signature Algorithm: sha256WithRSAEncryption
         51:36:cd:eb:11:06:5f:15:b7:c3:22:9e:cd:fe:b3:94:73:78:
         dd:b6:c4:66:6c:80:2a:23:7e:5f:47:f5:9a:59:e7:bb:e2:5c:
         32:a4:90:d0:a4:18:33:5c:3e:2e:4a:d3:3b:3d:d2:15:ae:55:
         b2:e5:76:47:14:10:d4:cc:e0:e5:b6:8a:1d:6d:db:3d:29:8e:
         16:48:be:8e:2b:b0:1d:01:c1:94:3b:de:1e:29:6e:09:1f:48:
         93:6a:65:4f:39:2a:5a:29:12:4b:e9:73:1e:12:b7:09:e7:9e:
         34:8c:91:29:15:6b:ea:83:b2:98:1e:94:f3:e6:b0:51:6a:e9:
         75:35:e2:57:13:5c:4f:bb:11:58:60:bf:74:ff:2a:2d:cb:87:
         b7:d6:48:b9:72:cb:2f:73:4a:4e:eb:c8:37:97:76:42:75:a0:
         2c:c8:7c:d9:3d:9c:bb:0a:e5:9a:87:0e:9d:1e:cb:de:38:fc:
         f5:1f:b9:1b:1f:fb:66:35:b1:6f:63:87:c2:0f:ac:6f:c9:4c:
         b9:f3:dc:74:67:7a:51:98:b4:1a:c9:2c:7a:83:8b:f7:79:1d:
         78:9b:35:18:15:77:fe:62:a4:40:fd:fa:95:6d:a8:5a:f6:19:
         39:95:cc:31:32:b1:3b:74:e4:f5:26:1f:e1:1a:2c:21:b9:a4:
         6b:96:7c:f8
```

### kubernetes系统用到的证书
k8s中的https访问都是双向认证，客户端需要认证服务的证书，反之服务端也需要认证客户端的证书。
```shell script
root@hostname:/etc/kubernetes/pki# tree
.
├── apiserver.crt  # apiserver服务端的数字证书，包含了公钥
├── apiserver-etcd-client.crt  # apiserver访问etcd的数字证书，包含了公钥
├── apiserver-etcd-client.key  # apiserver访问etcd的私钥
├── apiserver.key  # apiserver服务端的私钥
├── apiserver-kubelet-client.crt  # apiserver访问kubelet的数字证书，包含了公钥
├── apiserver-kubelet-client.key  # apiserver访问kubelet的私钥
├── ca.crt  # 给apiserver签证书的CA的数字证书，包含了公钥
├── ca.key  # 给apiserver签证书的CA的私钥
├── etcd
│   ├── ca.crt  # 给etcd签证书的CA的数字证书，包含了公钥
│   ├── ca.key  # 给etcd签证书的CA的私钥
│   ├── healthcheck-client.crt  
│   ├── healthcheck-client.key  
│   ├── peer.crt  # peer之间互相访问的数字证书，包含了公钥
│   ├── peer.key  # peer之间互相访问的私钥
│   ├── server.crt  # etcd服务端的数字证书，包含了公钥
│   └── server.key  # etcd服务端的私钥
├── front-proxy-ca.crt
├── front-proxy-ca.key
├── front-proxy-client.crt
├── front-proxy-client.key
├── sa.key # 对sericeaccount token进行签名加密的私钥
└── sa.pub # 对seviceaccount token做解密的公钥
```


### jwt token认证
k8s给每一个service account分配的secret中token就属于jwt token \
jwt token的解码
```shell script
#!/bin/bash

# base64url解码
decode_base64_url() {
  LEN=$((${#1} % 4))
  RESULT="$1"
  if [ $LEN -eq 2 ]; then
    RESULT+='=='
  elif [ $LEN -eq 3 ]; then
    RESULT+='='
  fi
  echo "$RESULT" | tr '_-' '/+' | base64 -d
}

# 解码JWT
decode_jwt()
{
  JWT_RAW=$1
  for line in $(echo "$JWT_RAW" | awk -F '.' '{print $1,$2}'); do
    RESULT=$(decode_base64_url "$line")
    echo "$RESULT" | python -m json.tool
  done
}

# 获取k8s sa token
get_k8s_sa_token()
{
  NAMESPACE=$1
  NAME=$2
  TOKEN_NAME=$(kubectl get sa -n "$NAMESPACE" "$NAME" -o jsonpath='{.secrets[0].name}')
  kubectl get secret -n "$NAMESPACE" "${TOKEN_NAME}" -o jsonpath='{.data.token}' | base64 -d
}

main()
{
  NAMESPACE=$1
  NAME=$2
  if [ -z $NAMESPACE ] || [ -z $NAME ]; then
    echo "Usage: $0 <secret_namespace> <secret_name>"
    exit 1
  fi
  TOKEN=$(get_k8s_sa_token "$NAMESPACE" "$NAME")
  decode_jwt "$TOKEN"
}

main "$@"
```
#### jwt token的结构
jwt token的结构包含三部分，分别为: Header,Payload,Signature，之间用"."分隔，所以一般形式为xxx.yyy.zzz。
```shell script
root@hostname:/# kubectl get secrets -n monitor-platform szdevops-reader-token-zwzs7 -o jsonpath='{.data.token}' | \
base64 -d | \
awk -F '.' '{print $1"\n"$2"\n"$3}'
${HeaderBase64Encode}
${PayloadBase64Encode}
${SignatureBase64Encode}
```

```shell script
root@hostname:/# Header=`echo ${HeaderBase64Encode} | base64 -d`
root@hostname:/# echo ${Header}
{
    "alg": "RS256",
    "kid": "n2BgITQaBhm2wdTMsy9Dxg7OM4kiRbTY2QkRRweyOjg"
}
root@hostname:/# Payload=`echo ${PayloadBase64Encode} | base64 -d`
root@hostname:/# echo ${Payload}
{
    "iss": "kubernetes/serviceaccount",
    "kubernetes.io/serviceaccount/namespace": "monitor-platform",
    "kubernetes.io/serviceaccount/secret.name": "szdevops-reader-token-zwzs7",
    "kubernetes.io/serviceaccount/service-account.name": "szdevops-reader",
    "kubernetes.io/serviceaccount/service-account.uid": "43427a49-8fce-4cf5-a2f7-a90e65517b8c",
    "sub": "system:serviceaccount:monitor-platform:szdevops-reader"
}
```

#### 生成jwt token的步骤
1. value1 = base64Encode(${Header})
2. value2 = base64Encode(${Payload})
3. value3 = hash(value1 + "." + value2)
4. Signature = 私钥加密(value3) 
> 这里的私钥是在 controller-manager指定的参数 --service-account-private-key-file=/etc/kubernetes/pki/sa.key
5. token = base64Encode(base64Encode(${Header}) + "." + base64Encode(${Payload}) + "." + base64Encode(${Signature}))

#### k8s中jwt token的验证过程
生成jwt token步骤的逆过程，其中Signature用公钥解密

### bootstrap tokens认证
> 要启用bootstrap tokens需要在apiserver打开下面参数
> ```shell script 
> --enable-bootstrap-token-auth=true
> ```

### 用token访问apiserver
token=`kubectl get secret -n ${namespace} ${name} -o jsonpath={".data.token"} | base64 -d`
curk -k -H "Authorization: Bearer $token"  https://${apiserver_ip}:6443

bootstrap tokens是一种简单的持有者令牌（Bearer Token），这种令牌是在新建集群，或者在现有集群中添加新节点时使用的。
```shell script
root@hostname:/# kubeadm token list
TOKEN                     TTL         EXPIRES   USAGES                   DESCRIPTION   EXTRA GROUPS
112233.445566778899aabb   <forever>   <never>   authentication,signing   <none>        system:bootstrappers:kubeadm:default-node-token
shopee.kubernetes666666   <forever>   <never>   authentication,signing   <none>        system:bootstrappers:kubeadm:default-node-token
```
#### token的格式
token使用```abcdef.0123456789abcdef```的格式。以```.```为分隔符，第一部分为```{token-id}```，是一种公开信息，用于引用令牌并确保不会泄露认证所使用的秘密信息；第二部分为```{token-secret}```，应该被共享给受信的第三方。
```shell script
# 获取token-secret的明文
kubectl get secret -n kube-system bootstrap-token-${token-id} -o jsonpath='{.data.token-secret}' | base64 -d 
# 获取该token所属的Group
kubectl get secret -n kube-system bootstrap-token-${token-id} -o jsonpath='{.data.auth-extra-groups}' | base64 -d
# 需要有一个clusterrolebinding将该Group绑定clusterrole的权限
kubectl get clusterrolebinding kubeadm:kubelet-bootstrap -o yaml
```

### kubeconfig认证
kubeconfig认证本质上也是证书认证，先看下kubeconfig文件的内容
```shell script
apiVersion: v1
clusters:  # k8s集群的信息，可以配置多个集群
- cluster:
    certificate-authority-data: "${certificate-authority-data}"  # CA的证书内容，用于验证apiserver，经过了base64的编码
    server: https://${apiserver_ip}:6443
  name: kubernetes
contexts:  # 上下文信息，表示用某个用户访问某个集群
- context:
    cluster: kubernetes
    user: kubernetes-admin
    namespace: default
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes  # 该kubeconfig默认使用的上下文
kind: Config
preferences: {}
users:  # 用户信息
- name: kubernetes-admin
  user:
    client-certificate-data: "${client-certificate-data}"  # 用户的数字证书，包含公钥
    client-key-data: "${client-key-data}"  # 用户的私钥
```


