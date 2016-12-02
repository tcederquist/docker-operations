# Docker Operational snippets
These are a few command line snippets for managing a docker host

* [Clean up](#clean-up)
* [Build Patterns](#builds)

## Clean-up
`Docker build` does not always clean up old images. If you build the same release over and over, artifacts and orphans with repository
`<none>` can be left behind. Here are some command line snippets for recovering lost space.

### Linux
`docker rmi $(docker images | grep "^<none>" | awk '{print $3}')`

### Windows
`((docker images | Select-String -Pattern "none" -List -SimpleMatch)) | %{ $a=($_ -split '\s+')[2]; $b=($_ -split '\s+')[0]; docker rmi $a; }`

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
