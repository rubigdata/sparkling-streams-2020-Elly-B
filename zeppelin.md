# Apache Zeppelin 4 RUBigData

## Setup

Checkout the assignment repo and change directory into it.

Create the docker image:

    docker build -t hadoop:devel .

Create and start the container:

    docker create --name snbz -p 9001:8080 -p 4040-4045:4040-4045 hadoop:devel
    docker start snbz

Open the notebook on [localhost:9001](http://localhost:9001/).

Now import the "Spark Streaming.zpln" notebook from the Web UI (the file in this directory);
first choose import, and then click on the new notebook link.

## Additional steps
   
Check what's going on:

    docker logs snbz

Start the stream:

    docker cp stream.py snbz:/
    docker exec snbz sh -c "python stream.py &"
	
