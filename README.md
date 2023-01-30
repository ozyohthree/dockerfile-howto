# How To Write A Secure and Maintainable Dockerfile

This document consolidates tips for some of the best practices I have gathered for creating Dockerfiles that are highly maintainable, secure, efficient and easy to use. Like code, Dockefiles will require refactoring over time and therefore should be written in such a way that makes them easy to update. It is also important that the images that are created are also secure ensuring that they do not contain unnessary vulnerabilites that increase the attack surface for your application. The image(s) created out of the Dockerfile should be minimal in size, due to the fact that they will need to be stored and moved in the network, it is therefore impreative that they are not blotted. Finaly, the Dockerfile, like any well written code, should be easy to understand and use. 
Here are the tips for creating such a Dockerfile;

1. Always use the most current release base upstream image for security reasons. [Red Hat recommends](https://developers.redhat.com/articles/2021/11/11/best-practices-building-images-pass-red-hat-container-certification) to;

    > * Use the latest release of a base image. This release should contain the latest security patches available when the base image is built. When a new release of the base image is available, rebuild the application image to incorporate the base image's latest release, because that release contains the latest fixes.
    > * Conduct vulnerability scanning. Scan a base or application image to confirm that it doesn't contain any known security vulnerabilities.
   

2. Use a specific Tag or version for your image instead of “latest”. This gives your image traceability, in the event of trouble shooting the running container, it is obvious the exact image used.

    ### Example:
    ```
    DO      nginx:1.23.1 
    DON'T   nginx:latest
    ```

3. For security purposes always ensure that your Images run as NON-ROOT by defining “USER” in your Dockerfile additionally set the permissions for the files and directories to the USER. This is because Docker images run as root by default, because the Docker deamon runs as root. This means if a processes in the container went rogue or is hijacked and gets access to the host, it will run with root access. This is certainly not secure. 
    
    ### Example, add USER as follows in your Dockerfile:
    ```
    ...
    USER 1001
    RUN chown -R 1001:0 /some/directory
        chmod -R g=u /some/directory
    ...
    ```

4. Always choose the smallest base images that do not contain the complete or full-blown OS with system utilities installed. You can install the specific tools and utilities needed for your application in the Dockerfile build. This will reduce possible vulnerabilities and attack surface of your image.

5. Build images using [Multi-Stage Dockerfiles](https://docs.docker.com/develop/develop-images/multistage-build/) to keep image size small. For example, for a Java application use one stage to do the your build and compile and another stage to copy the binary artifact(s) and dependencies, discarding all unneeded artifacts. Another example is for an Angular application, run the npm install and build in one stage and copy the built artifacts in the next stage.

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
    ### Example: On your local machine install the docker scan plugin then run
    ``` 
    docker scan myappimage:1.0
    ```
    Automated scanning tools should also be implemented in the CI pipeline and on the enterprise registry. It is also recommended to have runtime scanning on applications deployed in case a vulnerability is uncovered in the future.

8. Order your Docker commands, especially the ```COPY``` command, in such a way that the COPY commands that copy files that change the most frequent are at the bottom of the ```Dockerfile```. This will speed up the build process. This is because Docker ```build``` creates a layer for each command, the layers are cached and reused in future builds to speed up the process. However, If a command encounters a change in the subsequent build, say a file that is being copied is updated, ALL the commands that follow will run again and not use the cached layers, new layers will be created and to replace the existing ones regardless of whether the subsequent commands had any updates or not. Having the most volatile COPY statements later in the ```Dockerfile``` maximizes on build caching feature.

9. Concatenate RUN commands to create as few layers as possible, thus smaller images and readable files. Each RUN statement in the ```Dockerfile``` creates a layer that cached, concatening reduces these layers.
    ### Example: DON'T Do this
    ```
    ...
    RUN yum --disablerepo=* --enablerepo=”rhel-7-server-rpms”
    RUN yum update
    RUN yum instal -yl httpd
    ...
    ```
    
    ### DO this instead 
    ```
    ...
    RUN yum --disablerepo=* --enablerepo=”rhel-7-server-rpms” && yum update && yum instal -yl httpd
    ...
    ```

    ### Even better, DO this for readability
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

