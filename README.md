# cockroachdb-install-k8s

To Install cockroachdb,

kubectl create ns cockroach-operator-system

kubectl config set-context --current --namespace cockroach-operator-system

kubectl apply -f crds.yaml

kubectl apply -f operator.yaml

kubectl apply -f example.yaml


Detailed information can be found thru cockroachdb Official Git Repo: https://github.com/cockroachdb/cockroach-operator


Install the Operator
Apply the custom resource definition (CRD) for the Operator:

kubectl apply -f https://raw.githubusercontent.com/cockroachdb/cockroach-operator/master/install/crds.yaml
Apply the Operator manifest. By default, the Operator is configured to install in the cockroach-operator-system namespace. To use the Operator in a custom namespace, download the Operator manifest and edit all instances of namespace: cockroach-operator-system to specify your custom namespace. Then apply this version of the manifest to the cluster with kubectl apply -f {local-file-path} instead of using the command below.

kubectl apply -f https://raw.githubusercontent.com/cockroachdb/cockroach-operator/master/install/operator.yaml
Validate that the Operator is running:

kubectl get pods
NAME                                  READY   STATUS    RESTARTS   AGE
cockroach-operator-6f7b86ffc4-9ppkv   1/1     Running   0          54s
Start CockroachDB
Download the example.yaml custom resource.

Note: The latest stable CockroachDB release is specified by default in image.name.

Resource requests and limits
By default, the Operator allocates 2 CPUs and 8Gi memory to CockroachDB in the Kubernetes pods. These resources are appropriate for n2-standard-4 (GCP) and m5.xlarge (AWS) machines.

On a production deployment, you should modify the resources.requests object in the custom resource with values appropriate for your workload. For details, see the CockroachDB documentation.

Certificate signing
The Operator generates and approves 1 root and 1 node certificate for the cluster.

Apply the custom resource
Apply example.yaml:

kubectl create -f example.yaml
Check that the pods were created:

kubectl get pods
NAME                                  READY   STATUS    RESTARTS   AGE
cockroach-operator-6f7b86ffc4-9t9zb   1/1     Running   0          3m22s
cockroachdb-0                         1/1     Running   0          2m31s
cockroachdb-1                         1/1     Running   0          102s
cockroachdb-2                         1/1     Running   0          46s
Each pod should have READY status soon after being created.

Access the SQL shell
To use the CockroachDB SQL client, first launch a secure pod running the cockroach binary.

kubectl create -f https://raw.githubusercontent.com/cockroachdb/cockroach-operator/master/examples/client-secure-operator.yaml
Get a shell into the client pod:

kubectl exec -it cockroachdb-client-secure -- ./cockroach sql --certs-dir=/cockroach/cockroach-certs --host=cockroachdb-public
If you want to access the DB Console, create a SQL user with a password while you're here:

CREATE USER roach WITH PASSWORD 'Q7gc8rEdS';
Then assign roach to the admin role to enable access to secure DB Console pages:

GRANT admin TO roach;
\q
Access the DB Console
To access the cluster's DB Console, port-forward from your local machine to the cockroachdb-public service:

kubectl port-forward service/cockroachdb-public 8080
Access the DB Console at https://localhost:8080.

Scale the CockroachDB cluster
Note: Due to a known issue, automatic pruning of PVCs is currently disabled by default. This means that after decommissioning and removing a node, the Operator will not remove the persistent volume that was mounted to its pod. If you plan to eventually scale up the cluster after scaling down, you will need to manually delete any PVCs that were orphaned by node removal before scaling up. For more information, see the documentation.

To scale the cluster up and down, modify nodes in the custom resource. For details, see the CockroachDB documentation.

Do not scale down to fewer than 3 nodes. This is considered an anti-pattern on CockroachDB and will cause errors.

Note: You must scale by updating the nodes value in the Operator configuration. Using kubectl scale statefulset <cluster-name> --replicas=4 will result in new pods immediately being terminated.

Upgrade the CockroachDB cluster
Perform a rolling upgrade by changing image.name in the custom resource. For details, see the CockroachDB documentation.

Stop the CockroachDB cluster
Delete the custom resource:

kubectl delete -f example.yaml
Remove the Operator:

kubectl delete -f https://raw.githubusercontent.com/cockroachdb/cockroach-operator/master/manifests/operator.yaml
Note: If you want to delete the persistent volumes and free up the storage used by CockroachDB, be sure you have a backup copy of your data. Data cannot be recovered once the persistent volumes are deleted. For more information, see the Kubernetes documentation.

Releases
We have a few phases to our releases. The first involves creating a new branch, updating the version, and then getting a PR merged into master with all of the generated files.

Subsequent steps will need to be carried out in TeamCity and RedHat Connect.

Creating a new release PR
From a clean, up-to-date master (seriously...check), run the following where <version> is the desired new version (e.g. 2.2.0).

$ make release/new VERSION=<version>
...
...
$ git push origin release-$(cat version.txt)
This will do the following for you:

Create a new branch named release-<version>
Update version.txt
Generate the manifest, bundles, etc.
Commit the changes with the message Bump version to <version>.
Push to a new branch on origin (that wasn't automated)
Tag the release
After the PR is merged run the following to create the tag (you'll need to be a member of CRL to do this).

git tag v$(cat version.txt)
git push upstream v$(cat version.txt)
Run Release Automation
From here, the rest of the release process is done with TeamCity. A CRL team member will need to perform some manual steps in RedHat Connect as well. Ping one of us in Slack for info.
