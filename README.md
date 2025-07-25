## PL/Container

This is an implementation of trusted language execution engine capable of
bringing up Docker containers to isolate executors from the host OS, i.e.
implement sandboxing.

The architecture of PL/Container is described at [PL/Container-Architecture](https://github.com/greenplum-db/plcontainer/wiki/PLContainer-Architecture)

### Requirements

1. PL/Container runs on any linux distributions which support Greenplum Database.
1. PL/Container requires minimal Docker version 17.05.
1. GPDB version should be 5.2.0 or later. [For PostgreSQL](README_PG.md)

### Building PL/Container

Get the code repo
```shell
git clone https://github.com/greenplum-db/plcontainer.git
```

### Configue the build

Create the build directory:

```
cd plcontainer
mkdir build
```

To configure the build with the specific version of GPDB, either source the `greenplum_path.sh` first:

```
source /path/to/gpdb/greenplum_path.sh
cd build
cmake ..
```

Or pass the `pg_config` path through command line:

```
cd build
cmake .. -DPG_CONFIG=/path/to/gpdb/bin/pg_config
```

### Build & install

To build & install the plcontainer extension:

```
make
make install
```

To build clients (pyclient, py2client, rclient):

```
make clients
make install
```

Use `make help` to see more build targets.


### Configuring PL/Container

To configure PL/Container environment, you need to enable PL/Container for specific databases by running:

```shell
psql -d your_database -c 'create extension plcontainer;'
```

### Preparing Runtime Environment

Before using PL/Container, you need to prepare the runtime environment by setting up Docker images and runtime configurations:

1. Build the docker images for R & Python environment:

```
cd build
make images_artifact
```

2. Install the PL/Container R & Python docker images:

```shell
# The tar files are in the build directory, you can find the version in the file name
plcontainer image-add -f plcontainer-python-image-<PYTHON_IMAGE_VERSION>-gp<GP_MAJOR_VERSION>.tar.gz
plcontainer image-add -f plcontainer-python2-image-<PYTHON_IMAGE_VERSION>-gp<GP_MAJOR_VERSION>.tar.gz
plcontainer image-add -f plcontainer-r-image-<R_IMAGE_VERSION>-gp<GP_MAJOR_VERSION>.tar.gz
```

3. Add runtime configurations:

```shell
make prepare_runtime
```

After completing these steps, you can verify the available runtimes by running:

```shell
plcontainer runtime-show
```

### Running Simple Examples

Once the runtime environment is prepared, you can test PL/Container with a simple function. Here's an example of a Python function:

```sql
CREATE FUNCTION dummyPython() RETURNS text AS $$
# container: plc_python_shared
return 'hello from Python'
$$ LANGUAGE plcontainer;

SELECT dummyPython();
```

### Running the regression tests

After the runtime environment is properly configured, you can run the full regression test suite:

```
make installcheck
```

### PL/Container Configuration File

PL/Container can be configured by editing the `plcontainer_configuration.xml` file.

The file is located in the `$PGDATA` directory.

Generally, we do not edit this configuration file directly. Instead, after the cluster is started, we update the configuration using the `plcontainer runtime-edit` command.
After executing the above command, the default configuration file appears as follows:

```xml
<?xml version="1.0" ?>
<configuration>
    <runtime>
        <id>plc_python_shared</id>
        <image>python311.alpine:latest</image>
        <command>/clientdir/py3client.sh</command>
        <shared_directory access="ro" container="/clientdir" host="/home/gpadmin/install/cbdb/bin/plcontainer_clients"/>
    </runtime>
    <runtime>
        <id>plc_python_shared_oom</id>
        <image>python311.alpine:latest</image>
        <command>/clientdir/py3client.sh</command>
        <shared_directory access="ro" container="/clientdir" host="/home/gpadmin/install/cbdb/bin/plcontainer_clients"/>
        <setting memory_mb="100"/>
    </runtime>
    <runtime>
        <id>plc_python2_shared</id>
        <image>python27.ubuntu:latest</image>
        <command>/clientdir/pyclient.sh</command>
        <shared_directory access="ro" container="/clientdir" host="/home/gpadmin/install/cbdb/bin/plcontainer_clients"/>
    </runtime>
    <runtime>
        <id>plc_r_shared</id>
        <image>r.alpine:latest</image>
        <command>/clientdir/rclient.sh</command>
        <shared_directory access="ro" container="/clientdir" host="/home/gpadmin/install/cbdb/bin/plcontainer_clients"/>
    </runtime>
</configuration>
```

> Note that all XML elements, names, and attributes are case sensitive.

#### PL/Container Configuration File Elements

These are the XML elements and attributes in a PL/Container configuration file.

##### configuration

Root element for the XML file.

##### backend

Optional. Child of configuration element. This element defines backend configuration.
which can be used to specify the backend in runtime element.

```xml
<backend name="custom_docker" type="remote_docker">
            <address>127.0.0.1</address>
            <port>2375</port>  
</backend>
```

##### runtime

Required. Child of configuration element. One element for each specific container available in the system. These are child elements of the configuration element.

##### id

Required. Child of runtime element. The value is used to reference a Docker container from a PL/Container user-defined function. The id value must be unique in the configuration. The id must start with a character or digit (a-z, A-Z, or 0-9) and can contain characters, digits, or the characters \_ (underscore), . (period), or - (dash). Maximum length is 63 Bytes.

The id specifies which Docker image to use when PL/Container creates a Docker container to run a user-defined function.

##### image

Required. Child of runtime element. The value is the full Docker image name, including image tag. The same way you specify them for starting this container in Docker. Configuration allows to have many container objects referencing the same image name, this way in Docker they would be represented by identical containers.

For example, you might have two runtime elements, with different id elements, `plc_python_128` and `plc_python_256`, both referencing the Docker image `pivotaldata/plcontainer_python:1.0.0`. The first runtime specifies a 128MB RAM limit and the second one specifies a 256MB limit that is specified by the `memory_mb` attribute of a setting element.

##### command

Required. Child of runtime element. The value is the command to be run inside of container to start the client process inside in the container. When creating a runtime element, the plcontainer utility adds a command element based on the language (the -l option).

command element for the Python 2 language.

```xml
<command>/clientdir/pyclient.sh</command>
```

command element for the Python 3 language.

```xml
<command>/clientdir/pyclient3.sh</command>
```

command element for the R language.

```xml
<command>/clientdir/rclient.sh</command>
```

##### shared_directory
Optional. Child of runtime element. This element specifies a shared Docker shared volume for a container with access information. Multiple shared_directory elements are allowed. Each shared_directory element specifies a single shared volume. XML attributes for the shared_directory element:

* host - a directory location on the host system.
* container - a directory location inside of container.
* access - access level to the host directory, which can be either ro (read-only) or rw (read-write).

When creating a runtime element, the plcontainer utility adds a shared_directory element.

```xml
<shared_directory access="ro" container="/clientdir" host="/home/gpadmin/install/cbdb/bin/plcontainer_clients"/>    
```
For each runtime element, the container attribute of the shared_directory elements must be unique. For example, a runtime element cannot have two shared_directory elements with attribute container="/clientdir".

> Allowing read-write access to a host directory requires special consideration.
> * When specifying read-write access to host directory, ensure that the specified host directory has the correct permissions.
> * When running PL/Container user-defined functions, multiple concurrent Docker containers that are running on a host could change data in the host directory. Ensure that the functions support multiple concurrent access to the data in the host directory.

##### setting

Optional. Child of runtime element. This element specifies Docker container configuration information. Each setting element contains one attribute. The element attribute specifies logging, memory, or networking information. For example, this element enables logging.

```xml
<setting use_container_logging="yes"/>
```

**Valid attributes:**

**`cpu_share`**
- Optional. Specify the CPU usage for each PL/Container version in the runtime. 
- The value is a positive integer. The default value is 1024. 
- The value is a relative weighting of CPU usage compared to other containers.
- Example: A container with a `cpu_share` of 2048 is allocated double the CPU slice time compared with container with the default value of 1024.

**`memory_mbs="size"`**
- Optional. The value specifies the amount of memory, in MB, that each container is allowed to use. 
- Each container starts with this amount of RAM and twice the amount of swap space. 
- The container memory consumption is limited by the system cgroups configuration, which means in case of memory overcommit, the container is terminated by the system.

**`resource_group_id="rg_groupid"`**
- Optional. The value specifies the `groupid` of the resource group to assign to the PL/Container runtime. 
- The resource group limits the total CPU and memory resource usage for all running containers that share this runtime configuration. 
- You must specify the `groupid` of the resource group. If you do not assign a resource group to a PL/Container runtime configuration, its container instances are limited only by system resources. 
- For information about managing PL/Container resources, see [About PL/Container Resource Management](http://googleusercontent.com/file_content/0).

**`roles="list_of_roles"`**
- Optional. The value is a Greenplum Database role name or a comma-separated list of roles. 
- PL/Container runs a container that uses the PL/Container runtime configuration only for the listed roles. 
- If the attribute is not specified, any Greenplum Database role can run an instance of this container runtime configuration. 
- Example: You create a UDF that specifies the PL/Container language and identifies a `# container:` runtime configuration that has the `roles` attribute set. When a role (user) runs the UDF, PL/Container checks the list of roles and runs the container only if the role is on the list.

**`use_container_logging="{yes | no}"`**
- Optional. Activates or deactivates Docker logging for the container. 
- The attribute value `yes` enables logging. The attribute value `no` deactivates logging (the default).

**`enable_network="{yes | no}"`**
- Optional. Available starting with PL/Container version 2.2, this attribute activates or deactivates network access for the UDF container. 
- The attribute value `yes` enables UDFs to access the network. The attribute value `no` deactivates network access (the default).

The Greenplum Database server configuration parameter `log_min_messages` controls the PL/Container log level. The default log level is warning.

##### backend

Optional. Child of runtime element. Available starting with PL/Container version 2.2, this attribute specifies the backend to use for the container.

**Valid attributes:**

**`name="{ your custom name | default}"`**
- Required. The value is the name of the backend to use.



### Unsupported feature
There some features PLContainer doesn't support. For unsupported feature list and their corresponding issue,
please refer to [Unsupported Feature](https://github.com/greenplum-db/plcontainer/wiki/PLContainer-Unsupported-Features)

### Design

The idea of PL/Container is to use containers to run user defined functions. The current implementation assume the PL function definition to have the following structure:

```sql
CREATE FUNCTION dummyPython() RETURNS text AS $$
# container: plc_python_shared
return 'hello from Python'
$$ LANGUAGE plcontainer;
```

There are a couple of things you need to pay attention to:

1. The `LANGUAGE` argument to Greenplum is `plcontainer`

1. The function definition starts with the line `# container: plc_python_shared` which defines the name of runtime that will be used for running this function. To check the list of runtimes defined in the system you can run the command `plcontainer runtime-show`. Each runtime is mapped to a single docker image, you can list the ones available in your system with command `docker images`

PL/Container supports various parameters for docker run, and also it supports some useful UDFs for monitoring or debugging. Please read the official document for details.

## Debugging Locally

Sometimes, it is much eaiser to use a debugger like GDB to debug the clients. As they are typically run in containers, debugging info might not be loaded, such as the debug symbols of `libpython3.x.so`.

To make debugging easier, we can run the clients on the same host as the database backend process with the following steps:

1. Compile the client on host with bash command `cmake --build build/pyclient/ && make -C build/ install`.
1. Start a new database session and run SQL command `SET plcontainer.backend_type='process';`.
1. Start the client with `LOCAL_PROCESS_MODE=1 $GPHOME/bin/plcontainer_clients/py3client`.
1. Run the PL/Container UDF and the UDF will run in a process on host.

### Contributing
PL/Container is maintained by a core team of developers with commit rights to the [plcontainer repository](https://github.com/greenplum-db/plcontainer) on GitHub. At the same time, we are very eager to receive contributions and any discussions about it from anybody in the wider community.

Everyone interests PL/Container can [subscribe gpdb-dev](mailto:gpdb-dev+subscribe@greenplum.org) mailist list, send related topics to [gpdb-dev](mailto:gpdb-dev@greenplum.org), create issues or submit PR.

### License

The 'plcontainer' and 'pyclient' are distributed under the [BSD license](BSD LICENSE) and the [license described here](LICENSE).

---

With the exception of the 'rclient' source code is distributed under [GNU GPL v3](src/rclient/COPYING).
