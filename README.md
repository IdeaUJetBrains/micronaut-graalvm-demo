
<h2>Run/debug the Micronaut application in native (no JVM) mode inside docker container</h2>


We build our native application with maven in jetbrains/graalvm-debugger image. 
We use the result executable files in the `GraalVM Native Image` run configuration to debug.

1. Run Maven run configuration on Docker target 
   - Run on target: Docker
       - Image to pull: jetbrains/graalvm-debugger:21  or jetbrains/graalvm-debugger:17
       - Optional:
           - Run options: add `-p 8080:8080`
   - "Run": `package`
   - "Profiles": `graalvm`
   
3. Configure `GraalVM Native Image` run configuration:
    - Executable: `target/demo`
    - Symbol file: `target/demo.debug`
    - Run on target: select the same target, created on the (1) step (based on jetbrains/graalvm-debugger:17)

4. Set break point to `UserController` class, `random` get method 
5. Press "Debug" on the created `GraalVM Native Image` run configuration 
6. Go to http://localhost:8080/users/random to stop on this endpoint.


Troubleshooting
https://youtrack.jetbrains.com/issue/IDEA-331760/ It is not possible to set breakpoints for some classes/methods




//=================================================================================================================//
//=================================================================================================================//
OLD WAY (builds 233.*)
We build our native application  with maven using docker-native packaging. 
Then build docker debug graalvm image with coping there built classes and dependencies. 
Then we take from this docker image the "application" executable file and run debug using it on docker target.

1. Run maven target `mvn package -Dpackaging=docker-native` to build native application, profile: `graalvm`

2. Copy target/Dockerfile, name it `Debug.dockerfile` and place it into the same `target` folder.  
Change it:
   ```Dockerfile
   # GraalVM image based on some image, here on Oracle Linux 9
   ARG BASE_IMAGE="ghcr.io/graalvm/native-image-community:17-ol9"
   # because on dependencies on system libraies it's better to reuse same image for run
   ARG BASE_IMAGE_RUN="oraclelinux:9"
   
   FROM ${BASE_IMAGE} AS builder
   WORKDIR /home/app
   COPY classes /home/app/classes
   COPY dependency/* /home/app/libs/
   COPY *.args /home/app/graalvm-native-image.args
   #Here main class of your server application
   ARG CLASS_NAME="example.micronaut.Application"
   RUN native-image @/home/app/graalvm-native-image.args -H:Class=${CLASS_NAME} -g -H:Name=application -cp "/home/app/libs/*:/home/app/classes/"
   FROM ${BASE_IMAGE_RUN}
   ARG EXTRA_CMD=""
   RUN if [[ -n "${EXTRA_CMD}" ]] ; then eval ${EXTRA_CMD} ; fi
   
   COPY --from=builder /home/app/application /app/application
   #copy debug info of application
   COPY --from=builder /home/app/application.debug /app/application.debug
   
   #install gdbserver to debug
   RUN dnf install -y gdb-gdbserver
   ARG PORT=8080
   EXPOSE ${PORT}
   ```
3. Run "Build image" for `Debug.dockerfile`(click on the gutter ->Build image)
4. Extract application and application.debug into `target` or your output directory: 
    - select the built image in the Services view
    - click `Show Layers` button
    - click `Analyse image for more information`
    - select the layer with "COPY" of `application` file 
    - select this file in the tree and press "Download file" button on the toolbar
    - do the same for `application.debug` file
    - place the downloaded files into `target` folder

5. Make `application` executable:
   - select it in the project view-> 
      - run `chmod 755 target/application`

6. Configure `GraalVM Native Image` run configuration:
   - Executable: `target/application`
   - Use classpath module: `demo`
   - Run on target: Docker
     - Dockerfile: `target/Debug.dockerfile` 
     - Optional:
       - Image tag: gdbserver 
       - Run options: add `-p 8080:8080`
       
7. Set break point to `UserController` class, `random` get method
8. Press "Debug" on the created `GraalVM Native Image` run configuration
9. Go to http://localhost:8080/users/random to stop on this endpoint.
 


