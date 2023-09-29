## Use Maven container to compile and build the package for PRG or WEU datacenters
FROM provide docker url/maven:3.6-jdk-11

## User docker-registry URL to get non DHL Custom docker base image from public hub
##FROM docker-registry.artifactory.dhl.com/adoptopenjdk/openjdk11:latest

## User CBJ specific URL for CloudBees Jenkins and other CBJ based Jenkins agents. 
##FROM docker.artifactory=cbj.com/adoptopenjdk/openjdk11:latest


RUN mkdir /appl
RUN addgroup --system app-user && adduser --system --shell /bin/false --group app-user

COPY target/esbproxy-*.jar /appl/esbproxy.jar

WORKDIR /appl

RUN chown -R app-user:app-user /appl

USER app-user

CMD [ "java", "-jar", "appJarFile.jar" ]

# Use the below debug the container 
# CMD ["sleep","3000"]
