# Dockerized Atlassian Bitbucket

This project is build by concourse.ci, see [our oss pipelines here](https://github.com/EugenMayer/concourse-our-open-pipelines)


## Supported tags and respective Dockerfile links

| Product |Version | Tags  | Dockerfile |
|---------|--------|-------|------------|
| Bitbucket | 5.8.2-latest | [see tags](https://hub.docker.com/r/eugenmayer/bitbucket/tags/) | [Dockerfile](https://github.com/blacklabelops/bitbucket/blob/master/Dockerfile) |

## Related Images

You may also like:

* [jira](https://github.com/EugenMayer/docker-image-atlassian-jira)
* [confluence](https://github.com/EugenMayer/docker-image-atlassian-confluence)
* [rancher catalog - corresponding catalog for bitbucket](https://github.com/EugenMayer/docker-rancher-extra-catalogs/tree/master/templates/bitbucket)

# Make It Short

Docker-CLI:

Just type and follow the manual installation procedure in your browser:

~~~~
docker-compose up
~~~~

> Point your browser to http://yourdockerhost:7990

# Setup

1. Start database server for Bitbucket.
2. Start Bitbucket.
3. Manual Bitbucket setup.

Firstly, start the database server for Bitbucket:

> Note: Change Password!

~~~~
$ docker run --name postgres_bitbucket -d \
    -e 'POSTGRES_DB=bitbucketdb' \
    -e 'POSTGRES_USER=bitbucketdb' \
    -e 'POSTGRES_PASSWORD=jellyfish' \
    -e 'POSTGRES_ENCODING=UTF8' \
    blacklabelops/postgres
~~~~

Secondly, start Bitbucket:

~~~~
$ docker run -d --name bitbucket \
	  --link postgres_bitbucket:postgres_bitbucket \
	  -p 7990:7990 blacklabelops/bitbucket
~~~~

>  Starts Bitbucket and links it to the postgresql instances. JDBC URL: jdbc:postgresql://postgres_bitbucket/bitbucketdb

Thirdly, configure your Bitbucket yourself and fill it with a test license.

Point your browser to http://yourdockerhost:7990

1. Choose `External` for `Database` and fill out the form:
  * Database Type: `PostgreSQL`
  * Hostname: `postgres_bitbucket`
  * Port: `5432`
  * Database name: `bitbucketdb`
  * Database username: `bitbucketdb`
  * Database password: `jellyfish`
2. Create and enter license information
3. Fill out the rest of the installation procedure.

# Embedded Elasticsearch

You need to use the environment variable BITBUCKET_EMBEDDED_SEARCH if you want to use the Embedded Elasticsearch:

Example:

~~~~
$ docker run -d --name bitbucket \
    -v your-local-folder-or-volume:/var/atlassian/bitbucket \
    -e "BITBUCKET_EMBEDDED_SEARCH=true" \
    -p 7990:7990 \
    blacklabelops/bitbucket /opt/bitbucket/bin/start-bitbucket.sh -fg
~~~~

> A separate java process for Elasticsearch will be started.

# Database Wait Feature

The bitbucket container can wait for the database container to start up. You have to specify the
host and port of your database container and Bitbucket will wait up to one minute for the database.

You can define the waiting parameters with the environment variables:

* `DOCKER_WAIT_HOST`: The host to poll. Mandatory!
* `DOCKER_WAIT_PORT`: The port to poll Mandatory!
* `DOCKER_WAIT_TIMEOUT`: The timeout in seconds. Optional! Default: 60
* `DOCKER_WAIT_INTERVAL`: The polling interval in seconds. Optional! Default:5

Example waiting for a postgresql database:

First start the polling container:

~~~~
$ docker run -d --name bitbucket \
    -e "DOCKER_WAIT_HOST=your_postgres_host" \
    -e "DOCKER_WAIT_PORT=5432" \
    -p 80:8090 blacklabelops/bitbucket
~~~~

> Waits at most 60 seconds for the database.

Start the database within 60 seconds:

~~~~
$ docker run --name postgres -d \
    --network jiranet \
    -v postgresvolume:/var/lib/postgresql \
    -e 'POSTGRES_USER=jira' \
    -e 'POSTGRES_PASSWORD=jellyfish' \
    -e 'POSTGRES_DB=jiradb' \
    -e 'POSTGRES_ENCODING=UNICODE' \
    -e 'POSTGRES_COLLATE=C' \
    -e 'POSTGRES_COLLATE_TYPE=C' \
    blacklabelops/postgres
~~~~

> Bitbucket will start after postgres is available!

# SSH Keys

If you need to use SSH Keys to authenticate Bitbucket to other services (eg, replicating to Github), put the entire contents of what you want to have in the .ssh directory in a directory called 'ssh' on your persistent volume.

When the container is started, the contents of `/var/atlassian/bitbucket/ssh` directory will be copied to `/home/bitbucket/.ssh`, and the permissions will be set to 700.

Example:

~~~~
$ docker run -d --name bitbucket \
    -v your-local-ssh-folder-or-volume:/var/atlassian/bitbucket/ssh \
    -e "BITBUCKET_PROXY_NAME=myhost.example.com" \
    -e "BITBUCKET_PROXY_PORT=443" \
    -e "BITBUCKET_PROXY_SCHEME=https" \
    blacklabelops/bitbucket
~~~~

> ssh keys will be copied and are available at runtime.

Alternatively copy the files in your running container and restart the container:

~~~~
# Creating the folder
$ docker exec bitbucket mkdir -p /var/atlassian/bitbucket/ssh
# Copy the keys
$ docker cp your-local-ssh-folder your-container-name:/var/atlassian/bitbucket/ssh
# Restart container
$ docker restart your-container-name
~~~~

# Embedded Backup And Restore Clients

This image has the Atlassian Bitbucket Backup Client included. The homepage can be found [HERE](https://marketplace.atlassian.com/plugins/com.atlassian.stash.backup.client/server/overview).The full documentation can be found [HERE](http://confluence.atlassian.com/display/BitbucketServer/Data+recovery+and+backups).

The client must be either executed inside the running container or inside a container that has the bitbucket
home directory as an attached volume.

Executing the backup client inside the running container:

~~~~
$ docker exec bitbucket java -jar /opt/backupclient/bitbucket-backup-client/bitbucket-backup-client.jar --help
~~~~

> Displays the help page inside the container with name `bitbucket`.

Executing the restore client inside the running container:

~~~~
$ docker exec bitbucket java -jar /opt/backupclient/bitbucket-backup-client/bitbucket-restore-client.jar --help
~~~~

> Displays the help page inside the container with name `bitbucket`.

The required parameters can be passed using environment variables:

* `BITBUCKET_HOME`: Bitbucket home directory, should be already set in the running container.
* `BITBUCKET_BASEURL`: Bitbucket base url.
* `BITBUCKET_USER`: Bitbucket admin user name.
* `BITBUCKET_PASSWORD`: Bitbucket admin password.

Example:

~~~~
$ docker exec -it \
    -e "BITBUCKET_BASEURL=http://localhost:7990" \
    -e "BITBUCKET_USER=youradmin" \
    -e "BITBUCKET_PASSWORD=yourpassword" \
    bitbucket bash
$ java -jar ${BITBUCKET_BACKUP_CLIENT}/bitbucket-backup-client.jar
~~~~

> Executes the backup client.

Running in a separate container:

~~~~
$ docker run --rm --name bitbucket_backup \
    -v your-local-bitbucket-folder-or-volume:/var/atlassian/bitbucket \
    -e "BITBUCKET_BASEURL=http://yourbitbucketserverurl:yourport" \
    -e "BITBUCKET_USER=youradmin" \
    -e "BITBUCKET_PASSWORD=yourpassword" \
    blacklabelops/bitbucket \
    java -jar /opt/backupclient/bitbucket-backup-client/bitbucket-backup-client.jar
~~~~

> Executing the backup client in a separate container that has access to bitbucket volume.

# JVM memory settings

By default Bitbucket starts with `-Xms=512m -Xmx=1g`. In some cases, this may not be enough to work with.

If needed, you can specify your own memory settings:

~~~~
$ docker run -d \
    -e "JVM_MINIMUM_MEMORY=2g" \
    -e "JVM_MAXIMUM_MEMORY=3g" \
    -p 7990:7990 \
    blacklabelops/bitbucket
~~~~

This will start Bitbucket with `-Xms=2g -Xmx=3g`.

# Proxy Configuration

You can specify your proxy host and proxy port with the environment variables BITBUCKET_PROXY_NAME and BITBUCKET_PROXY_PORT. The value will be set inside the Atlassian server.xml at startup!

When you use https then you also have to include the environment variable BITBUCKET_PROXY_SCHEME.

You can also specify the context path with BITBUCKET_CONTEXT_PATH.

Example HTTPS:

* Proxy Name: myhost.example.com
* Proxy Port: 443
* Poxy Protocol Scheme: https

Just type:

~~~~
$ docker run -d --name bitbucket \
    -e "BITBUCKET_PROXY_NAME=myhost.example.com" \
    -e "BITBUCKET_PROXY_PORT=443" \
    -e "BITBUCKET_PROXY_SCHEME=https" \
    blacklabelops/bitbucket
~~~~

> Will set the values inside the bitbucket.properties in /var/atlassian/bitbucket/bitbucket.properties

# Credits

This repo and project is based on the great work of

[blacklabelops/bitbucket](https://bitbucket.org/blacklabelops/bitbucket)


# References

* [Atlassian Bitbucket](https://www.atlassian.com/software/bitbucket)
* [Docker Homepage](https://www.docker.com/)
* [Docker Compose](https://docs.docker.com/compose/)
* [Docker Userguide](https://docs.docker.com/userguide/)
* [Oracle Java](https://java.com/de/download/)
