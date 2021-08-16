FROM jenkins/jenkins
ENV JAVA_OPTS -Djenkins.install.runSetupWizard=false
USER root


COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN /usr/local/bin/install-plugins.sh < /usr/share/jenkins/ref/plugins.txt

ENV JENKINS_USER admin
ENV JENKINS_PASS password

COPY default-user.groovy /usr/share/jenkins/ref/init.groovy.d/
