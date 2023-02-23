# How To Write A Secure and Maintainable Dockerfile/Containerfile

This document consolidates tips for some of the best practices I have learned for creating ```Dockerfiles``` or ```Containerfiles``` that are highly maintainable. Because, like code, ```Dockerfiles/Containerfiles``` will need to change over time and therefore should be written in such a way that makes them easy to update in the future. It is also important that the images that they create are secure and do not contain unnecessary vulnerabilities that increase the attack surface for your application. The image produced should be as small as possible in size due to the fact that the image(s) will need to be stored remotely and transported in the network. it is therefore imperative that they are not blotted. Finally, the ```Dockerfile/Containerfile```, like any well-written code, should be easy to understand and use. 

1. Always use the most current release base upstream image for security reasons. [Red Hat recommends](https://developers.redhat.com/articles/2021/11/11/best-practices-building-images-pass-red-hat-container-certification) to;

    > * Use the latest release of a base image. This release should contain the latest security patches available when the base image is built. When a new release of the base image is available, rebuild the application image to incorporate the base image's latest release, because that release contains the latest fixes.
    > * Conduct vulnerability scanning. Scan a base or application image to confirm that it doesn't contain any known security vulnerabilities.
   

2. Use a specific Tag or version for your image instead of “latest”. This gives your image traceability, in the event of trouble shooting the running container, it is obvious the exact image used.

    ### Example:
    ```
    DO      nginx:1.23.1 
    DON'T   nginx:latest
    ```

3. For security purposes always ensure that your Images run as NON-ROOT by defining ```USER``` in your ```Dockerfile/Containerfile```, additionally set the permissions for the files and directories to the ```USER```. This is because the ```Docker``` deamon runs as root and therefore ```Docker``` images run as root by default. This means if a process in the container went rogue or is hijacked and gets access to the host, it will run with root access. This is certainly not secure. 
    > ```Podman``` however is daemonless and rootless by design and therefore more secure.
    
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

   ### Example: Podman On local machine

    > Podman integrates with multiple open-source scanning tools. For example, you can use;

      * [Synk](https://www.redhat.com/en/blog/using-snyk-and-podman-scan-container-images-development-deployment) 
      * [Trivy](https://aquasecurity.github.io/trivy/v0.37/docs/target/container_image/#podman)

    ### Example: Docker On your local machine

    > Docker also integragrates with its own plugin. Install the [plugin](https://docs.docker.com/engine/scan/) the run the below command:
    ``` 
    docker scan myappimage:1.0
    ```
    Automated scanning tools should also be implemented in the CI pipeline and on the enterprise registry. It is also recommended to have runtime scanning on applications deployed in case a vulnerability is uncovered in the future.

8. Order your Docker commands especially the ```COPY``` command in such a way that the files that change most frequently are at the bottom. This will speed up the build process. The reason for this is to take advantage of the Docker build process and speed up future builds. Each Docker build command creates a layer which is cached to be and reused in the next build, this is designed to speed up subsequent builds. The caveat is that, in the subsequent build, If a command encounters a change, ALL commands after that will run and recreate new layers and cached replacing the old ones even if they did not contain any changes. Having the most volatile COPY statements later in the ```Dockerfile/Containerfile``` maximizes on build caching.

9. Concatenate ```RUN``` commands to make your ```Dockerfile/Containerfile``` more readable and importantly, create fewer layers. Fewer layers mean a smaller container image. As mentioned previously, each ```RUN``` statement in the ```Dockerfile/Containerfile``` creates a layer that gets cached, concatening reduces the number of layers.

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

