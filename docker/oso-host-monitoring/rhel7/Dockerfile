FROM oso-rhel7-ops-base:latest
#     ___ ___ _  _ ___ ___    _ _____ ___ ___         
#    / __| __| \| | __| _ \  /_\_   _| __|   \        
#   | (_ | _|| .` | _||   / / _ \| | | _|| |) |       
#    \___|___|_|\_|___|_|_\/_/_\_\_|_|___|___/_ _____ 
#   |   \ / _ \  | \| |/ _ \_   _| | __|   \_ _|_   _|
#   | |) | (_) | | .` | (_) || |   | _|| |) | |  | |  
#   |___/ \___/  |_|\_|\___/ |_|   |___|___/___| |_|  
# 


# Pause indefinitely if asked to do so.
ARG OO_PAUSE_ON_BUILD
RUN test "$OO_PAUSE_ON_BUILD" = "true" && while sleep 10; do true; done || :

# PCP
##################
# install pcp-collector and it's dependencies, clean the cache.
RUN yum-install-check.sh -y pcp pcp-conf pcp-collector xz && yum clean all
# Run in the container as root - avoids PCP_USER mismatches
RUN sed -i -e 's/PCP_USER=.*$/PCP_USER=root/' -e 's/PCP_GROUP=.*$/PCP_GROUP=root/' /etc/pcp.conf

# Disable service advertising - no avahi daemon in the container
# (dodges warnings from pmcd attempting to connect during startup)
RUN . /etc/pcp.conf && echo "-A" >> $PCP_PMCDOPTIONS_PATH

# denote this as a container environment, for rc scripts
ENV PCP_CONTAINER_IMAGE pcp-collector
ENV NAME pcp-collector
ENV IMAGE pcp-collector
ENV PATH /usr/share/pcp/lib:/usr/libexec/pcp/bin:$PATH

# script to watch health of pmcd
ADD check-pmcd-status.sh /usr/local/bin/check-pmcd-status.sh
##################




RUN echo -e "\n\nalias oca='KUBECONFIG=/tmp/admin.kubeconfig oc '" >> /root/.bashrc
RUN echo "alias oadma='KUBECONFIG=/tmp/admin.kubeconfig oadm '" >> /root/.bashrc

# /usr/bin/oc workaround
# python-openshift-tools-monitoring-openshift depends on /usr/bin/oc
# since origin-clients and atomic-openshift-clients provide the binary
# for Origin/OpenShift respectively. yum isn't happy that a package named 'openshift'
# used to provide /usr/bin/oc, and 'openshift' has been replaced by
# 'atomic-openshift', but 'atomic-openshift' doesn't provide /usr/bin/oc.
# So just install atomic-openshift-clients before python-openshift-tools-monitoring-openshift
# until the yum repo is cleared of the older packages.
RUN yum-install-check.sh -y atomic-openshift-clients && yum clean metadata

RUN yum clean metadata && \
    yum-install-check.sh -y python-pip pcp pcp-conf pcp-testsuite \
        python-requests pyOpenSSL \
        python-openshift-tools \
        python-openshift-tools-monitoring-pcp \
        python-openshift-tools-monitoring-docker \
        python-openshift-tools-monitoring-zagg \
        python-openshift-tools-monitoring-openshift \
        python-openshift-tools-ansible \
        python-openshift-tools-web \
        openshift-tools-scripts-cloud-aws \
        openshift-tools-scripts-cloud-gcp \
        openshift-tools-scripts-monitoring-pcp \
        openshift-tools-scripts-monitoring-docker \
        openshift-tools-scripts-monitoring-aws \
        openshift-tools-scripts-monitoring-gcp \
        openshift-tools-scripts-monitoring-openshift \
        openshift-tools-scripts-monitoring-autoheal \
        pcp-manager pcp-webapi python-pcp \
        python-httplib2 \
        python-pyasn1 python-pyasn1-modules python-rsa \
        openvswitch \
        python-configobj \
        python-psutil \
        pylint \
        tito \
        python-devel \
        libyaml-devel \
        oso-simplesamlphp \
        python2-ruamel-yaml \
        python-pygithub \
        rpm-sign \
        createrepo \
        python2-boto3 \
        python-lxml \
        docker-python && \
    yum-install-check.sh -y gcloud python-uri-templates google-api-python-client python-oauth2client --disablerepo="oso-rhui-rhel-server-releases-optional" && \
    yum clean all

ADD urllib3-connectionpool-patch /root/
RUN yum-install-check.sh -y patch && yum clean all && cd /usr/lib/python2.7/site-packages/ && patch -p1 < /root/urllib3-connectionpool-patch

# make mount points for security update count check, and make a circular symlink because yum is dumb about its root
RUN mkdir -p /host \
             /var/local/hostpkg/etc/rhsm/ca \
             /var/local/hostpkg/etc/rpm \
             /var/local/hostpkg/etc/pki/entitlement \
             /var/local/hostpkg/etc/pki/rpm-gpg \
             /var/local/hostpkg/var/local \
             /var/local/hostpkg/var/cache/yum \
             /var/local/hostpkg/var/lib/yum && \
    ln -s /var/local/hostpkg /var/local/hostpkg/var/local/hostpkg

# Install python-hawkular-client
RUN yum-install-check.sh -y python-hawkular-client && yum clean all

# Ansible startup configuration playbook
ADD root /root

# FIXME: These are vendor libs that need to be packaged and installed via RPM.
ADD vendor/prometheus_client /usr/lib/python2.7/site-packages/prometheus_client/

# Create ops-runner.log file with proper permissions
RUN touch /var/log/ops-runner.log && chmod 664 /var/log/ops-runner.log

# Setup the AWS credentials file so that we can populate it on startup.
RUN mkdir -p /root/.aws && \
    touch /root/.aws/credentials && \
    chmod g+rw /root/.aws/credentials

# Add container-build-env-fingerprint
ADD container-build-env-fingerprint.output /etc/oso-container-build-env-fingerprint

# Add the start script and tell the container to run it by default
ADD start.sh /usr/local/bin/
CMD /usr/local/bin/start.sh
