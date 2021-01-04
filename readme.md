## Understanding kube-proxy on Self-Managed Kubernetes 

### Create simple demo application 
```
git clone https://github.com/xxradar/app_routable_demo.git
cd ./app_routable_demo
./setup.sh
watch kubectl get po -n app-routable-demo
```

### Anaylse the app

##### Get a list of all pods in the namespace
```
kubectl get po -n app-routable-demo -o wide --show-labels
```

##### We can select pods using a selector
```
kubectl get po -n app-routable-demo -o wide -l app=echoserver-1
```

##### We can scale the deployment
```
kubectl scale deploy echoserver-1-deployment --replicas=5 -n app-routable-demo
kubectl get po -n app-routable-demo -o wide -l app=echoserver-1
```

##### We can get the svc in a namespace
```
kubectl get svc -n app-routable-demo -o wide
```
These CLUSTER-IP are choosen from the service CIDR

##### And take a detailed look at a svc
```
kubectl describe svc zone6 -n app-routable-demo
```

##### Accessing the service from within a pod
```
kubectl run -it --rm --image xxradar/hackon curler -n app-routable-demo -- bash
$ curl http://zone1/app1
```

### So how is this implemented ?  
```
sudo iptables --list -t nat
```

Kube-proxy creates a set of chains<br>
One of the first to be hit is ... 


```
sudo iptables --list KUBE-SERVICES -t nat
```
```
Chain KUBE-SERVICES (2 references)
target     prot opt source               destination
...
KUBE-MARK-MASQ  tcp  -- !192.168.0.0/16       10.109.197.25        /* app-routable-demo/zone6:echo-http cluster IP */ tcp dpt:http
KUBE-SVC-RGCN5N2WQYBIR5LN  tcp  --  anywhere             10.109.197.25        /* app-routable-demo/zone6:echo-http cluster IP */ tcp dpt:http
...
```

There is also a chain created todo the distribution ...
```
sudo iptables --list KUBE-SVC-RGCN5N2WQYBIR5LN -t nat
```
```
Chain KUBE-SVC-RGCN5N2WQYBIR5LN (1 references)
target     prot opt source               destination
KUBE-SEP-VFQ5BKTMS2WQMEDV  all  --  anywhere             anywhere             /* app-routable-demo/zone6:echo-http */ statistic mode random probability 0.33333333349
KUBE-SEP-W3FM4QSTZ6S45B6W  all  --  anywhere             anywhere             /* app-routable-demo/zone6:echo-http */ statistic mode random probability 0.50000000000
KUBE-SEP-OFLDMAZYM3MKDZOA  all  --  anywhere             anywhere             /* app-routable-demo/zone6:echo-http */
```



```
sudo iptables --list KUBE-SEP-VFQ5BKTMS2WQMEDV -t nat
```
```
...
Chain KUBE-SEP-VFQ5BKTMS2WQMEDV (1 references)
target     prot opt source               destination
KUBE-MARK-MASQ  all  --  192.168.136.131      anywhere             /* app-routable-demo/zone6:echo-http */
DNAT       tcp  --  anywhere             anywhere             /* app-routable-demo/zone6:echo-http */ tcp to:192.168.136.131:80

Chain KUBE-SEP-W3FM4QSTZ6S45B6W (1 references)
target     prot opt source               destination
KUBE-MARK-MASQ  all  --  192.168.202.1        anywhere             /* app-routable-demo/zone6:echo-http */
DNAT       tcp  --  anywhere             anywhere             /* app-routable-demo/zone6:echo-http */ tcp to:192.168.202.1:80

Chain KUBE-SEP-OFLDMAZYM3MKDZOA (1 references)
target     prot opt source               destination
KUBE-MARK-MASQ  all  --  192.168.214.66       anywhere             /* app-routable-demo/zone6:echo-http */
DNAT       tcp  --  anywhere             anywhere             /* app-routable-demo/zone6:echo-http */ tcp to:192.168.214.66:80
...
```

### Let's create / update a nodeport service
```
sudo iptables --list KUBE-NODEPORTS -t nat
```
```
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: zone1
spec:
  ports:
  - name: echo-http
    port: 80
    protocol: TCP
  - name: echo-https
    port: 443
    protocol: TCP
  selector:
    app: nginx-zone1
  type: NodePort
EOF
```
```
sudo iptables --list KUBE-NODEPORTS -t nat
```
```
Chain KUBE-NODEPORTS (1 references)
target     prot opt source               destination
KUBE-MARK-MASQ  tcp  --  anywhere             anywhere             /* app-routable-demo/zone1:echo-https */ tcp dpt:31811
KUBE-SVC-L3YZ2TTPQWRWIZIY  tcp  --  anywhere             anywhere             /* app-routable-demo/zone1:echo-https */ tcp dpt:31811
KUBE-MARK-MASQ  tcp  --  anywhere             anywhere             /* app-routable-demo/zone1:echo-http */ tcp dpt:30428
KUBE-SVC-TU5JABPMVVCAF4VM  tcp  --  anywhere             anywhere             /* app-routable-demo/zone1:echo-http */ tcp dpt:30428
````
