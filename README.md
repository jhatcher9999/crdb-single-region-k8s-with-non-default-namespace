# Running Cockroach DB in k8s with non-default namespace



This content is adapted from: https://www.cockroachlabs.com/docs/stable/orchestrate-cockroachdb-with-kubernetes.html#manual

with changes to show how to use a non-default namespace.



### Steps:

1. Create your k8s cluster
   You can do this in various ways.  I did mine using GKE:

   ```bash
   $ gcloud container clusters create cockroachdb --machine-type=n1-standard-4 --cluster-ipv4-cidr=10.1.0.0/16
   $ ACCOUNT=`gcloud info | grep Account | awk '{print $2}' | cut -d "[" -f 2 | cut -d "]" -f 1`
   $ kubectl create clusterrolebinding $USER-cluster-admin-binding --clusterrole=cluster-admin --user=$ACCOUNT
   ```

   

2. Create the namespace you want to use

   ```bash
   $ kubectl create namespace non-default-namespace
   ```

   

3. I took the single yaml file "cockroachdb-statefulset-secure.yaml")" referenced in the above article and broke it down into its respective pieces.  I numbered these files from 01 to 09.
   Edit these files.  Anywhere where you see "non-default-namespace," change this to the name of the namespace that you want to use.  Also, make changes to the statefulset as described in the article.

   Some of the objects are "namespaced" (like service accounts, roles, role bindings, services, and stateful sets); but some k8s primitives/objects are not namespaced (like cluster roles and cluster role bindings).  To know whether an object is namespaced or not, you can run the following command and look at the "NAMESPACED" column:

   ```bash
   $ kubectl api-resources -o wide
   ```

   You'll get output like this (output edited by me, for readability):

   ```
   NAME                              SHORTNAMES   APIGROUP                       NAMESPACED   KIND                             
   nodes                             no                                          false        Node                             
   persistentvolumeclaims            pvc                                         true         PersistentVolumeClaim            
   persistentvolumes                 pv                                          false        PersistentVolume                 
   pods                              po                                          true         Pod                              
   serviceaccounts                   sa                                          true         ServiceAccount                   
   services                          svc                                         true         Service                          
   deployments                       deploy       apps                           true         Deployment                       
   replicasets                       rs           apps                           true         ReplicaSet                       
   statefulsets                      sts          apps                           true         StatefulSet                      
   jobs                                           batch                          true         Job                              
   certificatesigningrequests        csr          certificates.k8s.io            false        CertificateSigningRequest        
   pods                                           metrics.k8s.io                 true         PodMetrics                       
   clusterrolebindings                            rbac.authorization.k8s.io      false        ClusterRoleBinding               
   clusterroles                                   rbac.authorization.k8s.io      false        ClusterRole                      
   rolebindings                                   rbac.authorization.k8s.io      true         RoleBinding                      
   roles                                          rbac.authorization.k8s.io      true         Role                             
   storageclasses                    sc           storage.k8s.io                 false        StorageClass                     
   ```

   

4. Run the edited files. 

   ```bash
   $ kubectl create -f 01-serviceaccount.yaml
   
   $ kubectl get serviceaccounts -n non-default-namespace
   NAME          SECRETS   AGE
   cockroachdb   1         25s
   default       1         38s
   
   
   $ kubectl create -f 02-role.yaml 
   role.rbac.authorization.k8s.io/cockroachdb created
   
   $ kubectl get role -n non-default-namespace
   NAME          AGE
   cockroachdb   12s
   
   $  kubectl create -f 03-clusterrole.yaml 
   clusterrole.rbac.authorization.k8s.io/cockroachdb created
   
   $ kubectl get clusterrole cockroachdb
   NAME          AGE
   cockroachdb   56s
   
   $ kubectl create -f 04-rolebinding.yaml 
   rolebinding.rbac.authorization.k8s.io/cockroachdb created
   
   $ kubectl get rolebinding -n non-default-namespace
   NAME          AGE
   cockroachdb   12s
   
   $ kubectl create -f 05-clusterrolebinding.yaml
   
   $ kubectl get clusterrolebinding cockroachdb
   NAME          AGE
   cockroachdb   44s
   
   $ kubectl create -f 06-service.yaml 
   service/cockroachdb-public created
   
   $ kubectl get services -n non-default-namespace
   NAME                 TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)              AGE
   cockroachdb-public   ClusterIP   10.1.252.60   <none>        26257/TCP,8080/TCP   30s
   
   $ kubectl create -f 07-service-internal.yaml 
   service/cockroachdb created
   
   $ kubectl get services -n non-default-namespace
   NAME                 TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)              AGE
   cockroachdb          ClusterIP   None          <none>        26257/TCP,8080/TCP   11s
   cockroachdb-public   ClusterIP   10.1.252.60   <none>        26257/TCP,8080/TCP   2m11s
   
   $ kubectl create -f 08-poddisruptionbudget.yaml 
   poddisruptionbudget.policy/cockroachdb-budget created
   
   $ kubectl get poddisruptionbudget -n non-default-namespace
   NAME                 MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
   cockroachdb-budget   N/A             1                 0                     13s
   
   $ kubectl create -f 09-statefulset.yaml 
   statefulset.apps/cockroachdb created
   
   $ kubectl get statefulset -n non-default-namespace
   NAME          READY   AGE
   cockroachdb   0/3     15s
   
   ```

   

5. Check to see if: a) the stateful set was created, the stateful set created pods, and that those pods kicked off their init-containers successfully.

   ```bash
   $ kubectl describe statefulset cockroachdb -n non-default-namespace
   ```

   ```bash
   $ kubectl describe pod cockroachdb-0 -n non-default-namespace
   ```

   ```bash
   $ kubectl logs cockroachdb-0 -c init-certs -n non-default-namespace
   ```

   This last log is the important bit (in my experience).  What you want to see is this:

   ```
   + hostname -f
   + hostname -f
   + cut -f 1-2 -d .
   + hostname -f
   + cut -f 3- -d .
   + + cuthostname -f 3-4 -f -d
    .
   + + cuthostname -f -f 3
    -d .
   + /request-cert '-namespace=non-default-namespace' '-certs-dir=/cockroach-certs' '-type=node' '-addresses=localhost,127.0.0.1,cockroachdb-0.cockroachdb.non-default-namespace.svc.cluster.local,cockroachdb-0.cockroachdb,cockroachdb-public,cockroachdb-public.non-default-namespace.svc.cluster.local,cockroachdb-public.non-default-namespace.svc,cockroachdb-public.non-default-namespace' '-symlink-ca-from=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt'
   2020/10/20 19:14:40 Looking up cert and key under secret non-default-namespace.node.cockroachdb-0
   W1020 19:14:40.692870       1 client_config.go:529] Neither --kubeconfig nor --master was specified.  Using the inClusterConfig.  This might not work.
   2020/10/20 19:14:40 Secret non-default-namespace.node.cockroachdb-0 not found, sending CSR
   Sending create request: non-default-namespace.node.cockroachdb-0 for localhost,127.0.0.1,cockroachdb-0.cockroachdb.non-default-namespace.svc.cluster.local,cockroachdb-0.cockroachdb,cockroachdb-public,cockroachdb-public.non-default-namespace.svc.cluster.local,cockroachdb-public.non-default-namespace.svc,cockroachdb-public.non-default-namespace
   Request sent, waiting for approval. To approve, run 'kubectl certificate approve non-default-namespace.node.cockroachdb-0'
   ```

   

   Examples of some of the non-successful messages I got here:

   ```
   2020/10/20 18:56:14 Secret non-default-namespace.node.cockroachdb-0 not found, sending CSR
   Sending create request: non-default-namespace.node.cockroachdb-0 for localhost,127.0.0.1,cockroachdb-0.cockroachdb.non-default-namespace.svc.cluster.local,cockroachdb-0.cockroachdb,cockroachdb-public,cockroachdb-public.non-default-namespace.svc.cluster.local,cockroachdb-public.non-default-namespace.svc,cockroachdb-public.non-default-namespace
   2020/10/20 18:56:14 failed to get certificate: CertificateSigningRequest.Create(non-default-namespace.node.cockroachdb-0) failed: certificatesigningrequests.certificates.k8s.io is forbidden: User "system:serviceaccount:non-default-namespace:cockroachdb" cannot create resource "certificatesigningrequests" in API group "certificates.k8s.io" at the cluster scope
   ```

   ```
   2020/10/20 18:36:41 failed to read from secrets: secrets "non-default-namespace.node.cockroachdb-0" is forbidden: User "system:serviceaccount:non-default-namespace:cockroachdb" cannot get resource "secrets" in API group "" in the namespace "non-default-namespace"
   ```

   If you don't see this successful message, check the first 5 files for typos.  There's probably a problem with the role binding or the cluster role binding.  Also, make sure you gave yourself cluster admin privileges while creating the cluster.
   

6. Approve the cert requests.

   ```bash
   $ kubectl get csr
   
   NAME                                                   AGE    REQUESTOR                                                 CONDITION
   non-default-namespace.node.cockroachdb-0               18s    system:serviceaccount:non-default-namespace:cockroachdb   Pending
   non-default-namespace.node.cockroachdb-1               18s    system:serviceaccount:non-default-namespace:cockroachdb   Pending
   non-default-namespace.node.cockroachdb-2               17s    system:serviceaccount:non-default-namespace:cockroachdb   Pending
   
   ```

   ```bash
   $ kubectl certificate approve non-default-namespace.node.cockroachdb-0
   certificatesigningrequest.certificates.k8s.io/non-default-namespace.node.cockroachdb-0 approved
   $ kubectl certificate approve non-default-namespace.node.cockroachdb-1
   certificatesigningrequest.certificates.k8s.io/non-default-namespace.node.cockroachdb-1 approved
   $ kubectl certificate approve non-default-namespace.node.cockroachdb-2
   certificatesigningrequest.certificates.k8s.io/non-default-namespace.node.cockroachdb-2 approved
   ```

   Once the cert requests are approved, the pods should show that they're Running but not yet Ready.

   ```bash
   $ kubectl get pods -n non-default-namespace
   NAME            READY   STATUS    RESTARTS   AGE
   cockroachdb-0   0/1     Running   0          4m30s
   cockroachdb-1   0/1     Running   0          4m29s
   cockroachdb-2   0/1     Running   0          4m29s
   ```

   

7. Edit (with correct namespace name) and run the next file.

   ```bash
   $ kubectl create -f 10-cluster-init-secure.yaml 
   job.batch/cluster-init-secure created
   ```

   

8. This file should have created a new cert request.  Approve it.  Notice that the cert's name has the namespace name in it.

   ```bash
   $ kubectl certificate approve non-default-namespace.client.root
   certificatesigningrequest.certificates.k8s.io/non-default-namespace.client.root approved
   ```

   

9. Now the pods should show that they're ready.

   ```bash
   $ kubectl get job cluster-init-secure -n non-default-namespace
   NAME                  COMPLETIONS   DURATION   AGE
   cluster-init-secure   1/1           36s        48s
   
   
   $ kubectl get pods -n non-default-namespace
   NAME                        READY   STATUS      RESTARTS   AGE
   cluster-init-secure-s57wv   0/1     Completed   0          74s
   cockroachdb-0               1/1     Running     0          8m14s
   cockroachdb-1               1/1     Running     0          8m13s
   cockroachdb-2               1/1     Running     0          8m13s
   ```

   

10. Edit (with the desired namespace name) and run the last file.

    ```bash
    $ kubectl create -f 11-clientsecure.yaml 
    pod/cockroachdb-client-secure created
    ```

    

11. Start a sql session and create some database objects.

    ```bash
    $ kubectl exec -it cockroachdb-client-secure -n non-default-namespace -- ./cockroach sql --certs-dir=/cockroach-certs --host=cockroachdb-public
    #
    # Welcome to the CockroachDB SQL shell.
    # All statements must be terminated by a semicolon.
    # To exit, type: \q.
    #
    # Server version: CockroachDB CCL v20.1.7 (x86_64-unknown-linux-gnu, built 2020/10/12 16:04:22, go1.13.9) (same version as client)
    # Cluster ID: cca8637e-cd6e-4984-8c54-af4fbdf06fc8
    #
    # Enter \? for a brief introduction.
    #
    root@cockroachdb-public:26257/defaultdb> 
    ```

    While you're in the SQL shell, create a few objects which can be used to verify basic functionality.

    ```SQL
    root@cockroachdb-public:26257/defaultdb> CREATE DATABASE bank;
    CREATE DATABASE
    
    Time: 31.91642ms
    
    root@cockroachdb-public:26257/defaultdb> CREATE TABLE bank.accounts (id INT PRIMARY KEY, balance DECIMAL);
    CREATE TABLE
    
    Time: 34.785854ms
    
    root@cockroachdb-public:26257/defaultdb> INSERT INTO bank.accounts VALUES (1, 1000.50);
    INSERT 1
    
    Time: 25.955251ms
    
    root@cockroachdb-public:26257/defaultdb> SELECT * FROM bank.accounts;
      id | balance
    -----+----------
       1 | 1000.50
    (1 row)
    
    Time: 3.364896ms
    
    root@cockroachdb-public:26257/defaultdb> CREATE USER roach WITH PASSWORD 'Q7gc8rEdS';
    CREATE ROLE
    
    Time: 108.23164ms
    
    root@cockroachdb-public:26257/defaultdb> GRANT admin TO roach;
    
    \q		
    ```

    

12. Run the port forwarding command that lets you work in the Admin UI.

    ```bash
    $ kubectl port-forward cockroachdb-0 8080 -n non-default-namespace
    ```

    

13. At this point, you can tear down your cluster.

    ```bash
    $ kubectl delete pods,statefulsets,services,persistentvolumeclaims,persistentvolumes,poddisruptionbudget,jobs,rolebinding,clusterrolebinding,role,clusterrole,serviceaccount -l app=cockroachdb -n non-default-namespace
    pod "cockroachdb-0" deleted
    pod "cockroachdb-1" deleted
    pod "cockroachdb-2" deleted
    service "cockroachdb" deleted
    service "cockroachdb-public" deleted
    persistentvolumeclaim "datadir-cockroachdb-0" deleted
    persistentvolumeclaim "datadir-cockroachdb-1" deleted
    persistentvolumeclaim "datadir-cockroachdb-2" deleted
    poddisruptionbudget.policy "cockroachdb-budget" deleted
    job.batch "cluster-init-secure" deleted
    rolebinding.rbac.authorization.k8s.io "cockroachdb" deleted
    warning: deleting cluster-scoped resources, not scoped to the provided namespace
    clusterrolebinding.rbac.authorization.k8s.io "cockroachdb" deleted
    role.rbac.authorization.k8s.io "cockroachdb" deleted
    clusterrole.rbac.authorization.k8s.io "cockroachdb" deleted
    serviceaccount "cockroachdb" deleted
    
    ```

    

