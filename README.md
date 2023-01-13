# How To Write A Secure and Maintainable Dockerfile

This document consolidates tips for some of the best practices I have learned for creating Dockerfiles that is that are highly maintainable, because like code, Dockefiles will change over time and therefore should be written in such a way that makes them easy to update in the future. It is also important the images that are created as also secure ensuring that they do not contain unnessary vulnerabilites that increase the attack surface for your application. The Dockerfile should produced the smallest image size as possible, due to the fact that images will need to be stored and moved in the network, it is therefore impreative that they are not blotted. Finaly, the Dockerfile, like any well written code, should be easy to understand and use. 

1. Always use the most current release base upstream image for security reasons. [Red Hat recommends](https://developers.redhat.com/articles/2021/11/11/best-practices-building-images-pass-red-hat-container-certification) to;

    > * Use the latest release of a base image. This release should contain the latest security patches available when the base image is built. When a new release of the base image is available, rebuild the application image to incorporate the base image's latest release, because that release contains the latest fixes.
    > * Conduct vulnerability scanning. Scan a base or application image to confirm that it doesn't contain any known security vulnerabilities.
   

2. Use a specific Tag or version for your image instead of “latest”. This gives your image traceability incase of trouble shooting, everyone knows the exact image used.

    ### Example:
    ```
    USE        nginx:1.23.1 
    INSTEAD of nginx:latest
    ```

3. Ensure that your Images run as NON-ROOT by defining “USER” in your Dockerfile then set the permissions for the files and directories to the USER. The Docker deamon runs as root and by default Docker images run as Root. This means if a processes in the container went rogue and had access to the host, it will run with root access. This is certainly not secure. 
    
    ### Example, add USER as follows in your Dockerfile:
    ```
    USER 1001
    RUN chown -R 1001:0 /some/directory
        chmod -R g=u /some/directory
    ```

4. Always choose the smallest base images that do not contain the complete or full-blown OS with system utilities installed. You can install the tools and utilities needed for your application in the Dockerfile build. This will reduce possible vulnerabilities and attack surface in your image.

    ### Example :
    ```
    USE        nginx:1.23.1-alpine 
    INSTEAD of nginx:1.23-suse (NOT A REAL IMAGE)
    ```

5. Build images using [Multi-Stage Dockerfiles](https://docs.docker.com/develop/develop-images/multistage-build/) to keep image size small. For example, for a Java application use 1st stage to do the maven build and compile and then a 2nd stage to copy the binary artifact(s) and dependencies only and discarding all the unneeded artifacts OR for an Angular application, run the npm install and build in one stage and copy built artifacts in the next stage.

    ### Example: Open Liberty Java application ( ... indicates skipped configurations)
    ```
    FROM registry.access.redhat.com/ubi8/openjdk-8:latest as builder
    
    USER 0
    WORKDIR /tmp/app
    COPY src/ src/
    COPY pom.xml pom.xml
    RUN mvn clean package
    ...
    FROM quay.io/ohthree/open-liberty:22.0.0.4
    ...
    COPY --from=builder /tmp/app/src/main/liberty/config/server.xml /config/
    COPY --from=builder /tmp/app/target/*.war /config/apps/
    
    RUN  \
        chown -R 1001:0 /config && \
        chmod -R g=u /config

    # Run as non-root user
    USER 1001

    EXPOSE 9081
    ```



6. Use a ```.dockerignore``` file to ignore files you do not need added to the image.

7. Scan your images for known vulnerabilities. 
    ### Example: On your local machine run
    ``` 
    docker scan myappimage:1.0
    ```
    Automated scanning tools should also be implemented in the CI pipeline and or the enterprise registry.

8. Order your Docker commands especially the ```COPY``` command in such a way that the files that change the most frequent are at the bottom. This will speed up the build process. The reason being, each Docker build command creates a layer, this layer is cached and reused to speed up the next build. The caveat is that, in the subsequent build, If a command encounters a change, ALL the layers after that command will be rebuilt and recached even if they did not contain any changes. Having the most volatile COPY statements later in the Docker maximizes on build caching.

9. Concatenate RUN commands to create fewer layers thus smaller images and readable files. Each RUN statement in the ```Dockerfile``` creates a layer that cached, concatening reduces these layers.
    ### Example: INSTEAD OF
    ```
    ...
    RUN yum --disablerepo=* --enablerepo=”rhel-7-server-rpms”
    RUN yum update
    RUN yum instal -yl httpd
    ...
    ```
    
    ### DO THIS
    ```
    ...
    RUN yum --disablerepo=* --enablerepo=”rhel-7-server-rpms” && yum update && yum instal -yl httpd
    ...
    ```

    ### EVEN BETTER FOR READABILITY
    ```
    ...
    RUN yum --disablerepo=*  --enablerepo=”rhel-7-server-rpms” && \
        yum update && \
        yum instal -yl httpd
    ...
    ```

 
Sources:

[Docker website](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)

[Red Hat Developers Site](https://developers.redhat.com/articles/2021/11/11/best-practices-building-images-pass-red-hat-container-certification#)

[Red Hat DO 180 training](https://rhtapps.redhat.com/promo/course/do080?segment=4) 

