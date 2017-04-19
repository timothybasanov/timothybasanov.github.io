# Dockerfile for running TeamCity.
# 
# Build an image locally (~10min):
# > docker build -t teamcity .
# Run an image with port 80 published (~1min):
# > docker run --rm -p 80:5000 --name teamcity teamcity

# Using Ubuntu with Oracle JDK 8
FROM java:8
MAINTAINER Timothy Basanov <tbasanov@evernote.com>
LABEL Description="TeamCity server with one build agent" Version="9.1.7"

# TeamCity data is stored in a volume to facilitate container upgrade
VOLUME /data/teamcity
ENV TEAMCITY_DATA_PATH=/data/teamcity

# Download and install TeamCity to /opt
ADD https://download.jetbrains.com/teamcity/TeamCity-9.1.7.tar.gz /opt/teamcity.tar.gz
RUN tar zxf /opt/teamcity.tar.gz -C /opt && rm /opt/teamcity.tar.gz

# Make Dokku happy by listening on :5000 and *not* using EXPOSE.
# > https://github.com/dokku/dokku/blob/master/docs/deployment/dockerfiles.md
RUN sed -e s/8111/${PORT:-5000}/ -i~ \
  /opt/TeamCity/conf/server.xml \
  /opt/TeamCity/buildAgent/conf/buildAgent.properties

# Start build agent and TeamCity server and wait for it to terminate
ENTRYPOINT /opt/TeamCity/buildAgent/bin/agent.sh start \
  && /opt/TeamCity/bin/teamcity-server.sh run






# Dockerfile for running TeamCity's build agent.
# 
# Build an image locally (~1min):
# > docker build -t teamcity-buildagent .
# Run an image with port 80 published (~1min):
# > docker run -e TEAMCITY_SERVER_URL=http://teamcity.yourinternaldomain.com --rm --name teamcity-buildagent teamcity-buildagent

# Using Ubuntu with Oracle JDK 8
FROM java:8
MAINTAINER Timothy Basanov <tbasanov@evernote.com>
LABEL Description="TeamCity build agent"

# Url to a server is very important
VOLUME /data/teamcity-buildagent
ENV TEAMCITY_SERVER_URL=${TEAMCITY_SERVER_URL:-http://teamcity.yourinternaldomain.com}

# Download and install TeamCity to /opt
ADD $TEAMCITY_SERVER_URL/update/buildAgent.zip /opt/buildAgent.zip
RUN unzip /opt/buildAgent.zip -d /opt/buildAgent && rm /opt/buildAgent.zip
RUN cp /opt/buildAgent/conf/buildAgent.dist.properties /opt/buildAgent/conf/buildAgent.properties
RUN chmod +x /opt/buildAgent/bin/agent.sh

# Configure build agent to connect to TeamCity server
RUN sed -e s!serverUrl=.*/!serverUrl=${TEAMCITY_SERVER_URL}! \
  -e s!workDir=.*!workDir=/data/teamcity-buildagent! \
  -i~ /opt/buildAgent/conf/buildAgent.properties

# Configure agent with a TeamCity url, no need to open any ports
ENTRYPOINT /opt/buildAgent/bin/agent.sh run
