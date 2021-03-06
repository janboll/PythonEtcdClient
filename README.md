Introduction
------------

*PEC* was created as a more elegant and proper client for *etcd* than existing 
solutions. It has an intuitive construction, provides access to the 
complete *etcd* API (of 0.2.0+), and just works.

Every request returns a standard and obvious result, and HTTP exceptions are 
re-raised as Python-standard exceptions where it makes sense (like "KeyError").

The full API is documented [here](http://python-etcd-client.readthedocs.org/en/latest/).


Quick Start
-----------

There's almost nothing to it:

```python
from etcd.client import Client

# Uses the default *etcd* port on *localhost* unless told differently.
c = Client()

c.node.set('/test/key', 5)

r = c.node.get('/test/key')

print(r.node.value)
# Displays "5".

r = c.node.set('/test/key', 10)
print(r.prev_node.value)
# Displays "5".
```


SSL
---
*PEC* also allows for SSL authentication and encrypted communication.

To use it for communication, pass the hostname as the *host* parameter and a 
*is_ssl* of *True*. If you need to pass a bundle of CA certificates (for less 
well-known root authorities), pass *ssl_ca_bundle_filepath*.

```python
c = Client(host='etcd.local', 
           is_ssl=True, 
           ssl_ca_bundle_filepath='ssl/rootCA.pem')
```


General Functions
-----------------

These functions represent the basic key-value functionality of *etcd*.

Set a value:

```python
# Can provide a "ttl" parameter with seconds, for expiration.
r = c.node.set('/node_test/subkey1', 5)

print(r)
# Prints: <RESPONSE: <NODE(ResponseV2AliveNode) [set] [/node_test/subkey1] 
#           IS_HID=[False] IS_DEL=[False] IS_DIR=[False] IS_COLL=[False] 
#           TTL=[None] CI=(5) MI=(5)>>
```

Get a value:

```python
r = c.node.get('/node_test/subkey1')

print(r)
# Prints: <RESPONSE: <NODE(ResponseV2AliveNode) [get] [/node_test/subkey1] 
#           IS_HID=[False] IS_DEL=[False] IS_DIR=[False] IS_COLL=[False] 
#           TTL=[None] CI=(4) MI=(4)>>

print(r.node.value)
# Prints "5"
```

Wait for a change to a specific node:

```python
r = c.node.wait('/node_test/subkey1')

print(r)
# Prints: <RESPONSE: <NODE(ResponseV2AliveNode) [set] [/node_test/subkey1] 
#           IS_HID=[False] IS_DEL=[False] IS_DIR=[False] IS_COLL=[False] 
#           TTL=[None] CI=(5) MI=(5)>>

print(r.node.value)
# Prints "20"
```

In this case, a set with a value of (20) was performed from another terminal, 
and we were given the same exact response that they got. We can set the 
*recursive* parameter to *True* to watch subdirectories and subdirectories-of-
subdirectories as well.

Get children:

```python
r = c.node.set('/node_test/subkey2', 10)

print(r)
# Prints: <RESPONSE: <NODE(ResponseV2AliveNode) [set] [/node_test/subkey2] 
#           IS_HID=[False] IS_DEL=[False] IS_DIR=[False] IS_COLL=[False] 
#           TTL=[None] CI=(6) MI=(6)>>

r = c.node.get('/node_test')

print(r)
# Prints: <RESPONSE: <NODE(ResponseV2AliveDirectoryNode) [get] [/node_test] 
#           IS_HID=[False] TTL=[None] IS_DIR=[True] IS_COLL=[True] 
#           COUNT=[2] CI=(5) MI=(5)>>
```

Get children, recursively:

```python
r = c.node.get('/node_test', recursive=True)

print(r)
# Prints: <RESPONSE: <NODE(ResponseV2AliveDirectoryNode) [get] [/node_test] 
#           IS_HID=[False] TTL=[None] IS_DIR=[True] IS_COLL=[True] 
#           COUNT=[2] CI=(5) MI=(5)>>

for node in r.node.children:
    print(node)

# Prints:
# <NODE(ResponseV2AliveNode) [get] [/node_test/subkey1] IS_HID=[False] 
#   IS_DEL=[False] IS_DIR=[False] IS_COLL=[False] TTL=[None] CI=(5) MI=(5)>
# <NODE(ResponseV2AliveNode) [get] [/node_test/subkey2] IS_HID=[False] 
#   IS_DEL=[False] IS_DIR=[False] IS_COLL=[False] TTL=[None] CI=(6) MI=(6)>
```

Delete node:

```python
r = c.node.delete('/node_test/subkey2')

print(r)
# Prints: <RESPONSE: <NODE(ResponseV2DeletedNode) [delete] 
#           [/node_test/subkey2] IS_HID=[False] IS_DEL=[True] 
#           IS_DIR=[False] IS_COLL=[False] TTL=[None] CI=(6) MI=(7)>>
```


Compare and Swap (CAS) Functions
--------------------------------

These functions represent *etcd*'s atomic comparisons. These allow for a "set"-
type operation when one or more conditions are met.

The core call takes one or more of the following conditions as arguments:

    current_value
    prev_exists
    current_index

If none of the conditions are given, the call acts like a *set()*. If a 
condition is given that fails, a *etcd.exceptions.EtcdPreconditionException* is 
raised.

The core call is:

```python
r = c.node.compare_and_swap('/cas_test/val1', 30, current_value=5, 
                            prev_exists=True, current_index=5)
```

The following convenience functions are also provided. They only allow you to 
check one, specific condition:

```python
r = c.node.create_only('/cas_test/val1', 5)

print(r)
# Prints: <RESPONSE: <NODE(ResponseV2AliveNode) [create] [/cas_test/val1] 
#           IS_HID=[False] IS_DEL=[False] IS_DIR=[False] IS_COLL=[False] 
#           TTL=[None] CI=(10) MI=(10)>>

r = c.node.update_only('/cas_test/val1', 10)

print(r)
# Prints: <RESPONSE: <NODE(ResponseV2AliveNode) [update] [/cas_test/val1] 
#           IS_HID=[False] IS_DEL=[False] IS_DIR=[False] IS_COLL=[False] 
#           TTL=[None] CI=(10) MI=(13)>>

r = c.node.update_if_index('/cas_test/val1', 15, r.node.modified_index)

print(r)
# Prints: <RESPONSE: <NODE(ResponseV2AliveNode) [compareAndSwap] 
#           [/cas_test/val1] IS_HID=[False] IS_DEL=[False] IS_DIR=[False] 
#           IS_COLL=[False] TTL=[None] CI=(10) MI=(14)>>

r = c.node.update_if_value('/cas_test/val1', 20, 15)

print(r)
# Prints: <RESPONSE: <NODE(ResponseV2AliveNode) [compareAndSwap] 
#           [/cas_test/val1] IS_HID=[False] IS_DEL=[False] IS_DIR=[False] 
#           IS_COLL=[False] TTL=[None] CI=(10) MI=(15)>>
```

Directory Functions
-------------------

These functions represent directory-specific calls. Whereas creating a node has 
side-effects that contribute to directory management (like creating a node 
under a directory implicitly creates the directory), these functions are 
directory specific.

Create directory:

```python
# Can provide a "ttl" parameter with seconds, for expiration.
r = c.directory.create('/dir_test/new_dir')

print(r)
# Prints: <RESPONSE: <NODE(ResponseV2AliveDirectoryNode) [set] 
#           [/dir_test/new_dir] IS_HID=[False] TTL=[None] IS_DIR=[True] 
#           IS_COLL=[False] COUNT=[<NA>] CI=(16) MI=(16)>
```

Remove an empty directory:

```python
r = c.directory.delete('/dir_test/new_dir')

print(r)
# Prints: <RESPONSE: <NODE(ResponseV2DeletedDirectoryNode) [delete] 
#           [/dir_test/new_dir] IS_HID=[False] IS_DEL=[True] IS_DIR=[True] 
#           IS_COLL=[False] TTL=[None] CI=(16) MI=(17)>>
```

Recursively remove a directory, and any contents:

```python
c.directory.create('/dir_test/new_dir')
c.directory.create('/dir_test/new_dir/new_subdir')

# This will raise a requests.exceptions.HTTPError ("403 Client Error: 
# Forbidden") because it has children.
r = c.directory.delete('/dir_test/new_dir')

# You have to recursively delete it.
r = c.directory.delete_recursive('/dir_test')

print(r)
# Prints: <RESPONSE: <NODE(ResponseV2DeletedDirectoryNode) [delete] 
#           [/dir_test] IS_HID=[False] IS_DEL=[True] IS_DIR=[True] 
#           IS_COLL=[False] TTL=[None] CI=(16) MI=(20)>>
```


Compare and Delete (CAD)
------------------------

Similar to the "compare and swap" functionality (mentioned above), 
compare-and-delete functionality (introduced in *etcd 0.3.0*) is also exposed.
This functionality allows you to delete a node only if its value or index meets
conditions. Unlike the CAS functionality, CAD functionality is located in the
node and directory functions.

```python
r = c.node.delete_if_value('/node_test/testkey', 'some_existing_value')

print(r)
# Prints: <RESPONSE: <NODE(ResponseV2DeletedNode) [compareAndDelete] 
#           [/node_test/testkey] IS_HID=[False] IS_DEL=[True] IS_DIR=[False] 
#           IS_COLL=[False] TTL=[None] CI=(35) MI=(36)>>

r = c.node.delete_if_index('/node_test/testkey2', 22)

print(r)
# (waiting on bugfixes, to test)

r = c.directory.delete_if_index('/dir_test/testkey2', 22)

print(r)
# (waiting on bugfixes, to test)

r = c.directory.delete_recursive_if_index('/dir_test/testkey2', 22)

print(r)
# (waiting on bugfixes, to test)


Server Functions
----------------

These functions represent calls that return information about the server or 
cluster.

Get version of the specific host being connected to:

```python
r = c.server.get_version()

print(r)
# Prints "0.2.0-45-g98351b9", on my system.
```

The URL prefix of the current cluster leader:

```python
r = c.server.get_leader_url_prefix()

print(r)
# Prints "http://127.0.0.1:7001" with my single-host configuration.
```

Enumerate the prefixes of the hosts in the cluster:

```python
machines = c.server.get_machines()

print(machines)
# Prints: [(u'etcd', u'http://127.0.0.1:4001'), 
#          (u'raft', u'http://127.0.0.1:7001')]
```

Get URL of the dashboard for the server being connected-to:

```python
r = c.server.get_dashboard_url()

print(r)
# Prints: http://127.0.0.1:4001/mod/dashboard
```


In-Order-Keys Functions
-----------------------

These calls represent the in-order functionality, where a directory can be used 
to store a series of values with automatically-assigned, increasing keys. 
Though not quite sufficient as a queue, this might be used to automatically 
generate unique keys for a set of values.

Enqueue values:

```python
io = c.inorder.get_inorder('/queue_test')

io.add('value1')
io.add('value2')
```

Enumerate existing values:

```python
# If you want to specifically return the entries in order of the keys 
# (which is to say that they're in insert-order), use the "sorted"
# parameter.
r = io.list()

for child in r.node.children:
    print(child.value)

# Prints:
# value1
# value2
```


Statistics Functions
--------------------

This functions provide access to the statistics information published by the 
*etcd* hosts.

```python
s = c.stat.get_leader_stats()

print(s)
# Prints: (u'test01', {u'dustinlenovo': LStatFollower(counts=
#         LStatCounts(fail=412, success=75214), latency=
#         LStatLatency(average=7.201827094703149, current=0.535978, 
#         maximum=350.543234, minimum=0.462994, 
#         standard_deviation=22.639299448915402))})

s = c.stat.get_self_stats()

print(s)
# Prints: SStat(leader_info=SStatLeader(leader=u'test01', 
#         uptime=datetime.timedelta(0, 3971, 790306)), name=u'test01', 
#         recv_append_request_cnt=0, send_append_request_cnt=75626, 
#         send_bandwidth_rate=538.5960990745054, 
#         send_pkg_rate=20.10061948402707, 
#         start_time=datetime.datetime(2014, 2, 8, 16, 26, 13), 
#         state=u'leader')
```


Locking Module Functions
------------------------

These functions represent the fair locking functionality that comes packaged.

### Standard Locking

A simple, distributed lock:

```python
l = c.module.lock.get_lock('test_lock_1', ttl=10)
l.acquire()
l.renew(ttl=30)
l.release()
```

This returns the index of the current lock holder:

```python
l.get_active_index()
```

It's also available as a *with* statement:

```python
with c.module.lock.get_lock('test_lock_1', ttl=10):
    print("In lock 1.")
```

### Reentrant Locking

Here, a name for the lock is provided, as well as a value that represents a 
certain locking purpose, process, or host. Subsequent requests having the same 
value currently stored for the lock will return immediately, where others will 
block until the current lock has been released or expired. 

This can be used when certain parts of the logic need to participate during the 
same lock, or a specific function/etc might be invoked multiple times during a 
lock.

This is the basic usage (nearly identical to the traditional lock):

```python
rl = c.module.lock.get_rlock('test_lock_2', 'proc1', ttl=10)
rl.acquire()
rl.renew(ttl=30)
rl.release()
```

This returns the current value of the lock holder(s):

```python
rl.get_active_value()
```

This is also provided as a *with* statement:

```python
with c.module.lock.get_rlock('test_lock_2', 'proc1', ttl=10):
    print("In lock 2.")
```


Leader Election Module Functions
--------------------------------

The leader-election API does consensus-based assignment, meaning that, of all 
of the clients potentially attempting to assign a value to a name, only one 
assignment will be allowed for a certain period of time, or until the key is
deleted.

To set or renew a value:

```python
c.module.leader.set_or_renew('consensus-based key', 'test value', ttl=10)
```

To get the current value:

```python
# This will return:
# > A non-empty string if the key is set and unexpired.
# > None, if the key was set but has been expired or deleted.
r = c.module.leader.get('consensus-based key')

print(r)
# Prints "test value".
```

To delete the value (will fail unless there's an unexpired value):

```python
c.module.leader.delete('consensus-based key', 'test value')
```
