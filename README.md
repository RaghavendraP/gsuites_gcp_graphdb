# Representing Gsuites and Google Cloud Org structure as a Graph Database

Sample procedure to import

1. Gsuites Users-Groups
2. Google Cloud IAM Policies (users,Roles,Resource)
3. Google Cloud Projects

into a JanusGraph Database.  

This allows sysadmins to easily 'see' how users, groups are structured across cloud org domain and can surface access privileges not
readily visible (i.e, group of groups, external service accounts).  Visualizing and having nested group structure also allows to easy see
relationships between projects/users/roles/serviceAccounts.

For example, the following trivial section shows how a user has indirect access to a resource via nested groups:


Annotated flow that shows the links

user vertex `user1@`

1. has edge `in` to group vertex `subgroup1@`
   (i.e., user is member of a group)

2. group vertex `subgroup1@` has edge `in` to group vertex `group_of_groups1@`
   (i.e. group of groups)

3. group vertex `group_of_groups1@`  has edge `in` to role vertex `roles/appengine.codeViewer`
   (i.e, this role includes a group)  

4. Role vertex `roles/appengine.codeViewer`  has edge `in` to resource vertex  `gcp-project-200601`
   (i.,e this resource/project has a role assigned to it)

- ![images/cytoscape_annotation.png](images/cytoscape_annotation.png)

You are free to alter the `vertex->edge->vertex` relationship in anyway you want; i just picked this simple scheme for starters.

SysAdmins can optionally query directly via `gremlin` command line for the same information.


> **WARNING:**  this is just a simple proof of concept: the script attached runs serially and the object hierarchy described below is very basic and likely incorrect!

The intent here is to demonstrate how to load some sample data covering cloud org and gsuites structure into a  graphDB

## Schema

The schema used in this sample is pretty basic and indexes a few properties across users, groups, serviceAccounts project, IAM roles, and resources.  However, there is boilerplate snippet that demonstrates iterating other resource types.   For reference, see `func getGCS(ctx context.Context)`

The code snippet contained there iterates projects and for each bucket in that project, extracts the IAM bindings and members.  The idea is to use that info to generate groovy snippets to emit to a file.

-![image/IAM_UML.png](images/IAM_UML.png)

> Note: the script and sample below does not cover cloud org, pubsub or GCE resource types.  It only iterates and covers projects

- Users
```python
  g.addV('user').property(label, 'user').property('email', email).id().next()
```

- ServiceAccount
```python
  g.addV('serviceAccount').property(label, 'serviceAccount').property('email', email).id().next()
```

- Groups
```python
  g.addV('group').property(label, 'group').property('email', group_email).next()
```

- Projects
```python
  g.addV('project').property(label, 'project').property('projectId', projectId).id().next()
```

- Roles
```python
  g.addV('role').property(label, 'role').property('rolename', name).id().next()  
```


## Setup

The setup steps for this script primarily involves configuring a service account for both domain-wide-delegation and GCP cloud org access.


### Configure Service Account for Domain Wide Delegation

1a. Create Service Account

1b. Enable DWD

- ![images/admin_dwd.png](images/admin_dwd.png)


1c) Find Oauth2 Client ID:

 ```109909984808619572478```

1d) Set Scopes

- ![images/admin_api_scope.png](images/admin_api_scope.png)


### Identify Gsuites CustomerID

You can derive it from your gsuites admin console [here](https://stackoverflow.com/questions/33493998/how-do-i-find-the-immutable-id-of-my-google-apps-account?answertab=votes#tab-top).


In my case its:

```
"customerId": "C023zw3x8"
```


### Configure Service Account for Cloud Org Access


Set cloud ORG policies to apply from the root org node to all resources in the tree:

1e) Org-wide Access:

* Security Reviewer
* Editor


- ![images/iam_role.png](images/iam_role.png)


1f) Genrate and download a the service_account key

   - Make sure the project where you are generating the key has the following APIs enabled:

    - [directory_v1](https://godoc.org/google.golang.org/api/admin/directory/v1)
    - [iam](https://godoc.org/google.golang.org/api/iam/v1)
    - [cloudresourcemanager](https://godoc.org/google.golang.org/api/cloudresourcemanager/v1beta1)

## Install JanusGraph

- Local
Download and untar [JanusGraph](http://janusgraph.org/).  I used version [janusgraph-0.3.0-hadoop2](https://github.com/JanusGraph/janusgraph/releases/tag/v0.3.0)

* Start JanusGraph with defaults (Cassandra, ElasticSearch local)

```bash
$ janusgraph-0.3.0-hadoop2/bin/janusgraph.sh start
Forking Cassandra...
Running `nodetool statusthrift`.. OK (returned exit status 0 and printed string "running").
Forking Elasticsearch...
Connecting to Elasticsearch (127.0.0.1:9200)..... OK (connected to 127.0.0.1:9200).
Forking Gremlin-Server...
Connecting to Gremlin-Server (127.0.0.1:8182)..... OK (connected to 127.0.0.1:8182).
Run gremlin.sh to connect.
```

* Connect via Gremlin

- [gremlin-server](http://tinkerpop.apache.org/docs/current/reference/#gremlin-server)

```bash
$ janusgraph-0.3.0-hadoop2/bin/gremlin.sh

         \,,,/
         (o o)
-----oOOo-(3)-oOOo-----
SLF4J: Class path contains multiple SLF4J bindings.
plugin activated: janusgraph.imports
plugin activated: tinkerpop.server
plugin activated: tinkerpop.gephi
plugin activated: tinkerpop.utilities
gremlin>
```

- Setup Gremlin local connection

```
:remote connect tinkerpop.server conf/remote.yaml session
:remote console
```

At this point, local scripts on local will get sent to the running gremlin server



## Configure and Run ETL script

First edit [main.go][main.go] and set:

- `serviceAccountFile`: path to the service account file
- `subject`:  the email/identity of a gsuites domain admin to represent
- `cx`: Gsuites customerID

then on a system with `go 1.11`, run

```
go run main.go --logtostderr=1 -v 2
```

The parameters will iterate through all the gsuites user,groups as well as the projects and IAM memberships.

If you want to see more details, you can use log level `4` as shown here:

```
 go run main.go --logtostderr=1 -v 4
```

(full `groovy` text output to stdout, use level `10`)

If you want to iterate only a subcomponent, use the `--component` flag.   For example, if you just want to iterate users, run

```
 go run main.go --logtostderr=1 -v 4 --component users
```

```
go run main.go --help
  -component string
    	component to load: choices, all|projectIAM|users|serviceaccounts|roles|groups (default "all")
  -delay int
    	delay in ms for each goroutine (default 100)
```

The output of this run will generate several raw groovy files:

- `users.groovy`:  users to add to the map
- `groups.groovy`:  groups and groupmembers to add
- `projects.groovy`:  list of the projects to add to the graph
- `roles.groovy`:  Custom and possibly generic roles to add
- `serviceaccounts.groovy`:  list of the service ac
- `iam.groovy`:  IAM policy maps.


Then make sure Janusgraph and gremlin are both running before loading each file.

in the gremlin console, run

```
gremlin> :load  /path/to/groovyfile.groovy
```

if its all configured, you should see an output displaying the verticies and edges that were created.


## References

- [https://docs.janusgraph.org/latest/getting-started.html](https://docs.janusgraph.org/latest/getting-started.html)
- [gremlin-server](http://tinkerpop.apache.org/docs/current/reference/#gremlin-server)
- [https://github.com/bricaud/graphexp](https://github.com/bricaud/graphexp)
- [https://medium.com/@BGuigal/janusgraph-python-9e8d6988c36c](https://medium.com/@BGuigal/janusgraph-python-9e8d6988c36c)
- [https://github.com/apache/tinkerpop/tree/master/gremlin-python/](https://github.com/apache/tinkerpop/tree/master/gremlin-python/)
- [https://www.compose.com/articles/graph-101-traversing-and-querying-janusgraph-using-gremlin/](https://www.compose.com/articles/graph-101-traversing-and-querying-janusgraph-using-gremlin/)

### Gremlin References

#### Drop All Verticies and Edges

- On Gremlin Console
```bash
g.V().drop()
g.E().drop()
```

For gremlin-python, simply append suffix commands to submit the request to Gremlin-Server, eg: ```.next()```, ```.iterate()```:

```bash
g.V().drop().iterate()
g.E().drop().iterate()
```

Sample query to retrieve a user and its edges:

* VertexID for a user:
```bash
gremlin> u1 = g.V().has("uid", "user1@esodemoapp2.com")
==>v[2842792]
```

* Outbound Edges from a Vertex:
```
gremlin> g.V().hasLabel('user').has('email', 'user1@esodemoapp2.com').outE()
==>e[1d0d-1eqw-4etx-iyw][65768-in->24584]
==>e[1an1-1eqw-4etx-1o3s][65768-in->77896]
==>e[1925-1eqw-4etx-1o88][65768-in->78056]
==>e[1btp-1eqw-4etx-oej80][65768-in->40988880]
```

* Connected Verticies from a Vertex:
```
gremlin> g.V().hasLabel('user').has('email', 'user1@esodemoapp2.com').out().valueMap()
==>{gid=[subgroup1@esodemoapp2.com], isExternal=[false]}
==>{gid=[group1_3@esodemoapp2.com], isExternal=[false]}
==>{gid=[all_users_group@esodemoapp2.com], isExternal=[false]}
==>{gid=[group_external_mixed1@esodemoapp2.com], isExternal=[false]}
```

### Visualizing the Graph

There are several ways to visualize the generated graph:


#### graphexp

```
git clone https://github.com/bricaud/graphexp.git

cd graphexp
firefox index.html
```

- ![images/graphexp.png](images/graphexp.png)


#### Cytoscape

- Export graph to GraphML file:
```
gremlin> sg = g.V().outE().subgraph('sg').cap('sg').next()
==>tinkergraph[vertices:81 edges:140]

gremlin> sg.io(IoCore.graphml()).writeGraph("/tmp/mygraph.xml")
==>null
```

- Import GraphML to Cytoscape

on Cytoscape, ```File->Import->Network->File```,  Select ```GraphMLFile```  the ```/tmp/mygraph.xml```

Upon import you should see the Cytosscape rendering:

- ![images/cytoscape.png](images/cytoscape.png)


### Gephi

Export to Gephi for Streaming

```bash
gremlin> :remote connect tinkerpop.gephi
==>Connection to Gephi - http://localhost:8080/workspace1 with stepDelay:1000, startRGBColor:[0.0, 1.0, 0.5], colorToFade:g, colorFadeRate:0.7, startSize:10.0,sizeDecrementRate:0.33

gremlin> :remote list
==>0 - Gremlin Server - [localhost/127.0.0.1:8182]-[1f4452c0-4580-4ecf-9648-bc668c4ee68e]
==>*1 - Gephi - [workspace1]
```


## Disconnected Graph

While Google IAM Permissions are inherited down the tree from the Org and Folder, an API scan on a resource does not show those resources explictly.  What that means is once the script executes the scan of IAM permissions on a project at the project level, it does not recognize and link parent inherited owners or roles.  What that will mean is certain projects will be 'disconnected' from the main graph as shown here.  You can remedy this by adding in the domain admin account as an explicit `OWNER`:

![images/iam_visiblity.png](images/iam_visiblity.png)

## Admin API


The following is just a sampl raw JSON snippet for the various API calls made to ```directory_service```, ```IAM``` and ```Cloud Resource Manager```.


- https://cloud.google.com/resource-manager/reference/rest/
- https://cloud.google.com/iam/reference/rest/v1/projects.roles/list
- https://developers.google.com/admin-sdk/directory/v1/guides/authorizing
