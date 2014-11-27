---
layout: post
title:  "Build Shibboleth SP from source"
date:   2014-04-26 10:07:11
categories: shibboleth saml
---

Shibboleth SP build from source HOWTO
=====================================

This HOWTO documents how to build and install a Shibboleth SP on a Centos 6 machine.

	$ hostname
	shibsp-patch.localdomain
	$ uname -a
	Linux shibsp-patch.localdomain 2.6.32-431.1.2.0.1.el6.x86_64 #1 SMP Fri Dec 13 13:06:13 UTC 2013 x86_64 x86_64 x86_64 GNU/Linux
	$ more /etc/centos-release 
	CentOS release 6.5 (Final)


##References

- https://wiki.shibboleth.net/confluence/display/SHIB2/NativeSPLinuxSourceBuild

Make sure you have enough memory installed on your machine for building all software. 256 MB will not suffice (as I noticed when compiling openSAML).
If you get obscure error messages like:

	g++: Internal error: Killed (program cc1plus)

you did not install enough.
I installed 4 GB of memory, which should be enough.

###Install development tools:

	sudo yum groupinstall 'Development Tools'

###Install dependencies from CentOS repositories:

	sudo yum install openssl openssl-devel
	sudo yum install boost boost-devel
	sudo yum install log4cpp log4cpp-devel	# TODO: check if really necessary?
	sudo yum install curl curl-devel
	sudo yum install httpd httpd-devel
	sudo yum install httpd-tools	# TODO: check if really necessary?

Other dependencies are installed from source. Note that all components are installed into
`/opt/shibboleth-sp`

###Install log4shib from source:

	wget http://shibboleth.net/downloads/log4shib/latest/log4shib-1.0.8.tar.gz
	tar xzf log4shib-1.0.8.tar.gz 
	cd log4shib-1.0.8
	./configure --disable-static --disable-doxygen --prefix=/opt/shibboleth-sp
	make
	sudo make install
	cd ..

###Install xerces from source:

	wget http://www.apache.org/dist/xerces/c/3/sources/xerces-c-3.1.1.tar.gz
	tar xzf xerces-c-3.1.1.tar.gz 
	cd xerces-c-3.1.1
	./configure --prefix=/opt/shibboleth-sp --disable-netaccessor-libcurl
	make
	sudo make install
	cd ..

###Install xml-security from source:

(download from an appropriate mirror)

	wget http://mirror.tcpdiag.net/apache/santuario/c-library/xml-security-c-1.7.2.tar.gz
	tar xzf xml-security-c-1.7.2.tar.gz 
	cd xml-security-c-1.7.2
	./configure --without-xalan --disable-static --prefix=/opt/shibboleth-sp --with-xerces=/opt/shibboleth-sp
	make
	sudo make install
	cd ..

###Install xmltooling from source:

	wget http://shibboleth.net/downloads/c++-opensaml/latest/xmltooling-1.5.3.tar.gz
	tar xzf xmltooling-1.5.3.tar.gz 
	cd xmltooling-1.5.3
	./configure --with-log4shib=/opt/shibboleth-sp --prefix=/opt/shibboleth-sp -C
	make
	sudo make install
	cd ..

###Install openSAML from source:

	wget http://shibboleth.net/downloads/c++-opensaml/latest/opensaml-2.5.3.tar.gz
	tar xzf opensaml-2.5.3.tar.gz 
	cd opensaml-2.5.3
	./configure --with-log4shib=/opt/shibboleth-sp --prefix=/opt/shibboleth-sp -C
	make
	sudo make install
	cd ..

###Install Shibboleth from source:

	wget http://shibboleth.net/downloads/service-provider/latest/shibboleth-sp-2.5.3.tar.gz
	tar xzf shibboleth-sp-2.5.3.tar.gz 
	cd shibboleth-sp-2.5.3
	./configure --with-log4shib=/opt/shibboleth-sp --enable-apache-22  --prefix=/opt/shibboleth-sp --with-apxs2=/usr/sbin/apxs 
	make
	sudo make install
	cd ..


## Configure

Configure Shibboleth by enabling the sample apache config:

	ln -s /opt/shibboleth-sp/etc/shibboleth/apache22.config /etc/httpd/conf.d/shib.conf

This triggers shibboleth authentication for the `/secure` path. Create this location and a test page within the document root:

	mkdir /var/www/html/secure
	echo secure > /var/www/html/secure/index.html

This should now trigger an authentication request to the sample IDP (redirecting to server idp.example.org). You can test this by inspecting the Location header returned using:

	$ curl http://shibsp-patch.localdomain/secure -I

To test with one of your own IDPs, you need to exchange SAML 2.0 metadata. 

Your SP metadata is available at the URL:

	http://shibsp-patch.localdomain/Shibboleth.sso/Metadata

Donwload your IDPs metadata, and store it in the file `/opt/shibboleth-sp/etc/shibboleth/partner-metadata.xml`.

Edit the file `/opt/shibboleth-sp/etc/shibboleth/shibboleth2.xml` and change the entityID of your IDP in the `SSO` element, i.e. change

	<SSO entityID="https://idp.example.org/idp/shibboleth"

into the entityID of your IDP, for instance:

	<SSO entityID="http://localhost:8080/saml2/idp/metadata.php"          

## Test

	echo "export LD_LIBRARY_PATH=/opt/shibboleth-sp/lib" >> /etc/sysconfig/httpd

You may need to change your firewall settings. For testing, we'll just disable the firewall:

	sudo /etc/init.d/iptables stop

Now start both apache and shibd:

	sudo /etc/init.d/httpd start
	/opt/shibboleth-sp/sbin/shibd

Verify if everything is working:

	curl http://localhost/Shibboleth.sso/Status


## Patch

Now say you want to build a modified version of Shibboleth SP. For example, we want to extend the session lifetime without letting the IDP shorten it. 

This can be implemented by applying a simple patch file:

	--- SAML2Consumer.cpp	2012-07-23 20:08:22.000000000 +0000
	+++ /home/centos/SAML2Consumer.cpp	2014-04-04 12:04:29.457927223 +0000
	@@ -423,7 +423,7 @@
	     if (sessionExp == 0)
	         sessionExp = now + lifetime.second;     // IdP says nothing, calulate based on 	SP.
	     else
	-        sessionExp = min(sessionExp, now + lifetime.second);    // Use the lowest.
	+        sessionExp = max(sessionExp, now + lifetime.second);    // Use the highest (HACK 	ALERT).
	 
	     const AuthnContext* authnContext = ssoStatement->getAuthnContext();
     
     
   
In your `/opt/shibboleth-sp/etc/shibboleth/shibboleth2.xml` file, define the desired lifetime in the `Session` element's `lifetime` attribute:   

	<Sessions lifetime="288000" timeout="3600" relayState="ss:mem"

Stop shibd and apache:

	/etc/init.d/httpd stop

apply the patch, and rebuild:

	cd shibboleth-sp-2.5.3/shibsp/handler/impl/
	patch SAML2Consumer.cpp < ~/SAML2Consumer.patch
	cd -
	make clean
	./configure --with-log4shib=/opt/shibboleth-sp --enable-apache-22  --prefix=/opt/shibboleth-sp --with-apxs2=/usr/sbin/apxs
	make
	sudo make install

Start shibd and apache again to test.

	sudo /etc/init.d/httpd start

That's all!

# APPENDIX: patch src RPM

# using a new install from a disk image

For instance, using a netinstall CD image:

	wget http://ftp.tudelft.nl/centos.org/6.5/isos/x86_64/CentOS-6.5-x86_64-netinstall.iso

# using vagrant

Add a suitable box from decent source:

	vagrant box add centos-base http://developer.nrel.gov/downloads/vagrant-boxes/CentOS-6.5-x86_64-v20140311.box

launch a new VM:

    mkdir shibsp
    cd shibsp
    vagrant init centos-base
    vagrant up
  
Upload the patch file to the new VM:

	scp -P 2222 shibboleth-sp-2.5.3-oc.patch vagrant@localhost:.

When done, download the new rpm:

	scp -P 2222 vagrant@localhost:src/rpm/RPMS/x86_64/shibboleth-2.5.3-1.1.el6.x86_64.rpm .

and you can safely destroy the VM:

	vagrant destroy
	
# building the rpm

Install developer tools:

	sudo yum groupinstall 'Development Tools'

setup for a local build:

	echo "%_topdir $HOME/src/rpm" > .rpmmacros
	mkdir -p src/rpm
	cd src/rpm
	mkdir BUILD RPMS SOURCES SPECS SRPMS
	mkdir RPMS/{i386,i486,i586,i686,noarch,athlon}
	cd -
	
Download and install the srpm file:

	wget http://shibboleth.net/downloads/service-provider/latest/SRPMS/shibboleth-2.5.3-1.1.el5.src.rpm
	rpm -ivh ~/shibboleth-2.5.3-1.1.el5.src.rpm 

the sources will now be in `src/rpm/SOURCES/shibboleth-sp-2.5.3.tar.gz`. Extract them into the BUILD directory and apply patches:

	cd src/rpm/BUILD/
	tar xf ../SOURCES/shibboleth-sp-2.5.3.tar.gz 
	patch shibboleth-sp-2.5.3/shibsp/handler/impl/SAML2Consumer.cpp < ~/shibboleth-sp-2.5.3-oc.patch 
	cd -

Add the shibboleth repository so we can install dependencies:

	cd /etc/yum.repos.d/
	sudo wget http://download.opensuse.org/repositories/security://shibboleth/CentOS_CentOS-6/security:shibboleth.repo
	cd -
	sudo yum install libxerces-c-devel libxml-security-c-devel libxmltooling-devel libsaml-devel liblog4shib-devel chrpath boost-devel unixODBC-devel httpd-devel
	sudo yum install xmltooling-schemas opensaml-schemas

now rebuild:

	rpmbuild -ba SPECS/shibboleth.spec

Your new rpm will be in `src/rpm/RPMS/x86_64/shibboleth-2.5.3-1.1.el6.x86_64.rpm`.


# APPENDIX: install patched RPM

### References

- https://wiki.shibboleth.net/confluence/display/SHIB2/NativeSPLinuxSRPMBuild
- https://wiki.shibboleth.net/confluence/display/SHIB2/NativeSPLinuxRPMInstall
- https://github.com/KDK-Alli/NDL-VuFind/wiki/Shibboleth-Installation-on-CentOS-6
- https://shibsp.ntu.ac.uk/confluence/display/Shibboleth/Shibboleth+SP+Install+(Centos)

- http://shibboleth.net/downloads/service-provider/latest/SRPMS/
- http://download.opensuse.org/repositories/security://shibboleth/
- http://download.opensuse.org/repositories/security://shibboleth/CentOS_CentOS-6/x86_64/

- http://wiki.centos.org/HowTos/RebuildSRPM


Install apache web server

	sudo yum update
	sudo yum install httpd

	#sudo /etc/init.d/iptables stop
	sudo /etc/init.d/httpd start
	curl http://localhost/

### Install Shibboleth from repository 

	cd /etc/yum.repos.d/
	sudo wget http://download.opensuse.org/repositories/security://shibboleth/CentOS_CentOS-6/	security:shibboleth.repo
	sudo yum install shibboleth

uncomment partner-metadata inclusion, edit IDP entityID:

	sudo vi /etc/shibboleth/shibboleth2.xml
	sudo vi /etc/shibboleth/partner-metadata.xml
	sudo /etc/init.d/shibd start
	sudo /etc/init.d/httpd restart

	sudo mkdir /var/www/html/secure
	sudo sh -c 'echo secure > /var/www/html/secure/index.html'
	curl http://localhost/secure/

### Install patched RPM

	sudo /etc/init.d/shibd stop
	sudo /etc/init.d/httpd stop
	sudo rpm -ev shibboleth

Download patched shibboleth-2.5.3-1.1.el6.x86_64.rpm 

	sudo rpm -ivh shibboleth-2.5.3-1.1.el6.x86_64.rpm 
	sudo cp /etc/shibboleth/shibboleth2.xml.rpmsave /etc/shibboleth/shibboleth2.xml
	sudo /etc/init.d/shibd start
	sudo /etc/init.d/httpd start
