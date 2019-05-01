# MultiOrgSetup

Hyperledger fabric multiple organization setup.


## Prerequisites

```
$ curl -sSL http://bit.ly/2ysbOFE | bash -s 1.3.0

$ cd fabric-samples/bin
```
##### Install binaries

```
$ sudo cp peer orderer cryptogen configtxgen configtxlator /usr/bin/
```


## Cloning the repository

```

```


## MultiOrg Setup Walkthrough


Our multi org network will have two organizations:

1) Mphasis
2) Dxc

The walkthrough focuses on the following things

1) Creating network artifacts.
2) Creating channel.
3) Joining peers to the channel.
4) Adding a new peer to the organization.

### Creating network artifacts

Create crypto-config.yaml 

Add the following code to the crypto-config.yaml file

```
OrdererOrgs:

- Name: Orderer

  Domain: example.com

  Specs:

    - Hostname: orderer

PeerOrgs:

- Name: Mphasis

  Domain: mphasis.example.com

  Template:

    Count: 2

  Users:

    Count: 1

- Name: Dxc

  Domain: dxc.example.com

  Template:

    Count: 2

  Users:

    Count: 1

```

##### Generate certificates

```

$ cd multiorgsetup
$ cryptogen generate --config=./crypto-config.yaml

```


##### Create configtx.yaml file

```
Capabilities:
    # Channel capabilities apply to both the orderers and the peers and must be
    # supported by both.  Set the value of the capability to true to require it.
    Global: &ChannelCapabilities
        V1_1: true

    # Orderer capabilities apply only to the orderers, and may be safely
    # manipulated without concern for upgrading peers.  Set the value of the
    # capability to true to require it.
    Orderer: &OrdererCapabilities
        V1_1: true

    # Application capabilities apply only to the peer network, and may be safely
    # manipulated without concern for upgrading orderers.  Set the value of the
    # capability to true to require it.
    Application: &ApplicationCapabilities
        V1_2: true


Organizations:

    # SampleOrg defines an MSP using the sampleconfig.  It should never be used
    # in production but may be used as a template for other definitions
    - &OrdererOrg
        # DefaultOrg defines the organization which is used in the sampleconfig
        # of the fabric.git development environment
        Name: OrdererOrg

        # ID to load the MSP definition as
        ID: OrdererMSP

        # MSPDir is the filesystem path which contains the MSP configuration
        MSPDir: crypto-config/ordererOrganizations/example.com/msp

    - &Mphasis
        # DefaultOrg defines the organization which is used in the sampleconfig
        # of the fabric.git development environment
        Name: MphasisMSP

        # ID to load the MSP definition as
        ID: MphasisMSP

        MSPDir: crypto-config/peerOrganizations/mphasis.example.com/msp

        AnchorPeers:
            # AnchorPeers defines the location of peers which can be used
            # for cross org gossip communication.  Note, this value is only
            # encoded in the genesis block in the Application section context
            - Host: peer0.mphasis.example.com
              Port: 7051
    - &Dxc
        # DefaultOrg defines the organization which is used in the sampleconfig
        # of the fabric.git development environment
        Name: DxcMSP

        # ID to load the MSP definition as
        ID: DxcMSP

        MSPDir: crypto-config/peerOrganizations/dxc.example.com/msp

        AnchorPeers:
            # AnchorPeers defines the location of peers which can be used
            # for cross org gossip communication.  Note, this value is only
            # encoded in the genesis block in the Application section context
            - Host: peer0.dxc.example.com
              Port: 8051

################################################################################
#
#   SECTION: Orderer
#
#   - This section defines the values to encode into a config transaction or
#   genesis block for orderer related parameters
#
################################################################################
Orderer: &OrdererDefaults

    # Orderer Type: The orderer implementation to start
    # Available types are "solo" and "kafka"
    OrdererType: solo

    Addresses:
        - 127.0.0.1:7050

    # Batch Timeout: The amount of time to wait before creating a batch
    BatchTimeout: 8s

    # Batch Size: Controls the number of messages batched into a block
    BatchSize:

        # Max Message Count: The maximum number of messages to permit in a batch
        MaxMessageCount: 10

        # Absolute Max Bytes: The absolute maximum number of bytes allowed for
        # the serialized messages in a batch.
        AbsoluteMaxBytes: 99 MB

        # Preferred Max Bytes: The preferred maximum number of bytes allowed for
        # the serialized messages in a batch. A message larger than the preferred
        # max bytes will result in a batch larger than preferred max bytes.
        PreferredMaxBytes: 512 KB

    Kafka:
        # Brokers: A list of Kafka brokers to which the orderer connects
        # NOTE: Use IP:port notation
        Brokers:
            - 127.0.0.1:9092

    # Organizations is the list of orgs which are defined as participants on
    # the orderer side of the network
    Organizations:

################################################################################
#
#   SECTION: Application
#
#   - This section defines the values to encode into a config transaction or
#   genesis block for application related parameters
#
################################################################################
Application: &ApplicationDefaults

    # Organizations is the list of orgs which are defined as participants on
    # the application side of the network
    Organizations:

Profiles:

    TwoOrgOrdererGenesis:
        Orderer:
            <<: *OrdererDefaults
            Organizations:
                - *OrdererOrg
            Capabilities:
                <<: *OrdererCapabilities
        Consortiums:
            SampleConsortium:
                Organizations:
                    - *Mphasis
                    - *Dxc
        Capabilities:
            <<: *ChannelCapabilities
    TwoOrgChannel:
        Consortium: SampleConsortium
        Application:
            <<: *ApplicationDefaults
            Organizations:
                - *Mphasis
                - *Dxc
            Capabilities:
                <<: *ApplicationCapabilities


```


##### Generate channel-artifacts

```
$ mkdir channel-artifacts

$ configtxgen -profile TwoOrgOrdererGenesis -outputBlock ./channel-artifacts/genesis.block

```

###### Create channel configuration transactions

```
$ export CHANNEL_NAME=mychannel
$ configtxgen -profile TwoOrgChannel -outputCreateChannelTx ./channel-artifacts/channel.tx -channelID $CHANNEL_NAME

```

###### Define anchor peer for mphasis on the channel

```
$ configtxgen -profile TwoOrgChannel -outputAnchorPeersUpdate ./channel-artifacts/MphasisMSPanchors.tx -channelID $CHANNEL_NAME -asOrg MphasisMSP
```

###### Define anchor peer for dxc on the channel

```
$ configtxgen -profile TwoOrgChannel -outputAnchorPeersUpdate ./channel-artifacts/DxcMSPanchors.tx -channelID $CHANNEL_NAME -asOrg DxcMSP
```



###### Start the network

####### Docker compose file for mphasis

```
version: '2'

networks:
  multiorg:

services:
  ca.example.com:
    image: hyperledger/fabric-ca
    environment:
      - FABRIC_CA_HOME=/etc/hyperledger/fabric-ca-server
      - FABRIC_CA_SERVER_CA_NAME=ca.example.com
      - FABRIC_CA_SERVER_CA_CERTFILE=/etc/hyperledger/fabric-ca-server-config/ca.mphasis.example.com-cert.pem
      - FABRIC_CA_SERVER_CA_KEYFILE=/etc/hyperledger/fabric-ca-server-config/e3ac34db299c54e9950a17901fb760017e6b622c8df069df06583bf8384b4037_sk
    ports:
      - "7054:7054"
    command: sh -c 'fabric-ca-server start -b admin:adminpw -d'
    volumes:
      - ./crypto-config/peerOrganizations/mphasis.example.com/ca/:/etc/hyperledger/fabric-ca-server-config
    container_name: ca.example.com
    networks:
      - multiorg

  orderer.example.com:
    container_name: orderer.example.com
    image: hyperledger/fabric-orderer
    environment:
      - ORDERER_GENERAL_LOGLEVEL=debug
      - ORDERER_GENERAL_LISTENADDRESS=0.0.0.0
      - ORDERER_GENERAL_GENESISMETHOD=file
      - ORDERER_GENERAL_GENESISFILE=/etc/hyperledger/configtx/genesis.block
      - ORDERER_GENERAL_LOCALMSPID=OrdererMSP
      - ORDERER_GENERAL_LOCALMSPDIR=/etc/hyperledger/msp/orderer/msp
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric/orderer
    command: orderer
    ports:
      - 7050:7050
    volumes:
        - ./channel-artifacts/:/etc/hyperledger/configtx
        - ./crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/:/etc/hyperledger/msp/orderer
        - ./crypto-config/peerOrganizations/mphasis.example.com/peers/peer0.mphasis.example.com/:/etc/hyperledger/msp/peerMphasis
    networks:
      - multiorg

  couchdb0:
    container_name: couchdb0
    image: hyperledger/fabric-couchdb
    # Populate the COUCHDB_USER and COUCHDB_PASSWORD to set an admin user and password
    # for CouchDB.  This will prevent CouchDB from operating in an "Admin Party" mode.
    environment:
      - COUCHDB_USER=
      - COUCHDB_PASSWORD=
    ports:
      - 5984:5984
    networks:
      - multiorg

  peer0.mphasis.example.com:
    container_name: peer0.mphasis.example.com
    image: hyperledger/fabric-peer
    environment:
      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
      - CORE_PEER_ID=peer0.mphasis.example.com
      - CORE_LOGGING_PEER=debug
      - CORE_CHAINCODE_LOGGING_LEVEL=DEBUG
      - CORE_PEER_LOCALMSPID=MphasisMSP
      - CORE_PEER_MSPCONFIGPATH=/etc/hyperledger/msp/peer/
      - CORE_PEER_ADDRESS=peer0.mphasis.example.com:7051
      # # the following setting starts chaincode containers on the same
      # # bridge network as the peers
      # # https://docs.docker.com/compose/networking/
      - CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=mphasis_multiorg
      - CORE_LEDGER_STATE_STATEDATABASE=CouchDB
      - CORE_LEDGER_STATE_COUCHDBCONFIG_COUCHDBADDRESS=couchdb0:5984
      # The CORE_LEDGER_STATE_COUCHDBCONFIG_USERNAME and CORE_LEDGER_STATE_COUCHDBCONFIG_PASSWORD
      # provide the credentials for ledger to connect to CouchDB.  The username and password must
      # match the username and password set for the associated CouchDB.
      - CORE_LEDGER_STATE_COUCHDBCONFIG_USERNAME=
      - CORE_LEDGER_STATE_COUCHDBCONFIG_PASSWORD=
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric
    command: peer node start
    # command: peer node start --peer-chaincodedev=true
    ports:
      - 7051:7051
      - 7053:7053
    volumes:
        - /var/run/:/host/var/run/
        - ./crypto-config/peerOrganizations/mphasis.example.com/peers/peer0.mphasis.example.com/msp:/etc/hyperledger/msp/peer
        - ./crypto-config/peerOrganizations/mphasis.example.com/users:/etc/hyperledger/msp/users
        - ./channel-artifacts:/etc/hyperledger/configtx
    depends_on:
      - orderer.example.com
      - couchdb0
    networks:
      - multiorg

  mphasiscli:
    container_name: mphasiscli
    image: hyperledger/fabric-tools
    tty: true
    environment:
      - GOPATH=/opt/gopath
      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
      - CORE_LOGGING_LEVEL=DEBUG
      - CORE_PEER_ID=mphasiscli
      - CORE_PEER_ADDRESS=peer0.mphasis.example.com:7051
      - CORE_PEER_LOCALMSPID=MphasisMSP
      - CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/mphasis.example.com/users/Admin@mphasis.example.com/msp
      - CORE_CHAINCODE_KEEPALIVE=10
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer
    command: /bin/bash
    volumes:
        - /var/run/:/host/var/run/
        - ./chaincode/:/opt/gopath/src/github.com/
        - ./crypto-config:/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/
    networks:
        - multiorg


```



###### Docker compose file for dxc


```
version: '2'

networks:
  multiorg:

services:
  couchdb1:
    container_name: couchdb1
    image: hyperledger/fabric-couchdb
    # Populate the COUCHDB_USER and COUCHDB_PASSWORD to set an admin user and password
    # for CouchDB.  This will prevent CouchDB from operating in an "Admin Party" mode.
    environment:
      - COUCHDB_USER=
      - COUCHDB_PASSWORD=
    ports:
      - 6984:5984
    networks:
      - multiorg

  peer0.dxc.example.com:
    container_name: peer0.dxc.example.com
    image: hyperledger/fabric-peer
    environment:
      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
      - CORE_PEER_ID=peer0.dxc.example.com
      - CORE_LOGGING_PEER=debug
      - CORE_CHAINCODE_LOGGING_LEVEL=DEBUG
      - CORE_PEER_LOCALMSPID=DxcMSP
      - CORE_PEER_MSPCONFIGPATH=/etc/hyperledger/msp/peer/
      - CORE_PEER_ADDRESS=peer0.dxc.example.com:7051
      # # the following setting starts chaincode containers on the same
      # # bridge network as the peers
      # # https://docs.docker.com/compose/networking/
      - CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=dxc_basic
      - CORE_LEDGER_STATE_STATEDATABASE=CouchDB
      - CORE_LEDGER_STATE_COUCHDBCONFIG_COUCHDBADDRESS=couchdb1:6984
      # The CORE_LEDGER_STATE_COUCHDBCONFIG_USERNAME and CORE_LEDGER_STATE_COUCHDBCONFIG_PASSWORD
      # provide the credentials for ledger to connect to CouchDB.  The username and password must
      # match the username and password set for the associated CouchDB.
      - CORE_LEDGER_STATE_COUCHDBCONFIG_USERNAME=
      - CORE_LEDGER_STATE_COUCHDBCONFIG_PASSWORD=
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric
    command: peer node start
    # command: peer node start --peer-chaincodedev=true
    ports:
      - 8051:7051
      - 8053:7053
    volumes:
        - /var/run/:/host/var/run/
        - ./crypto-config/peerOrganizations/dxc.example.com/peers/peer0.dxc.example.com/msp:/etc/hyperledger/msp/peer
        - ./crypto-config/peerOrganizations/dxc.example.com/users:/etc/hyperledger/msp/users
        - ./channel-artifacts:/etc/hyperledger/configtx
    depends_on:
      - couchdb1
    networks:
      - multiorg

  dxccli:
    container_name: dxccli
    image: hyperledger/fabric-tools
    tty: true
    environment:
      - GOPATH=/opt/gopath
      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
      - CORE_LOGGING_LEVEL=DEBUG
      - CORE_PEER_ID=dxccli
      - CORE_PEER_ADDRESS=peer0.dxc.example.com:8051
      - CORE_PEER_LOCALMSPID=DxcMSP
      - CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/dxc.example.com/users/Admin@dxc.example.com/msp
      - CORE_CHAINCODE_KEEPALIVE=10
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer
    command: /bin/bash
    volumes:
        - /var/run/:/host/var/run/
        - ./chaincode/:/opt/gopath/src/github.com/
        - ./crypto-config:/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/
    networks:
        - multiorg


```


###### Start mphasis and dxc networks

```

$ docker-compose -f docker-compose-mphasis.yaml up

$ docker-compose -f docker-compose-dxc.yaml up

```


## Create channel

```
$ docker exec -e "CORE_PEER_LOCALMSPID=MphasisMSP" -e "CORE_PEER_MSPCONFIGPATH=/etc/hyperledger/msp/users/Admin@mphasis.example.com/msp" peer0.mphasis.example.com peer channel create -o orderer.example.com:7050 -c mychannel -f /etc/hyperledger/configtx/channel.tx

```

## Join peer0 of mphasis to the channel

```

$  docker exec -e "CORE_PEER_LOCALMSPID=MphasisMSP" -e "CORE_PEER_MSPCONFIGPATH=/etc/hyperledger/msp/users/Admin@mphasis.example.com/msp" peer0.mphasis.example.com peer channel join -b mychannel.block

```

## Join peer0 of dxc to the channel

```

$ docker exec -e "CORE_PEER_LOCALMSPID=DxcMSP" -e "CORE_PEER_MSPCONFIGPATH=/etc/hyperledger/msp/users/Admin@dxc.example.com/msp" peer0.dxc.example.com peer channel fetch config -o orderer.example.com:7050 -c mychannel

$ docker exec -e "CORE_PEER_LOCALMSPID=DxcMSP" -e "CORE_PEER_MSPCONFIGPATH=/etc/hyperledger/msp/users/Admin@dxc.example.com/msp" peer0.dxc.example.com peer channel join -b mychannel_config.block

```



## Add new peer (peer2.dxc.example.com) to dxc organization

### update crypto-config.yaml

> NOTE: We just incremented count to 3

```
PeerOrgs:

- Name: Dxc

  Domain: dxc.example.com

  Template:

    Count: 3

  Users:

    Count: 1

```


###### Extend crytpo config file

```
$ cryptogen extend --config=./crypto-config.yaml
```



###### Create new docker compose file for peer1 and peer2 of dxc

```
version: '2'

networks:
  multiorg2:

services:
  couchdb2:
    container_name: couchdb2
    image: hyperledger/fabric-couchdb
    # Populate the COUCHDB_USER and COUCHDB_PASSWORD to set an admin user and password
    # for CouchDB.  This will prevent CouchDB from operating in an "Admin Party" mode.
    environment:
      - COUCHDB_USER=
      - COUCHDB_PASSWORD=
    ports:
      - 7984:5984
    networks:
      - multiorg2

  couchdb3:
    container_name: couchdb3
    image: hyperledger/fabric-couchdb
    # Populate the COUCHDB_USER and COUCHDB_PASSWORD to set an admin user and password
    # for CouchDB.  This will prevent CouchDB from operating in an "Admin Party" mode.
    environment:
      - COUCHDB_USER=
      - COUCHDB_PASSWORD=
    ports:
      - 8984:5984
    networks:
      - multiorg2

  peer1.dxc.example.com:
    container_name: peer1.dxc.example.com
    image: hyperledger/fabric-peer
    environment:
      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
      - CORE_PEER_ID=peer1.dxc.example.com
      - CORE_LOGGING_PEER=debug
      - CORE_CHAINCODE_LOGGING_LEVEL=DEBUG
      - CORE_PEER_LOCALMSPID=DxcMSP
      - CORE_PEER_MSPCONFIGPATH=/etc/hyperledger/msp/peer/
      - CORE_PEER_ADDRESS=peer1.dxc.example.com:7051
      # # the following setting starts chaincode containers on the same
      # # bridge network as the peers
      # # https://docs.docker.com/compose/networking/
      - CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=dxc_multiorg2
      - CORE_LEDGER_STATE_STATEDATABASE=CouchDB
      - CORE_LEDGER_STATE_COUCHDBCONFIG_COUCHDBADDRESS=couchdb2:5984
      # The CORE_LEDGER_STATE_COUCHDBCONFIG_USERNAME and CORE_LEDGER_STATE_COUCHDBCONFIG_PASSWORD
      # provide the credentials for ledger to connect to CouchDB.  The username and password must
      # match the username and password set for the associated CouchDB.
      - CORE_LEDGER_STATE_COUCHDBCONFIG_USERNAME=
      - CORE_LEDGER_STATE_COUCHDBCONFIG_PASSWORD=
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric
    command: peer node start
    # command: peer node start --peer-chaincodedev=true
    ports:
      - 9051:8051
      - 9053:8053
    volumes:
        - /var/run/:/host/var/run/
        - ./crypto-config/peerOrganizations/dxc.example.com/peers/peer1.dxc.example.com/msp:/etc/hyperledger/msp/peer
        - ./crypto-config/peerOrganizations/dxc.example.com/users:/etc/hyperledger/msp/users
        - ./channel-artifacts:/etc/hyperledger/configtx
    depends_on:
      - couchdb2
    networks:
      - multiorg2

  peer2.dxc.example.com:
    container_name: peer2.dxc.example.com
    image: hyperledger/fabric-peer
    environment:
      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
      - CORE_PEER_ID=peer2.dxc.example.com
      - CORE_LOGGING_PEER=debug
      - CORE_CHAINCODE_LOGGING_LEVEL=DEBUG
      - CORE_PEER_LOCALMSPID=DxcMSP
      - CORE_PEER_MSPCONFIGPATH=/etc/hyperledger/msp/peer/
      - CORE_PEER_ADDRESS=peer2.dxc.example.com:7051
      # # the following setting starts chaincode containers on the same
      # # bridge network as the peers
      # # https://docs.docker.com/compose/networking/
      - CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=dxc_multiorg2
      - CORE_LEDGER_STATE_STATEDATABASE=CouchDB
      - CORE_LEDGER_STATE_COUCHDBCONFIG_COUCHDBADDRESS=couchdb3:5984
      # The CORE_LEDGER_STATE_COUCHDBCONFIG_USERNAME and CORE_LEDGER_STATE_COUCHDBCONFIG_PASSWORD
      # provide the credentials for ledger to connect to CouchDB.  The username and password must
      # match the username and password set for the associated CouchDB.
      - CORE_LEDGER_STATE_COUCHDBCONFIG_USERNAME=
      - CORE_LEDGER_STATE_COUCHDBCONFIG_PASSWORD=
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric
    command: peer node start
    # command: peer node start --peer-chaincodedev=true
    ports:
      - 9951:9051
      - 9953:9053
    volumes:
        - /var/run/:/host/var/run/
        - ./crypto-config/peerOrganizations/dxc.example.com/peers/peer2.dxc.example.com/msp:/etc/hyperledger/msp/peer
        - ./crypto-config/peerOrganizations/dxc.example.com/users:/etc/hyperledger/msp/users
        - ./channel-artifacts:/etc/hyperledger/configtx
    depends_on:
      - couchdb3
    networks:
      - multiorg2


```


###### Joine peer 1 and peer 2 to the channel

> NOTE: we did not stop the existing containers.


```
$  docker exec -e "CORE_PEER_LOCALMSPID=DxcMSP" -e "CORE_PEER_MSPCONFIGPATH=/etc/hyperledger/msp/users/Admin@dxc.example.com/msp" peer1.dxc.example.com peer channel fetch config -o {IPADDRESS}:7050 -c mychannel
```

```
$  docker exec -e "CORE_PEER_LOCALMSPID=DxcMSP" -e "CORE_PEER_MSPCONFIGPATH=/etc/hyperledger/msp/users/Admin@dxc.example.com/msp" peer1.dxc.example.com peer channel join -b mychannel_config.block
```


```

$ docker exec -e "CORE_PEER_LOCALMSPID=DxcMSP" -e "CORE_PEER_MSPCONFIGPATH=/etc/hyperledger/msp/users/Admin@dxc.example.com/msp" peer2.dxc.example.com peer channel fetch config -o  {IPADDRESS}:7050 -c mychannel 

```

```
$ docker exec -e "CORE_PEER_LOCALMSPID=DxcMSP" -e "CORE_PEER_MSPCONFIGPATH=/etc/hyperledger/msp/users/Admin@dxc.example.com/msp" peer2.dxc.example.com peer channel join -b mychannel_config.block

```