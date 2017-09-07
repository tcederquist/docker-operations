# Docker Operational snippets
These are a few command line snippets for managing a docker host

* [Clean up](#clean-up)
* [Logs](#logs)
* [Build Patterns](#builds)
* [Stats from curl](#stats)
* [Repair](#repair)

## Clean-up
`Docker build` does not always clean up old images. If you build the same release over and over, artifacts and orphans with repository
`<none>` can be left behind. Here are some command line snippets for recovering lost space.

## Logs
Where are my logs? No need to redirect from docker logs yourcontainer, here you can see where the logs are written to disk:

      docker inspect --format='{{.LogPath}}' yourcontainer
      
Thanks to: https://stackoverflow.com/questions/41144589/how-to-redirect-docker-logs-to-a-single-file

### Linux
Docker maintenance for storage

        `docker rmi $(docker images | grep "^<none>" | awk '{print $3}')`
        docker rm $(docker ps -q -f 'status=exited')
        docker rmi $(docker images -q -f "dangling=true")
        docker volume rm $(docker volume ls -qf dangling=true)

### Windows
* `((docker images | Select-String -Pattern "none" -List -SimpleMatch)) | %{ $a=($_ -split '\s+')[2]; $b=($_ -split '\s+')[0]; docker rmi $a; }`
* For windows 10 users, cannot run both VMWare and Docker due to Hyper-V compatability issues. It is workable to multi-boot one with Hyper-V for Docker and one without for VMWare. Note: this must be run from command prompt with admin rights (powershell does not work!) Credit: Thanks to Scott Hanselman at this site for this snippet:  http://www.hanselman.com/blog/SwitchEasilyBetweenVirtualBoxAndHyperVWithABCDEditBootEntryInWindows81.aspx

        C:\>bcdedit /copy {current} /d "No Hyper-V" 
        The entry was successfully copied to {ff-23-113-824e-5c5144ea}. 
        C:\>bcdedit /set {ff-23-113-824e-5c5144ea} hypervisorlaunchtype off 
        The operation completed successfully.

## Builds
Shortcuts and patterns for building smaller images
### Start small
It is not possible to make an image smaller than the image its inherited from unless you break the inheritance chain.
Example, use alpine based images like FROM nginx:alpine or FROM openjdk:8-jre-alpine. Alpine is roughly 5MB + the tools but
based on busybox.Since it is limited by the c-runtime features of your container, some containers will not be able to use them.

### Why is my container 2GB?
Metrics are the key. Here is how you can examine which inherited part of your container suddenly become massive. In this case, 
three entries added 287M + 270M + 257M. If those are combined into one operation it might be possible to remove up to 500M for example.

        > docker history myimage   <-- a 1.049G container example
        IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
        066f5589e692        14 hours ago        /bin/sh -c #(nop)  ENTRYPOINT ["/docker-entry   0 B
        7d5b1ebd9dc7        14 hours ago        /bin/sh -c #(nop) COPY file:71ee243b599362409   177 B
        792881606be7        14 hours ago        /bin/sh -c #(nop)  EXPOSE 8181/tcp              0 B
        83cd8cc843c4        14 hours ago        /bin/sh -c #(nop) ADD dir:430db3744c6c6f0a324   33.33 MB
        bf877756b8e3        2 weeks ago         /bin/sh -c R -e 'install.packages("Rserve")'    1.881 MB
        d605258391e0        2 weeks ago         /bin/sh -c R -e 'install.packages("rJava")'     2.018 MB
        26e4eb25d2fa        2 weeks ago         /bin/sh -c R CMD javareconf                     6.527 kB
        c0f9f705ea70        2 weeks ago         /bin/sh -c ./usr/share/r/microsoft-r-open/ins   257.3 MB
        6acb256b2f8b        2 weeks ago         /bin/sh -c #(nop) ADD file:c09d6efa52f5d26167   270.2 MB
        06cc6b4cc435        2 weeks ago         /bin/sh -c #(nop) COPY file:e38ddf2e2fc02c193   2.165 kB
        88d0757f70a9        2 weeks ago         /bin/sh -c yum install -y  java-1.8.0-openjdk   287.6 MB
        ceff9463113a        3 weeks ago         /bin/sh -c #(nop)  MAINTAINER Tim Cederquist    0 B
        0584b3d2cf6d        4 weeks ago         /bin/sh -c #(nop)  CMD ["/bin/bash"]            0 B
        <missing>           4 weeks ago         /bin/sh -c #(nop)  LABEL name=CentOS Base Ima   0 B
        <missing>           4 weeks ago         /bin/sh -c #(nop) ADD file:54df3580ac9fb66389   196.5 MB
        <missing>           3 months ago        /bin/sh -c #(nop)  MAINTAINER https://github.   0 B

### One Run command
Each command in the build file results in a new intermediate image. Docker storage devices are 'additive' between instances meaning the 
image can only get larger once an instance in a build is inherited from the previous image. Running commands in one statement
will allow rm and other clean up commands to actually reduce the image size as they run on the one instance 
before the device is journaled. As soon as your build starts a new command, the size of the container 
can only get larger from that point forward. This sample has the added bonus of a pattern for installing packages in a
Jython purposed container.

        run apk add --update curl \
              && mkdir /usr/share/extras/ \
              && cd /usr/share/extras \
              && curl https://pypi.python.org/packages/ca/ca/c2436fdb7bb75d772d9fa17ba60c4cfded6284eed053a7274b2beb96596a/SQLAlchemy-1.1.4.tar.gz#md5=be5af1bebe595206b71b513a14836e4f > /usr/share/extras/SQLAlchemy-1.1.4.tar.gz \
              && curl http://peak.telecommunity.com/dist/ez_setup.py > ez_setup.py \ 
              && mkdir -p /usr/share/llf/ermasLogicRestPython_lib/Lib/site-packages/ \
              && java -jar /usr/share/llf/ermasLogicRestPython_lib/jython-standalone-2.7.1b3.jar ez_setup.py \
              && tar -xzvf /usr/share/extras/SQLAlchemy-1.1.4.tar.gz \
              && cd SQLAlchemy-1.1.4 \
              && java -jar /usr/share/llf/ermasLogicRestPython_lib/jython-standalone-2.7.1b3.jar setup.py install \
              && rm -fR /usr/share/extras \
              && apt-get clean
              
## Stats
Using curl and unix sockets to connect local with the following sample:

        docker stats  {optional conatiner list space dlm}
           Runs like 'watch' and privides all container high level stats
           
        curl --unix-socket /var/run/docker.sock http:/containers/e152a2f6b469/stats?stream=0
        stream is optional 1=continue polling, 0=pull once and close connection

## Repair
When things go badly...
### Out of space errors - corrupted storage
After expanding storage and need to let docker know that is 'okay' to start working again. The path to your docker installation which may be /var/lib/docker instead of my case /docker.

        service docker stop
        thin_check /docker/devicemapper/devicemapper/metadata
        thin_check --clear-needs-check-flag /docker/devicemapper/devicemapper/metadata
### Find the storage location of a docker container - when all else fails and you just need to get the content and start over! Use your container name, my was just a simple 'ermMysql' for example. Use your folder location (ie: /var/lib/docker instead of my /docker)

        docker inspect ermMysql | grep Source
        cd "/docker/volumes/380b947b222cb31cf8c09ba46a8fde736b713e0272cf85323196633cea94cd0c/_data"
### Copy files with archive settings to get the 999 docker uses for example (when you copy stuff back into a new container (yes this is hacking it a bit!)

        rsync -avzp /docker/volumes/380b947b222cb31cf8c09ba46a8fde736b713e0272cf85323196633cea94cd0c/_data/* /home/you/.
