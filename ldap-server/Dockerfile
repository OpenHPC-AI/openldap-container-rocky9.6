# Use Rocky Linux 9.6 base image
FROM rockylinux:9.6

LABEL maintainer="satishg@cdac.in"

# ----------------------------------------------------------------------
# 1) LDAP folder redirection to external bind-mounted storage
# ----------------------------------------------------------------------
RUN rm -rf /etc/openldap && \
    mkdir -p /ldapdata/etc/openldap && \
    ln -s /ldapdata/etc/openldap /etc/openldap && \
    mkdir -p /ldapdata/var/lib/ldap && \
    ln -s /ldapdata/var/lib/ldap /var/lib/ldap

# ----------------------------------------------------------------------
# 2) Install Core Dependencies
# ----------------------------------------------------------------------
RUN dnf -y update && \
    dnf -y install epel-release && \
    dnf -y config-manager --set-enabled crb && \
    dnf -y install \
        wget vim procps-ng iproute \
        cyrus-sasl-devel libtool-ltdl-devel openssl-devel \
        libdb-devel make gcc tar autoconf gettext libtool \
        perl perl-devel openldap openldap-clients openldap-servers openldap-devel \
        nss-pam-ldapd oddjob openssh-server supervisor rsync sudo && \
    dnf clean all

# ----------------------------------------------------------------------
# 3) Configure OpenLDAP DB defaults
# ----------------------------------------------------------------------
#RUN cp /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG && \
RUN chown -R ldap:ldap /var/lib/ldap /etc/openldap /var/lib/ldap

# ----------------------------------------------------------------------
# 4) Copy required files
# ----------------------------------------------------------------------
COPY ./ldap_config /ldap_config
COPY ./initscripts/slapd /etc/init.d/slapd
COPY ./ldap_int-auth/nsswitch.conf /etc/nsswitch.conf
COPY ./ldap_int-auth/password-auth /etc/pam.d/password-auth
COPY ./ldap_int-auth/system-auth /etc/pam.d/system-auth
COPY ./etc_openldap_config/ldap.conf.template /ldap_config/
COPY ./ldap_run.sh /usr/local/bin/
COPY ./supervisord.conf /etc/supervisord.conf
COPY ./etc_openldap_config/schema/nsmattribute.ldif /etc/openldap/schema/
COPY ./migrationtools-47-37.el9.noarch.rpm /

RUN chmod +x /etc/init.d/slapd \
    && chmod +x /usr/local/bin/ldap_run.sh && \
    mv /ldapdata /ldapdata.NEEDINIT && \
    rpm -ivh /migrationtools-47-37.el9.noarch.rpm

COPY ./ldap_config/migrate_passwd.pl /usr/share/migrationtools/migrate_passwd.pl

# ----------------------------------------------------------------------
# 5) SSH Hardening
# ----------------------------------------------------------------------
RUN set -eux && \
    sed -i \
        -e 's/#PermitRootLogin yes/PermitRootLogin prohibit-password/' \
        -e 's/#Port 22/Port 2200/' \
        -e 's/#UseDNS yes/UseDNS no/' \
        /etc/ssh/sshd_config && \
    echo "StrictHostKeyChecking no" >> /etc/ssh/ssh_config

# ----------------------------------------------------------------------
# 6) Create xcatadmin User with sudo privileges
# ----------------------------------------------------------------------
RUN useradd -m -s /bin/bash xcatadmin && \
    echo 'xcatadmin:xcatadmin' | chpasswd && \
    echo 'root:ChangeMeNow!' | chpasswd && \
    usermod -aG wheel xcatadmin && \
    sed -i 's/^# %wheel ALL=(ALL) ALL/%wheel ALL=(ALL) ALL/' /etc/sudoers

# IMPORTANT: Do NOT delete /root/.ssh inside Dockerfile
# It breaks SSH host key initialization.

# ----------------------------------------------------------------------
# 7) Expose LDAP ports
# ----------------------------------------------------------------------
EXPOSE 389 636

# ----------------------------------------------------------------------
# 8) Entry point (runs root tasks, then switches to xcatadmin)
# ----------------------------------------------------------------------
ENTRYPOINT ["/usr/local/bin/ldap_run.sh"]
