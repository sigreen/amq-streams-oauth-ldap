# AMQ Streams (Kafka) / RH-SSO (Keycloak) RBAC Enforcement with OAUTH2 and LDAP

This project demonstrates how to setup and configure an AMQ Streams cluster on OpenShift with client RBAC enforcement (authentication only).

![IC Router RBAC Enforcement with LDAP Diagram](images/logical-arch.png)

## Prerequisites

The following product / OS prerequisities exist:

* OpenJDK 1.8+
* OpenShift 3.11.x
* An LDAP server.  If you don't have an LDAP server, follow this [procedure](https://hub.docker.com/r/larrycai/openldap/) to install a Docker image with a pre-built LDAP server.  Additionally, follow this [tutorial](https://access.redhat.com/documentation/en-US/Fuse_ESB_Enterprise/7.1/html/ActiveMQ_Security_Guide/files/LDAP.html) to configure sample users / groups for your test.

## Procedure

### Setup RH-SSO with custom self-signed keystore / truststore

To get things started, we need to setup a master / slave broker cluster and ensure it works correctly with shared-nothing replication.

1. Follow the [instructions](https://github.com/jbossdemocentral/amq-ha-replicated-demo) for setting-up an AMQ 7 shared-nothing replication master / slave cluster.  You will need to increment the product version to `7.0.3` in the `init.sh` script contained in the project.
2. Update the Acceptor section of `/amq-broker-7.0.3/instances/replicatedMaster/etc/broker.xml` so we're listening on the correct network adapter:

```

      <acceptors>

         <!-- Acceptor for every supported protocol -->
         <acceptor name="artemis">tcp://0.0.0.0:61616?tcpSendBufferSize=1048576;tcpReceiveBufferSize=1048576;protocols=CORE,AMQP,STOMP,HORNETQ,MQTT,OPENWIRE;useEpoll=true;amqpCredits=1000;amqpLowCredits=300</acceptor>

         <!-- AMQP Acceptor.  Listens on default AMQP port for AMQP traffic.-->
         <acceptor name="amqp">tcp://0.0.0.0:5673?tcpSendBufferSize=1048576;tcpReceiveBufferSize=1048576;protocols=AMQP;useEpoll=true;amqpCredits=1000;amqpMinCredits=300</acceptor>

         <!-- STOMP Acceptor. -->
         <acceptor name="stomp">tcp://0.0.0.0:61613?tcpSendBufferSize=1048576;tcpReceiveBufferSize=1048576;protocols=STOMP;useEpoll=true</acceptor>

         <!-- HornetQ Compatibility Acceptor.  Enables HornetQ Core and STOMP for legacy HornetQ clients. -->
         <acceptor name="hornetq">tcp://0.0.0.0:5445?protocols=HORNETQ,STOMP;useEpoll=true</acceptor>

         <!-- MQTT Acceptor -->
         <acceptor name="mqtt">tcp://0.0.0.0:1883?tcpSendBufferSize=1048576;tcpReceiveBufferSize=1048576;protocols=MQTT;useEpoll=true</acceptor>

      </acceptors>
```
3. Do the same for `/amq-broker-7.0.3/instances/replicatedSlave/etc/broker.xml`:

```

      <acceptors>

         <!-- useEpoll means: it will use Netty epoll if you are on a system (Linux) that supports it -->
         <!-- amqpCredits: The number of credits sent to AMQP producers -->
         <!-- amqpLowCredits: The server will send the # credits specified at amqpCredits at this low mark -->

         <!-- Acceptor for every supported protocol -->
         <acceptor name="artemis">tcp://0.0.0.0:61716?tcpSendBufferSize=1048576;tcpReceiveBufferSize=1048576;protocols=CORE,AMQP,STOMP,HORNETQ,MQTT,OPENWIRE;useEpoll=true;amqpCredits=1000;amqpLowCredits=300</acceptor>

         <!-- AMQP Acceptor.  Listens on default AMQP port for AMQP traffic.-->
         <acceptor name="amqp">tcp://0.0.0.0:5772?tcpSendBufferSize=1048576;tcpReceiveBufferSize=1048576;protocols=AMQP;useEpoll=true;amqpCredits=1000;amqpMinCredits=300</acceptor>

         <!-- STOMP Acceptor. -->
         <acceptor name="stomp">tcp://0.0.0.0:61713?tcpSendBufferSize=1048576;tcpReceiveBufferSize=1048576;protocols=STOMP;useEpoll=true</acceptor>

         <!-- HornetQ Compatibility Acceptor.  Enables HornetQ Core and STOMP for legacy HornetQ clients. -->
         <acceptor name="hornetq">tcp://0.0.0.0:5545?protocols=HORNETQ,STOMP;useEpoll=true</acceptor>

         <!-- MQTT Acceptor -->
         <acceptor name="mqtt">tcp://0.0.0.0:1983?tcpSendBufferSize=1048576;tcpReceiveBufferSize=1048576;protocols=MQTT;useEpoll=true</acceptor>

      </acceptors>
```
4. Update `/amq-broker-7.0.3/instances/replicatedMaster/etc/bootstrap.xml` and `/amq-broker-7.0.3/instances/replicatedMaster/etc/bootstrap.xml` so that the web bind address is listening on the correct host e.g. `<web bind="http://0.0.0.0:XXXX" path="web">`
5. Test your setup as described in the instructions, ensuring that master / slave failover occurs as expected.  Ensure you can access both HawtIO consoles (on port 8161 + 8261).
6. For an additional test, try sending an AMQP message to either port 5673 (master) or 5772 (slave) depending on which broker is active.  You can use the following command for this:
```
qsend amqp://127.0.0.1:5673/haQueue -m Abc
```
If you browse for the message via HawtIO, you'll notice that Durable=false.  This means the message will disappear during failover and won't be replicated.  The test in step 5. was Durable=true, therefore the message should have been replicated


### Test LDAP connectivity with your broker cluster

Although we want to enforce RBAC at the IC router level, it's important to test out LDAP connectivity using our users / groups.  For this exercise, we'll use a single standalone broker instead of the cluster.

1. Create a new broker instance using the following command:

```
./bin/artemis create instances/adbroker
```

Use admin/admin for credentials and type 'Y' for anonymous access

2. Replace the `etc/broker.xml`, `etc/bootstrap.xml` and `etc/login.config` files with those contained in this project (found in `/conf/broker/ldap`)

3. Update the HawtIO role in artemis.profile with `-Dhawtio.role=admins`

4. Startup the broker using `bin/artemis run`

5. Test login to the HawtIO console using both your admin and user credentials.  Only the admin user should be authorized to enter the console.

6. Test sending messages using a known username e.g. jdoe:

```
./bin/artemis producer --message-count 10 --url "tcp://127.0.0.1:61616" --destination queue://defQueue --user jdoe --password sunflower
```

7. Test consuming messages using a known username e.g. jdoe:

```
./artemis consumer --message-count 10 --url "tcp://127.0.0.1:61616" --destination queue://defQueue --user jdoe --password sunflower
```

### Setup Interconnect Router for SSL Client connections

This procedure sets up PLAIN SSL for clients connecting via the router.  The router contains autolinks to a couple of static queues (queue.foo and queue.grs).  The router will loadbalance between the active/passive master/slave pair, depending on which is active.

1. Install the Interconnect Router on RHEL 7 as per the [instructions](https://access.redhat.com/documentation/en-us/red_hat_jboss_amq/7.0/html/using_amq_interconnect/installation)

2. Follow the instructions to setup an SSL profile from the [documentation](https://access.redhat.com/documentation/en-us/red_hat_jboss_amq/7.0/html/using_amq_interconnect/security#setting_up_ssl_for_encryption_and_authentication).  I used openssl to generate my cacerts and keystores: `openssl req -newkey rsa:2048 -nodes -keyout key.pem -x509 -days 365 -out certificate.pem`.

3. Copy the `/conf/ic/qdrouterd.conf` to your RHEL server.

4. Update the qdrouterd.conf sslProfile section to include the correct path to your keys and keystores created in step 2.

5. Startup your master/slave brokers.  You should see the following in the router log indicating the connection to the master broker is active:

```
Fri Nov 10 13:17:52 2017 ROUTER_CORE (info) Auto Link Activated 'autoLink/0' on connection master
Fri Nov 10 13:17:52 2017 ROUTER_CORE (info) Auto Link Activated 'autoLink/1' on connection master
Fri Nov 10 13:17:52 2017 ROUTER_CORE (info) Auto Link Activated 'autoLink/2' on connection master
Fri Nov 10 13:17:52 2017 ROUTER_CORE (info) Auto Link Activated 'autoLink/3' on connection master
```

6. Send a test message to the master node by issuing the following command:

```
qsend amqps://127.0.0.1:5672/queue.foo -m Master
```

7. Login to the Master node HawtIO app.  You should see the message in the queue.foo by browsing.

8.  Failover to the Slave broker by killing the Master.  The following should appear in the router logs:

```
Fri Nov 10 13:21:34 2017 ROUTER_CORE (info) Auto Link Deactivated 'autoLink/0' on connection master
Fri Nov 10 13:21:34 2017 ROUTER_CORE (info) Auto Link Deactivated 'autoLink/1' on connection master
Fri Nov 10 13:21:34 2017 ROUTER_CORE (info) Auto Link Deactivated 'autoLink/2' on connection master
Fri Nov 10 13:21:34 2017 ROUTER_CORE (info) Auto Link Deactivated 'autoLink/3' on connection master
Fri Nov 10 13:21:34 2017 ROUTER_CORE (info) Auto Link Activated 'autoLink/4' on connection slave
Fri Nov 10 13:21:34 2017 ROUTER_CORE (info) Auto Link Activated 'autoLink/5' on connection slave
Fri Nov 10 13:21:34 2017 ROUTER_CORE (info) Auto Link Activated 'autoLink/6' on connection slave
Fri Nov 10 13:21:34 2017 ROUTER_CORE (info) Auto Link Activated 'autoLink/7' on connection slave
```

9. Send a test message to the slave node by issuing the following command:

```
qsend amqps://127.0.0.1:5672/queue.foo -m Slave
```

### Setup LDAP connection to IC Router

1. Install the necessary cyrus-sasl libraries required by the router and also set up the listener.  The
`cyrus-sasl` and `cyrus-sasl-plain` libraries need to be installed (`yum install cyrus-sasl cyrus-sasl-plain`). The client sends the user name and password in clear to the Router. Hence, in a production environment, this client-router communication needs to be done over a TLS connection.

Setup the following listener in the router's config file (this is the config file you use to start the router) - port number can be to your choosing:
```
listener {
    addr: 0.0.0.0
    port: 15677
    role: normal
    authenticatePeer: yes
    saslMechanisms: PLAIN
}
```
Notice that this sets up the Router listener to use PLAIN as the the only mechanism used to communicate with the client. PLAIN is used because the user name and password needs to be passed in clear to the PAM authentication module.

2. Configure Dispatch Router to use a program called saslauthd to connect to SSSD via PAM (Pluggable Authentication Modules).
The sasl config file is usually found in the /etc/sasl2/qdrouterd.conf file. Add the following properties to this file:
```
pwcheck_method: saslauthd
auxprop_plugin:pam
mech_list: PLAIN
```
Notice that the `pwcheck_method` is set to `saslauthd` which is a program that is installed as part of the `cyrus-sasl` library installation (`yum install cyrus-sasl`). `saslauthd` is used to supply the user name and password to the pam module. Notice here that the `auxprop_plugin` is set to pam which instructs cyrus-sasl to enable authentication using PAM via saslauthd. The `mech_list` is set to `PLAIN`  to match the mech_list in step 1.

Make sure that the `/etc/sysconfig/saslauthd` file contains
`MECH=pam`

This enables saslauthd to use PAM.

3. Configure PAM
Every application has to have its own config file in the `/etc/pam.d/` folder. The config file for Qpid Dispatch is called amqp. Open or create the `/etc/pam.d/amqp` file and add the following to it -
`#%PAM-1.0
auth    required  pam_securetty.so
auth    required pam_nologin.so
account required pam_unix.so
session required pam_unix.so
`
What each of the above lines means is explained in detail in Section 2.2 [here](docs/pam-step-3.pdf)
The above PAM configuration is not production grade but works in my situation.

4. Configure PAM service on SSSD - This instructs PAM to use SSSD to retrieve user information and is dealt with in detail in Section 30.3.2 [here](docs/pam-step-4.pdf) (I followed all steps in this link)
(SSSD is already installed on RHEL 7 machines)

5. Install Redhat IdM (Active Directory(AD) and any LDAP server can be used instead of Redhat IdM) in Section 2.3 [here](docs/pam-step-5.pdf).
Skip this step if you are not using IdM

6. Ask SSSD to discover AD or IdM services `realm discover test.example.com` - This instructs SSSD to discover a directory service running on the host test.example.com.
Use - `realm discover --server-software=active-directory test.example.com` - to discovery AD
If the discovery is successful, to join the system SSSD to an identity domain, use the realm join command and specify the domain name:
```
realm join test.example.com
```
SSSD will successfully join the realm. Test the whole setup with the testsaslauthd program that comes as part of the cyrus-sasl installation
```
[root@amq01 /]# testsaslauthd -u test -p test -r LAB.ENG.RDU2.REDHAT.COM -s amqp
0: OK "Success."
```
If you don't get "OK", you are in trouble.

To troubleshoot watch the output of `journalctl -f` as you run `testsaslauthd`

7. In summary, now you have enabled SASL in Qpid Dispatch Router to talk to PAM which in turn talks to SSSD which in turn can talk to any directory service like AD, LDAP or IdM.

8. Finally start the router and run qdstat

`reset; PN_TRACE_FRM=1 qdstat -b amqps://127.0.0.1:15677 -c --sasl-mechanisms=PLAIN --sasl-username=max@LAB.ENG.RDU2.REDHAT.COM --sasl-password=Abcd1234`

Note here again the the saslMechanisms is set to PLAIN and the realm (LAB.ENG.RDU2.REDHAT.COM) is included as part of the user name.
