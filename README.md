# beyla-alloy

🚀 **COMPLETE GUIDE: Beyla Observability Setup on Kubeadm 3-Node Cluster**
==========================================================================

📋 **BEFORE YOU START - What You Have**
---------------------------------------

*   ✅ Master Node: Dockerfile, grafana-deployment.yml, myapp-1.0.0.jar, myapp.tar, tempo-deployment.yml
    
*   ✅ Worker Nodes: myapp.tar file
    
*   ❌ Fresh Ubuntu VMs with nothing installed
    

🌐 **NETWORK DIAGRAM**
----------------------

text

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   [GCP External] → [Master Node] → [Worker 1: App + Beyla]         ↓               ↓         [Worker 2: App + Beyla]      Grafana        [Tempo + Calico]         ↑      [Traces/Metrics]   `

⏱️ **ESTIMATED TIME: 45-60 minutes**
------------------------------------

🎯 **PHASE 1: SETUP ALL 3 NODES**
=================================

**Step 1.1: On ALL 3 NODES (Master, Worker1, Worker2)**
-------------------------------------------------------

bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   # SSH to each node and run these  # 1. Update system  sudo apt update  sudo apt upgrade -y  # 2. Install Docker  sudo apt install -y docker.io  sudo systemctl enable docker  sudo systemctl start docker  # 3. Add Docker to user group  sudo usermod -aG docker $USER  newgrp docker  # 4. Disable swap  sudo swapoff -a  sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab  # 5. Load kernel modules  sudo modprobe overlay  sudo modprobe br_netfilter  # 6. Set sysctl parameters   `

`sudo tee /etc/sysctl.d/kubernetes.conf <  net.bridge.bridge-nf-call-ip6tables = 1  net.bridge.bridge-nf-call-iptables = 1  net.ipv4.ip_forward = 1  EOF  sudo sysctl --system  `

**Step 1.2: Install Kubernetes on ALL 3 NODES**
-----------------------------------------------

bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   # 1. Add Kubernetes repo  curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -  sudo apt-add-repository "deb https://apt.kubernetes.io/ kubernetes-xenial main"  # 2. Install kubelet, kubeadm, kubectl  sudo apt update  sudo apt install -y kubelet kubeadm kubectl  sudo apt-mark hold kubelet kubeadm kubectl  # 3. Enable kubelet  sudo systemctl enable kubelet   `

🎯 **PHASE 2: INITIALIZE MASTER NODE**
======================================

**Step 2.1: ONLY on MASTER NODE**
---------------------------------

bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   # 1. Initialize Kubernetes cluster with Calico CIDR  sudo kubeadm init --pod-network-cidr=192.168.0.0/16  # 2. After completion, you'll see a SUCCESS message  #    SAVE THE JOIN COMMAND! It looks like:  #    kubeadm join 10.160.0.X:6443 --token abcdef.1234567890abcdef \  #      --discovery-token-ca-cert-hash sha256:...   `

**Step 2.2: Configure kubectl on MASTER**
-----------------------------------------

bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   # 1. Setup kubectl config  mkdir -p $HOME/.kube  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config  sudo chown $(id -u):$(id -g) $HOME/.kube/config  # 2. Verify  kubectl get nodes  # Should show master as NotReady   `

**Step 2.3: Install Calico CNI on MASTER**
------------------------------------------

bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   # 1. Install Calico  kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml  kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml  # 2. Wait 2 minutes  echo "Waiting for Calico to initialize..."  sleep 120  # 3. Check Calico  kubectl get pods -n calico-system   `

🎯 **PHASE 3: JOIN WORKER NODES**
=================================

**Step 3.1: Prepare Worker Nodes**
----------------------------------

bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   # On EACH WORKER NODE (Worker1, Worker2):  # 1. Load your app image  sudo docker load -i myapp.tar  # 2. Pull required images in advance  sudo docker pull grafana/beyla:latest  sudo docker pull busybox:latest   `

**Step 3.2: Join Workers to Cluster**
-------------------------------------

bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   # 1. Get join command from MASTER  # On MASTER, run:  kubeadm token create --print-join-command  # 2. Copy the output command, then on EACH WORKER, run it:  # Example (USE YOUR ACTUAL COMMAND):  sudo kubeadm join 10.160.0.X:6443 --token YOUR_TOKEN \    --discovery-token-ca-cert-hash sha256:YOUR_HASH   `

**Step 3.3: Verify on MASTER**
------------------------------

bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   # After joining both workers, check  kubectl get nodes  # Should show:  # NAME          STATUS   ROLES           AGE   VERSION  # k8s-master    Ready    control-plane   5m    v1.28.x  # k8s-worker1   Ready              1m    v1.28.x  # k8s-worker2   Ready              1m    v1.28.x   `

🎯 **PHASE 4: DEPLOY OBSERVABILITY STACK**
==========================================

**Step 4.1: Create Namespace**
------------------------------

bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   # On MASTER node  kubectl create namespace beyla   `

**Step 4.2: Build Your App Image**
----------------------------------

bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   # On MASTER node  cd ~  # Go to where your Dockerfile is  # 1. Build image  docker build -t myapp:latest .  # 2. Save and copy to workers (optional, already done)  docker save myapp:latest -o myapp-new.tar  # 3. Copy to workers if needed  # scp myapp-new.tar worker1:~  # scp myapp-new.tar worker2:~   `

**Step 4.3: Deploy Tempo (Traces)**
-----------------------------------

bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   # 1. Create tempo-deployment.yml if not exists  cat > tempo-deployment.yml << 'EOF'  apiVersion: apps/v1  kind: Deployment  metadata:    name: tempo    namespace: beyla  spec:    replicas: 1    selector:      matchLabels:        app: tempo    template:      metadata:        labels:          app: tempo      spec:        containers:        - name: tempo          image: grafana/tempo:2.3.0          args: ["-config.file=/etc/tempo.yaml"]          ports:          - containerPort: 3200          - containerPort: 4317          volumeMounts:          - name: config            mountPath: /etc/tempo.yaml            subPath: tempo.yaml        volumes:        - name: config          configMap:            name: tempo-config  ---  apiVersion: v1  kind: ConfigMap  metadata:    name: tempo-config    namespace: beyla  data:    tempo.yaml: |      server:        http_listen_port: 3200        grpc_listen_port: 9095      distributor:        receivers:          otlp:            protocols:              grpc:      ingester:        max_block_duration: 5m      compactor:        compaction:          block_retention: 1h      storage:        trace:          backend: local          local:            path: /tmp/tempo          wal:            path: /tmp/tempo/wal  ---  apiVersion: v1  kind: Service  metadata:    name: tempo    namespace: beyla  spec:    selector:      app: tempo    ports:    - name: http      port: 3200      targetPort: 3200    - name: otlp-grpc      port: 4317      targetPort: 4317    type: ClusterIP  EOF  # 2. Apply  kubectl apply -f tempo-deployment.yml   `

**Step 4.4: Deploy Grafana**
----------------------------

bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   # 1. Create grafana-deployment.yml if not exists  cat > grafana-deployment.yml << 'EOF'  apiVersion: apps/v1  kind: Deployment  metadata:    name: grafana    namespace: beyla  spec:    replicas: 1    selector:      matchLabels:        app: grafana    template:      metadata:        labels:          app: grafana      spec:        containers:        - name: grafana          image: grafana/grafana:latest          ports:          - containerPort: 3000          env:          - name: GF_SECURITY_ADMIN_PASSWORD            value: "admin"          volumeMounts:          - name: grafana-storage            mountPath: /var/lib/grafana        volumes:        - name: grafana-storage          emptyDir: {}  ---  apiVersion: v1  kind: Service  metadata:    name: grafana    namespace: beyla  spec:    selector:      app: grafana    ports:    - port: 3000      targetPort: 3000    type: NodePort  EOF  # 2. Apply  kubectl apply -f grafana-deployment.yml   `

🎯 **PHASE 5: DEPLOY YOUR APP WITH BEYLA**
==========================================

**Step 5.1: Create RBAC for Beyla**
-----------------------------------

bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   # Create beyla-rbac.yml  cat > beyla-rbac.yml << 'EOF'  apiVersion: v1  kind: ServiceAccount  metadata:    namespace: default    name: beyla  ---  apiVersion: rbac.authorization.k8s.io/v1  kind: ClusterRole  metadata:    name: beyla  rules:  - apiGroups: ["apps"]    resources: ["replicasets"]    verbs: ["list", "watch"]  - apiGroups: [""]    resources: ["pods", "services", "nodes"]    verbs: ["list", "watch"]  ---  apiVersion: rbac.authorization.k8s.io/v1  kind: ClusterRoleBinding  metadata:    name: beyla  subjects:  - kind: ServiceAccount    name: beyla    namespace: default  roleRef:    apiGroup: rbac.authorization.k8s.io    kind: ClusterRole    name: beyla  EOF  # Apply  kubectl apply -f beyla-rbac.yml   `

**Step 5.2: Deploy Your Java App with Beyla Sidecar**
-----------------------------------------------------

bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   # Create myapp-deployment.yml  cat > myapp-deployment.yml << 'EOF'  apiVersion: apps/v1  kind: Deployment  metadata:    name: myapp    namespace: default  spec:    replicas: 2    selector:      matchLabels:        app: myapp    template:      metadata:        labels:          app: myapp      spec:        serviceAccountName: beyla        containers:        - name: myapp          image: myapp:latest          imagePullPolicy: IfNotPresent          ports:          - containerPort: 8080        - name: beyla          image: grafana/beyla:latest          securityContext:            privileged: true          env:          - name: BEYLA_OPEN_PORT            value: "8080"          - name: OTEL_EXPORTER_OTLP_ENDPOINT            value: "http://tempo.beyla.svc.cluster.local:4317"          - name: BEYLA_KUBE_METADATA_ENABLE            value: "true"          - name: BEYLA_PROMETHEUS_METRICS_ENABLE            value: "true"          - name: BEYLA_PROMETHEUS_METRICS_PORT            value: "9998"          volumeMounts:          - name: var-run-beyla            mountPath: /var/run/beyla        volumes:        - name: var-run-beyla          emptyDir: {}  ---  apiVersion: v1  kind: Service  metadata:    name: myapp-service  spec:    selector:      app: myapp    ports:    - protocol: TCP      port: 8080      targetPort: 8080    type: NodePort  EOF  # Apply  kubectl apply -f myapp-deployment.yml   `

🎯 **PHASE 6: CONFIGURE ACCESS**
================================

**Step 6.1: Get External IP and Ports**
---------------------------------------

bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   # On MASTER node  # 1. Get master node external IP  MASTER_IP=$(curl -s http://metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/access-configs/0/external-ip -H "Metadata-Flavor: Google")  # 2. Get NodePorts  GRAFANA_PORT=$(kubectl get svc -n beyla grafana -o jsonpath='{.spec.ports[0].nodePort}')  APP_PORT=$(kubectl get svc myapp-service -o jsonpath='{.spec.ports[0].nodePort}')  echo "========================================"  echo "✅ SETUP COMPLETE!"  echo "========================================"  echo ""  echo "📊 Grafana Dashboard:"  echo "   URL: http://$MASTER_IP:$GRAFANA_PORT"  echo "   Username: admin"  echo "   Password: admin"  echo ""  echo "🚀 Your Java Application:"  echo "   URL: http://$MASTER_IP:$APP_PORT"  echo ""  echo "🔧 To configure Grafana:"  echo "   1. Go to http://$MASTER_IP:$GRAFANA_PORT"  echo "   2. Login with admin/admin"  echo "   3. Go to Connections → Data sources"  echo "   4. Add 'Tempo' data source"  echo "   5. Set URL: http://tempo:3200"  echo "   6. Save & Test"  echo ""  echo "📡 To generate traces:"  echo "   for i in {1..20}; do curl http://$MASTER_IP:$APP_PORT; sleep 2; done"  echo "========================================"   `

**Step 6.2: Open GCP Firewall**
-------------------------------

bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   # Open NodePort range  gcloud compute firewall-rules create allow-nodeports \    --allow tcp:30000-32767 \    --source-ranges=0.0.0.0/0   `

🎯 **PHASE 7: FINAL VERIFICATION**
==================================

**Step 7.1: Run Complete Status Check**
---------------------------------------

bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   # On MASTER node  echo "=== FINAL VERIFICATION ==="  echo ""  echo "1. Cluster Nodes:"  kubectl get nodes  echo ""  echo "2. All Pods Status:"  kubectl get pods -A  echo ""  echo "3. Your Application:"  kubectl get pods -l app=myapp  echo ""  echo "4. Beyla Logs (check instrumentation):"  kubectl logs -l app=myapp -c beyla --tail=3  echo ""  echo "5. Services:"  kubectl get svc -A  echo ""  echo "=== VERIFICATION COMPLETE ==="   `

**Step 7.2: Test Everything**
-----------------------------

bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   # 1. Test your app  curl -v http://$MASTER_IP:$APP_PORT  # 2. Generate traffic for traces  for i in {1..10}; do    curl -s http://$MASTER_IP:$APP_PORT > /dev/null && echo "✓ Request $i" || echo "✗ Request $i"    sleep 2  done  # 3. Check Grafana is accessible  curl -I http://$MASTER_IP:$GRAFANA_PORT   `

🚨 **TROUBLESHOOTING COMMANDS**
===============================

**If pods stuck in ContainerCreating:**
---------------------------------------

bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   kubectl describe pod   kubectl logs  --previous   `

**If images not pulling:**
--------------------------

bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   # Manually pull on each node  sudo docker pull grafana/beyla:latest  sudo docker pull grafana/tempo:2.3.0  sudo docker pull grafana/grafana:latest   `

**If network issues:**
----------------------

bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   # Check Calico  kubectl get pods -n calico-system  kubectl logs -n calico-system --all-containers   `

**If nodes not joining:**
-------------------------

bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   # On worker nodes  sudo kubeadm reset  sudo systemctl restart docker  sudo systemctl restart kubelet  # Then join again   `

✅ **SUCCESS CRITERIA**
======================

1.  ✅ **kubectl get nodes** shows all 3 nodes as **Ready**
    
2.  ✅ **kubectl get pods -A** shows all pods as **Running**
    
3.  ✅ Can access **Grafana** at http://:
    
4.  ✅ Can access **Your App** at http://:
    
5.  ✅ **Beyla logs** show "instrumenting process"
    
6.  ✅ **Traces appear** in Grafana Tempo
    

📞 **IF STUCK - RUN THESE DIAGNOSTICS**
=======================================

bash

Plain textANTLR4BashCC#CSSCoffeeScriptCMakeDartDjangoDockerEJSErlangGitGoGraphQLGroovyHTMLJavaJavaScriptJSONJSXKotlinLaTeXLessLuaMakefileMarkdownMATLABMarkupObjective-CPerlPHPPowerShell.propertiesProtocol BuffersPythonRRubySass (Sass)Sass (Scss)SchemeSQLShellSwiftSVGTSXTypeScriptWebAssemblyYAMLXML`   # 1. Cluster health  kubectl get componentstatuses  # 2. Pod events  kubectl get events --sort-by='.lastTimestamp'  # 3. Node details  kubectl describe nodes  # 4. Service endpoints  kubectl get endpoints -A   `

**FOLLOW THIS GUIDE STEP-BY-STEP FROM PHASE 1****DO NOT SKIP ANY STEP****COPY-PASTE COMMANDS EXACTLY**

Good luck! You can do this! 🚀

**Start with Phase 1 on ALL 3 nodes, then move to Phase 2 on Master only.**
