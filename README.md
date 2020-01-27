# Introduction

When you first start working with Hasura, you probably spun up a simple installation on a Platform as A Service (PAAS) such as Heroku or Digital Ocean. PAAS services are great, but once you start building something more serious, things become riskier and you're going to want the ability to develop in a non-production environment where you don't care if you mess up. So - you want to set up a local development environment, but what if you've already got a sophisticated Hasura database setup running in the Cloud? You don't want to have to start again and rebuild your entire Hasura database structure, let alone have to do that every. single. time you need a fresh (or updated) environment.

This is particularly important if you are designing your application in a way where it is configured using database values an simply migrating the schema from your production environment to a local one is not going to meet your requirements.

This guide will take you through the process of running Hasura locally on your machine in Docker. If you've never used Docker before, that's perfectly fine; this guide will actually be a great way for you to learn. From there, this guide will show you how to migrate an existing Hasura database structure and its data hosted on a PAAS service, and set that up in a Hasura instance running on your machine for local and offline development. Whether you want to migrate your Hasura instance from a PAAS to a local environment for the first time, or want to update your locally running Hasura instance with a clone of your latest production environment, this guide will show you how. 

Digital Ocean and Heroku, two very different PAAS services, both host their infrastructure in unique ways. This means that the way you will interact with your production Hasura instance will be very different depending on which service you are deployed on. In order to tailor the steps inside this guide with information specific to the PAAS service, this guide is the first of two on this topic. 

This first focusses on Hasura instances that are hosted on Digital Ocean. Another guide focussing on Hasura instances hosted on Heroku will be published soon.

### Note: Hasura CLI Migrations.

If you are **only** looking to migrate the structure of your DB and GraphQL, aka the DB schema and relationships, this can be done easily via the [Hasura CLI](https://docs.hasura.io/1.0/graphql/manual/migrations/manage-migrations.html) . This guide is specifically for situations where you want to migrate both your DB structure, **and your DB data**, from a cloud Hasura instance to a local Hasura instance running in Docker.

### **Disclaimer:**

This guide was created on a machine running OSX, so if you are using Linux you will need to ensure that specific commands are altered for your environment. 

Where commands have been used, I have given two examples of these commands. The first command will demonstrate the sections of the command that needs to be replaced with your own text of choosing - these sections will be symbols with text in-between, and will look like this: `${ TEXT_HERE }`. The second command will demonstrate how your command should look once you have replaced all of the necessary sections with your own text.

For example: 

    # Command with the variable you need to replace.
    cd ${ DIRECTORY_NAME }
    
    # Example of what the completed command will look like with the replaced variable.
    cd local-hasura-setup

# Section 1. Preparing a local environment.

### Step 1. Install the necessary tools on your local machine.

Before we get started, you will need to install **[Docker](https://docs.docker.com/install/)** and **[Docker-Compose](https://docs.docker.com/compose/install/)** on your local machine. 

### Step 2. Create a new directory to run your local environment.

In your terminal, create a new directory to store all the relevant files needed to run Hasura locally and move into it.

    mkdir ${ DIRECTORY_NAME }; cd ${ DIRECTORY_NAME }
    
    mkdir local-hasura-setup; cd local-hasura-setup

### Step 3. Get your Hasura Manifest of choosing.

First, create a `docker-compose.yaml` file in your directory in preparation for the manifest.

    touch docker-compose.yaml

Now, you will need to acquire the Hasura `docker-compose.yaml` file contents.  

The default setup to run Hasura is called the "docker-compose" manifest, which can be found [here](https://github.com/hasura/graphql-engine/tree/master/install-manifests/docker-compose). Copy the contents of this `docker-compose.yaml` file into your `docker-compose.yaml`.

If you have explicitly configured your production Hasura environment to run on a different `docker-compose.yaml` environment, for example with PgAdmin, ideally use the same Hasura manifest that matches the Hasura setup that is running on your production environment. To find the manifest for this, navigate to the [Hasura Docker Manifests on Github](https://github.com/hasura/graphql-engine/tree/master/install-manifests) and select a Docker-Compose Hasura setup of your choosing. Once you have selected your desired setup, navigate into the directory and copy the contents of the `docker-compose.yaml` file into your `docker-compose.yaml`.

### Step 4. Run Hasura locally in docker containers as a background daemon.

Now that you have the `docker-compose.yaml` file, you can begin running a local Hasura instance. In this guide I will be running Hasura as a background daemon with docker-compose, which is indicated by the `-d` flag. Running Docker as a daemon simply means that it runs as a background process on your system, rather than an active process in your terminal.

    docker-compose up -d

**Note:** 

If you would like to see the Hasura logs updating real-time in your terminal, rather than running Hasura as a background daemon, simply leave off the "-d" from the docker-compose command.
If you have already started Hasura as a daemon, you can shut it down by entering `docker-compose down`.

Alternatively, you can see the logs of any running Docker container that is running as a background daemon by running `docker logs ${ CONTAINER_ID }`. You can get your container ID by following Step 5.

### Step 5. Confirm that Hasura is running locally on Docker.

Regardless of the Docker-Compose Hasura manifest you are using, two different docker containers will have started from your `docker-compose.yaml` file after you began running Hasura locally. Confirm that these docker containers are alive by using the docker command `ps`. 

The `docker ps` command will print a table of all the docker containers that are currently running on your local machine. The table will list information under the columns "Container ID", "Image", "Command", "Created", "Status", "Ports", and "Names".

    docker ps

Under the columns "Names" and "Image" in your terminal output, you should see the following containers:

- `${ DIRECTORY_NAME }_graphql-engine_1` running image `hasura/graphql-engine:v1.0.0`
- `${ DIRECTORY_NAME }_postgres_1` running image `postgres`

If the containers are all running, you will now have access to the following endpoints:

- GraphQL endpoint: [http://localhost:8080/v1/graphql](http://localhost:8080/v1/graphql)
- Hasura Console: [http://localhost:8080/console](http://localhost:8080/console)
- All the Hasura API endpoints via the [Hasura CLI](https://docs.hasura.io/1.0/graphql/manual/hasura-cli/index.html#commands)

# Section 2. Cloning your production Hasura setup.

Now that you have Hasura running in your local environment, it's time to get the schema, relationships mapping, and data from your Hasura production environment. Digital Ocean and Heroku both host their infrastructure in unique ways, so the way you will access your production Hasura instance will be very different depending on which service you are deployed on. This first guide covers Hasura instances hosted on Digital Ocean; a second guide covering Hasura instances hosted on Heroku will be published soon.

**Note:** If you are hosting your Hasura instance on a service not listed here but you do know the IP address of your production server, you will be able to follow the same steps under Digital Ocean.

## Step 6. SSH into your production server.

To access your production environment in Digital Ocean, you will need to gain access to your remote machine that is running your production Hasura instance. 

**For this step, you will need a few things before you start.**

- GROUP_ROLE
    - Unless you have configured a new Group Role, you will be able to use the Group Role "root" to access your production server.
- IP_ADDRESS
    - The IP address of your production server.
    - **To get the IP address from a server running on Digital Ocean:**
        1. Log into your Digital Ocean account
        2. On the homepage UI of Digital Ocean, select the Project that you are running Hasura in.
        3. Under the Resources tab, you will see a list of Droplets that you are running. Look for the Droplet that has "hasura-graphql" at the beginning of its name.
        4. Once you have found your droplet, to the left of the droplet's name you will see an IP address. This is the IP address of your production server, so copy this number.
        ****

The `ssh` command is a program for opening a secure channel to connect and log into a specified remote machine, with the ability to execute commands on this remote machine. In this case, the remote machine will be your production server. 

Using the `ssh` command will request a password. You would have had to set up a password on your Hasura droplet in Digital Ocean when you created it - this is the password that you will enter to obtain remote access to your production server.

    ssh ${ GROUP_ROLE }@${ IP_ADDRESS }
    
    ssh root@127.0.0.1

**If this is the first time you are SSH'ing into your production server:**

SSH will raise a security alert (example below) and will ask for your permission to put the fingerprint of the remote machine into your local `~/.ssh/known_hosts` file. This is completely normal and nothing to worry about - SSH is letting you know that you have never connected to this server before. 

Type `yes` to proceed.

    The authenticity of host '127.0.0.1' can't be established.
    RSA key fingerprint is 12:23:34:45:56:a1:a2:67:78:a3:a4:aa:89:a5:a6:a7.
    Are you sure you want to continue connecting (yes/no)? yes
    Warning: Permanently added '127.0.0.1' (RSA) to the list of known hosts.

**If this is NOT the first time you are SSH'ing into your production server:**

You should **NOT** see any security alerts when you log in. 

If you do see a security alert like in the example below, contact your hosting provider to determine if the RSA key fingerprint sent by your production server really has changed.

If the RSA key fingerprint has been determined to have **NOT** changed, you will be able to proceed as normal. 

There are legitimate cases where this security alert can occur following a specific sequence of events; if you have SSH'd into a previously created instance that you have since decided to shut down, proceeded to create a new instance which was assigned the same previously used IP address, and then you have SSH'd into this newly created instance - you will receive the following message because your computer perceives that you are connecting to a new virtualized computer and not the same virtualized computer as before. 

    @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
    @       WARNING: POSSIBLE DNS SPOOFING DETECTED!          @
    @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
    The RSA host key for 127.0.0.1 has changed,
    and the key for the according IP address 127.0.0.1
    is unchanged. This could either mean that
    DNS SPOOFING is happening or the IP address for the host
    and its host key have changed at the same time.
    Offending key for IP in /Users/jessicascanlan/.ssh/known_hosts:10
    @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
    @    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
    @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
    IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
    Someone could be eavesdropping on you right now (man-in-the-middle attack)!
    It is also possible that the RSA host key has just been changed.
    The fingerprint for the RSA key sent by the remote host is
    12:23:34:45:56:a1:a2:67:78:a3:a4:aa:89:a5:a6:a7.
    Please contact your system administrator.
    Add correct host key in /Users/jessicascanlan/.ssh/known_hosts to get rid of this message.
    Host key verification failed.

## Step 7. Confirm that the production Hasura instance is on the server.

**On your remote server:** Because you now know what docker containers are part of Hasura, check that those docker containers are indeed running on the remote server.

    docker ps

In your terminal output, you should see the following containers:

- `${ DIRECTORY_NAME }_graphql-engine_1` running image `hasura/graphql-engine:v1.0.0`
- `${ DIRECTORY_NAME }_postgres_1` running image `postgres`

Take note of the name of the container running postgres.

## Step 8. Record the IP address and port number of the Postgres container.

**On your remote server:** In order to obtain a copy of your production environment, you will need to record the IP address and port number of the Postgres docker container that is connected to the Hasura GraphQL engine container. This can be found by inspecting the Postgres docker container, which you already have the container name of from Step 7 using `docker ps`.

    docker inspect ${ DIRECTORY_NAME }_postgres_1

The output from the `docker inspect` command should be an array with one object, which is comprised of a series of key value pairs. Towards the end of this output, look for the key "NetworkSettings". From here, you will find the port number under the "Ports" key. 

To find the IP address, you will have to look deeper into the object under the key "Networks", within which the **"IPAddress"** key which holds the IP address value you need. Don't be fooled by the similar-looking IP address under the "Gateway" key - you **DON'T** need this.

The `docker inspect` output should look somewhat like the following (values omitted unless explicitly needed for this guide):

    [
        {
            "Id": "",
            "Created": "",
            "Path": "",
            "Args": [],
            "State": {},
            "Image": "",
            "ResolvConfPath": "",
            "HostnamePath": "",
            "HostsPath": "",
            "LogPath": "",
            "Name": "/${ DIRECTORY_NAME }_postgres_1",
            "RestartCount": 0,
            "Driver": "",
            "Platform": "",
            "MountLabel": "",
            "ProcessLabel": "",
            "AppArmorProfile": "",
            "ExecIDs": ,
            "HostConfig": {},
            "GraphDriver": {},
            "Mounts": [],
            "Config": {},
            "NetworkSettings": {
                "Bridge": "",
                "SandboxID": "",
                "HairpinMode": ,
                "LinkLocalIPv6Address": "",
                "LinkLocalIPv6PrefixLen": ,
                "Ports": {
                    "5432/tcp": null
                },
                "SandboxKey": "",
                "SecondaryIPAddresses": ,
                "SecondaryIPv6Addresses": ,
                "EndpointID": "",
                "Gateway": "",
                "GlobalIPv6Address": "",
                "GlobalIPv6PrefixLen": ,
                "IPAddress": "",
                "IPPrefixLen": ,
                "IPv6Gateway": "",
                "MacAddress": "",
                "Networks": {
                    "${ DIRECTORY_NAME }_default": {
                        "IPAMConfig": ,
                        "Links": ,
                        "Aliases": [
                            "",
                            "postgres"
                        ],
                        "NetworkID": "",
                        "EndpointID": "",
                        "Gateway": "",
                        "IPAddress": "172.0.0.1",
                        "IPPrefixLen": ,
                        "IPv6Gateway": "",
                        "GlobalIPv6Address": "",
                        "GlobalIPv6PrefixLen": ,
                        "MacAddress": "",
                        "DriverOpts": 
                    }
                }
            }
        }
    ]

## Step 9.  Get a copy of your production Hasura database data.

**For this step, you will need a few things before you start.**

- IP_ADDRESS_OF_POSTGRES_CONTAINER
    - This is the IP address of your PostgreSQL docker container, which you have obtained in Step 8 using the `docker inspect` command.
- PORT_NUMBER_OF_POSTGRES_CONTAINER
    - This is the port number of your PostgreSQL docker container, which you have obtained in Step 8 using the `docker inspect` command.
- POSTGRES_USERNAME
    - The default value of the username is "postgres". Unless you have intentionally set up a unique username for the Hasura PostgreSQL database running inside the docker container, this will be configured according to the defaults specified by PostgreSQL themselves inside their docker image.
- FILE_NAME
    - The name that you want your production Hasura clone file to take.

**On your remote server:** Now that you have the IP address of your Hasura postgres container, you can obtain a copy of your Hasura production environment and save it to a pgsql file. This copy will include your production schema, relationship mapping, and all data.

The `pg_dump` command takes a clone of your PostgreSQL database. Because the `-Fc` flags are being used, the `pg_dump` command will output the contents of the database in custom format, which is being directed into a .pgsql file via the `>` command.

    pg_dump -h ${ IP_ADDRESS_OF_POSTGRES_CONTAINER } --port ${ PORT_NUMBER_OF_POSTGRES_CONTAINER } -U ${ POSTGRES_USERNAME } -Fc > ${ FILE_NAME }.pgsql
    
    pg_dump -h 172.0.0.1 --port 5432 -U postgres -Fc > hasura_production_clone.pgsql

## Step 10. Confirm that you created the production Hasura clone.

**On your remote server:** To check that Step 9 has successfully cloned the Hasura production environment, first validate that the pgsql file exists.

    ls -a

The output from this should include this file:

- `${ FILE_NAME }.pgsql`

## Step 11. Confirm that there is data in the production Hasura clone file.

**On your remote server:** To finish confirming that the Hasura production environment clone was a success, confirm that there is actually data in the pgsql file.

    cat ${ FILE_NAME }.pgsql
    
    cat hasura_production_clone.pgsql

The output from the `cat` command should print the contents of the Hasura production environment clone into your terminal. If you see any errors, or if the file is empty, go back to Step 9 and repeat again.

## Step 12. Collect the file path to your production Hasura clone file.

**On your remote server:** Now that you have validated that the Hasura production environment clone was a success, use the `pwd` command and collect the file path to the pgsql file so that you can transfer it to your local environment. Make sure you save the output of this command so that you can use it later.

    pwd

## Step 13. Collect the file path to your local environment.

**On your local machine:** Keeping the remote server session open, open a new tab in your terminal window to run the following commands locally. You will want to ensure that you are inside the directory created in Step 2 under Section 1. 

    cd ${ DIRECTORY_NAME }
    
    cd local-hasura-setup

Once you are inside the directory containing the files for your local Hasura environment, type the `pwd` command into the terminal and copy the resulting output. You will need the file path to this directory so that you can transfer the pgsql file into it, so make sure you save the output of this command to use later.

    pwd

## Step 14. Copy the production Hasura clone file to your local directory.

**For this step, you will need a few things before you start.**

- GROUP_ROLE
    - Unless you have specifically changed the group role when you initially configured your docker files, this variable will be `root`.
- IP_ADDRESS_OF_PRODUCTION_SERVER
    - This IP is the address of your production server. You obtained this in Step 6.
- FULL_FILE_PATH_TO_PGSQL_FILE_ON_PRODUCTION_SERVER
    - This file path is the path to the pgsql file you created on your production server. You obtained this in Step 12 using the `pwd` command.
- FULL_FILE_PATH_TO_LOCAL_ENVIRONMENT_DIRECTORY
    - This file path is the path to the directory containing your **local** environment files. You obtained this in Step 13 using the `pwd` command.

**On your local machine:**  Copy the pgsql file from the remote production server to the directory on your local machine using `scp`. The SCP bash command is similar to the SSH command but is only for explicitly copying files between servers, rather than logging into a remote machine.

    scp ${ GROUP_ROLE }@${ IP_ADDRESS_OF_PRODUCTION_SERVER }:${ FULL_FILE_PATH_TO_PGSQL_FILE_ON_PRODUCTION_SERVER } ${ FULL_FILE_PATH_TO_LOCAL_ENVIRONMENT_DIRECTORY }
    
    scp root@127.000.000.01:/root/hasura_production_clone.pgsql /Users/jessicascanlan/local-hasura-setup/

You will now have copied the pgsql file from your remote production server to your local machine.

# Section 3. Seeding your local Hasura environment.

### Step 15. Copy the production Hasura clone file into your local PostgreSQL

**For this step, you will need a few things before you start.**

- FULL_FILE_PATH_TO_PGSQL_FILE_IN_LOCAL_DIRECTORY
    - This file path is the path to the pgsql file inside your local directory, where you are also storing your Hasura `docker-compose.yaml` file. To get your full file path: ensure you are currently in your local directory, type `pwd` and collect the output, then copy the pgsql file name onto the end of that output.
- NAME_OR_IP_ADDRESS_OF_POSTGRESQL_DOCKER_CONTAINER
    - To get your local PosgreSQL docker container name, use the command `docker ps` and copy the value under "Name" or "Container ID".
- INTENDED_FILE_PATH_DESTINATION_ON_POSTGRESQL_DOCKER_CONTAINER
    - This is the file path that will determine where your pgsql file is copied to inside your docker container. It is recommended that this file path is kept as the root of your docker container, so that importing the contents into your local Hasura instance is kept simple. The value that will represent the root of your docker is `/`.
- INTENDED_NAME_OF_PGSQL_FILE
    - This is the name that your pgsql file will take once it is copied inside your docker container. It is recommended to keep this name the same as the original pgsql name for simplicity.

**On your local machine:**  Your docker containers are running on separate process IDs, which means that you need to treat your docker containers like they are an entirely separate machine that has no access to any of your local files unless you move them across onto it. To get the data from your production Hasura clone file into your local environment setup, you now need to copy the pgsql file from your local directory into the docker container that is running your local (but currently empty) Hasura PostgreSQL database. 

The `docker cp` command copies files between the local filesystem and a container.

    docker cp ${ FULL_FILE_PATH_TO_PGSQL_FILE_IN_LOCAL_DIRECTORY } ${ NAME_OR_IP_ADDRESS_OF_POSTGRESQL_DOCKER_CONTAINER }:${ INTENDED_FILE_PATH_DESTINATION_ON_POSTGRESQL_DOCKER_CONTAINER }${ INTENDED_NAME_OF_PGSQL_FILE }
    
    docker cp /Users/jessicascanlan/local-hasura-setup/hasura_production_clone.pgsql local-hasura-setup_postgres_1:/hasura_production_clone.pgsql
    

### Step 16. Import your production Hasura data into your local PostgreSQL

**For this step, you will need a few things before you start.**

- NAME_OR_IP_ADDRESS_OF_POSTGRESQL_DOCKER_CONTAINER
    - This value should be the exact same value as NAME_OR_IP_ADDRESS_OF_POSTGRESQL_DOCKER_CONTAINER in Step 15.
- DATABASE_NAME
    - The default value of the database name is "postgres". Unless you have intentionally set up a unique database name for the Hasura PostgreSQL database running inside the docker container, this will be configured according to the defaults specified by PostgreSQL themselves inside their docker image.
- DATABASE_USERNAME
    - The default value of the username is "postgres". Unless you have intentionally set up a unique username for the Hasura PostgreSQL database running inside the docker container, this will be configured according to the defaults specified by PostgreSQL themselves inside their docker image.
- FILE_PATH_DESTINATION_TO_PGSQL_FILE_INSIDE_THE_CONTAINER
    - This value should be the exact same value as INTENDED_FILE_PATH_DESTINATION_ON_POSTGRESQL_DOCKER_CONTAINER in Step 15.
- NAME_OF_PGSQL_FILE_INSIDE_THE_CONTAINER
    - This value should be the exact same value as INTENDED_NAME_OF_PGSQL_FILE in Step 15.

**On your local machine:** You're nearly there! It's time to import your production Hasura schema, relationship mapping, and all data into your local environment setup. This can be done by getting the Hasura docker container running PostgreSQL to import the contents of the pgsql file that you have now copied inside that container.

In the code below, there are two commands running here at the same time. The `pg_restore` command is a PostgreSQL utility command that reconstructs a database from a pgsql file. The `docker exec` command takes commands as input (for example, the pg_restore command) and executes them inside the working directory of the specified docker container.

    docker exec ${ NAME_OR_IP_ADDRESS_OF_POSTGRESQL_DOCKER_CONTAINER } pg_restore \
        --dbname=${ DATABASE_NAME } \
        --verbose \
        -U ${ DATABASE_USERNAME } \
        ${ FILE_PATH_DESTINATION_TO_PGSQL_FILE_INSIDE_THE_CONTAINER }${ NAME_OF_PGSQL_FILE_INSIDE_THE_CONTAINER }
    
    docker exec local-hasura-setup_postgres_1 pg_restore \
        --dbname=postgres \
        --verbose \
        -U postgres \
        /hasura_production_clone.pgsql

### Congratulations! You're good to go!

You have now set up your local Hasura environment using the schema, relationship mappings, and data, from your production Hasura environment. You will now be able to use and develop inside your local environment, knowing that your local environment is as close to your production environment as possible.

If you have run your docker containers as a daemon (ie a background process) like specified in Step 4, your Hasura docker containers will keep running in the background unless you specifically tell them to shut down. Once you are ready to shut them down, enter: `docker-compose down`.

As long as you do not remove the affiliated docker volumes, you will not lose any data by shutting these docker containers down and restarting them when needed. To see your affiliated Hasura docker volumes, use the `docker volume list` command to display all docker volumes, and search for a volume name that has a suffix of "_db_data".

# Section 4. Start over with a clean local Hasura environment.

Need to delete your local Hasura environment and start again with a clean slate? Or just want to update your local Hasura environment with your latest Hasura production clone? Cleaning up your current local Hasura environment is just as simple as creating one.

### Step 17. Move into your local Hasura directory.

Ensure that you are currently in the directory containing your local Hasura files (such as the Hasura `docker-compose.yaml` file in Step 2).

### Step 18. Shut down your local Hasura instance.

Shut down your local Hasura instance by running the `docker-compose down` command. This command will stop all docker containers that are affiliated with the `docker-compose.yaml` file in this directory, remove them, and remove any affiliated networks.

    docker-compose down

The output from this command is your `docker-compose down` logs, which will contain a list of any errors, orphaned containers, or orphaned resources.

### Step 19. Confirm that your local Hasura containers are not running.

It is import to confirm that the docker containers associated with your local Hasura environment are no longer running. You can use the `docker ps` command to check if they are still listed as running containers.

    docker ps

If the Hasura docker containers have not shut down, check the logs to see if they are listed as an orphaned container or orphaned resource. You will need to read through the `docker-compose down` logs that were output to your terminal in Step 18, taking note of listed orphaned items. 

For any containers that have not been removed, you can shut these containers down manually using the `docker kill` command.

    docker kill ${ NAME_OR_CONTAINER_ID_OF_CONTAINER }
    
    docker kill local-hasura-setup_graphql-engine_1

### Step 20. List Hasura affiliated Docker volumes.

Docker volumes are used to persist data between shutting down and starting up docker containers. While this is great for maintaining a local environment setup with ease, you will need to list all the docker volumes that are affiliated with Hasura before you can start fresh. The names of any created volumes will be found in your `docker-compose.yaml` [manifest](https://github.com/hasura/graphql-engine/blob/f9daaf40d36afd201ae59d0e633a115542b4cf2f/install-manifests/docker-compose/docker-compose.yaml) under the "Volumes" key, such as "db_data".

The `docker volume list` command can be used to list all docker volumes that exist on your local machine. 

    docker volume list

### Step 21. Remove Hasura affiliated Docker volumes.

To ensure that no data is persisted when you rebuild your local Hasura environment again, you will need to remove all affiliated Hasura docker volumes.

The command `docker volume rm` will permanently remove specified docker volumes.

    docker volume rm ${ HASURA_DOCKER_VOLUME_NAME }
    
    docker volume rm local-hasura-setup_db_data

### All done! Now you have a clean slate to work with.

You have now shut down your existing local Hasura docker containers and removed all affiliated docker volumes. You will now be able to create a local Hasura setup environment again, knowing that your local environment has been cleaned of its previous schema, relationship mappings, and data, completely reset to a fresh state for you to work with.

If you would like to update your local Hasura environment with the latest schema, relationship mappings, and data from your production Hasura environment, you are now able to run the command `docker-compose up` and then follow the steps from Section 2. This means that you will use the same Hasura manifest that is in your `docker-compose.yaml` file and only the local Hasura instance itself will be updated.

If you would prefer to set up an entirely new local Hasura environment using a different Hasura manifest from the one currently in your `docker-compose.yaml` file , begin again from the start of this document under Section 1 at Step 3.

---

# Resources

## Documentation guides

- [Hasura Client documentation](https://www.notion.so/etalia04/Hasura-85a313a3f1834036bb03d0488ef1d9c1)
- [Docker documentation](https://www.notion.so/etalia04/Hasura-85a313a3f1834036bb03d0488ef1d9c1)
- [Docker Compose documentation](https://www.notion.so/etalia04/Hasura-85a313a3f1834036bb03d0488ef1d9c1)
- [PostgreSQL documentation](https://www.notion.so/etalia04/Hasura-85a313a3f1834036bb03d0488ef1d9c1)

## Commands Glossary

### Shell commands

- `pwd`
- `ssh`
- `>`
- `cat`
- `scp`
- `ls`

### Docker commands

- `docker ps`
- `docker cp`
- `docker exec`
- `docker volume list`
- `docker volume rm`
- `docker kill`
- `docker inspect`

### Docker-Compose commands

- `docker-compose up`
- `docker-compose down`

### PostgreSQL commands

- `pg_restore`
- `pg_dump`
- `pg_restore`

## Programs to Install / Download

- [**Docker**](https://docs.docker.com/install/)
- **[Docker-Compose](https://docs.docker.com/compose/install/)**
