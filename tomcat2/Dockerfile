FROM tomcat:9.0

COPY server.xml /usr/local/tomcat/conf/

# add context to /usr/local/tomcat/webapps
COPY sample.war /usr/local/tomcat/app1/sample.war
COPY sample.war /usr/local/tomcat/app2/sample.war

EXPOSE 9090 9091
