certificate generation using cryptogen:

    export PATH=${PWD}/../bin:${PWD}:$PATH
    cryptogen generate --config=./organizations/cryptogen/crypto-config-org4.yaml --output="organizations"
    cryptogen generate --config=./organizations/cryptogen/crypto-config-org5.yaml --output="organizations"
    cryptogen generate --config=./organizations/cryptogen/crypto-config-orderer.yaml --output="organizations"

Preparing to start docker network:

    export COMPOSE_PROJECT_NAME=net
    export DOCKER_SOCK=/var/run/docker.sock
    IMAGE_TAG=latest docker-compose -f compose/compose-test-net.yaml -f compose/docker/docker-compose-test-net.yaml up

Creating required channel:

    export PATH=${PWD}/../bin:${PWD}:$PATH
    export FABRIC_CFG_PATH=${PWD}/configtx
    export CHANNEL_NAME=supplychannel

    configtxgen -profile TwoOrgsApplicationGenesis -outputBlock ./channel-artifacts/${CHANNEL_NAME}.block -channelID $CHANNEL_NAME

    configtxgen -inspectBlock ./channel-artifacts/supplychannel.block > dump.json (optional)

    export ORDERER_CA=${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
    export ORDERER_ADMIN_TLS_SIGN_CERT=${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/tls/server.crt
    export ORDERER_ADMIN_TLS_PRIVATE_KEY=${PWD}/organizations/ordererOrganizations/example.com/orderers/orderer.example.com/tls/server.key

    osnadmin channel join --channelID $CHANNEL_NAME --config-block ./channel-artifacts/${CHANNEL_NAME}.block -o localhost:7053 --ca-file "$ORDERER_CA" --client-cert "$ORDERER_ADMIN_TLS_SIGN_CERT" --client-key "$ORDERER_ADMIN_TLS_PRIVATE_KEY"

Perparing to Join channel:

    source ./scripts/setOrgPeerContext.sh 1
    peer channel join -b ./channel-artifacts/supplychannel.block

    source ./scripts/setOrgPeerContext.sh 2
    peer channel join -b ./channel-artifacts/supplychannel.block

Updating the anchor peer:

    source ./scripts/setOrgPeerContext.sh 1
    docker exec cli ./scripts/setAnchorPeer.sh 1 $CHANNEL_NAME

    source ./scripts/setOrgPeerContext.sh 2
    docker exec cli ./scripts/setAnchorPeer.sh 2 $CHANNEL_NAME

Preparing Package chaincode:

    source ./scripts/setFabCarGolangContext.sh
    export FABRIC_CFG_PATH=$PWD/../config/
    export FABRIC_CFG_PATH=${PWD}/configtx
    export CHANNEL_NAME=supplychannel
    export PATH=${PWD}/../bin:${PWD}:$PATH

    source ./scripts/setOrgPeerContext.sh 1
    peer lifecycle chaincode package fabcar.tar.gz --path ${CC_SRC_PATH} --lang ${CC_RUNTIME_LANGUAGE} --label fabcar_${VERSION}

Installing the chaincode:

    peer lifecycle chaincode install fabcar.tar.gz

    source ./scripts/setOrgPeerContext.sh 2
    peer lifecycle chaincode install fabcar.tar.gz

    peer lifecycle chaincode queryinstalled 2>&1 | tee outfile

Approval of the chaincode:

    source ./scripts/setPackageID.sh outfile

    source ./scripts/setOrgPeerContext.sh 1
    peer lifecycle chaincode approveformyorg -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com --tls $CORE_PEER_TLS_ENABLED --cafile $ORDERER_CA --channelID $CHANNEL_NAME --name fabcar --version ${VERSION} --init-required --package-id ${PACKAGE_ID} --sequence ${VERSION}

    source ./scripts/setOrgPeerContext.sh 2
    peer lifecycle chaincode approveformyorg -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com --tls $CORE_PEER_TLS_ENABLED --cafile $ORDERER_CA --channelID $CHANNEL_NAME --name fabcar --version ${VERSION} --init-required --package-id ${PACKAGE_ID} --sequence ${VERSION}

Preparing to committing the chaincode:

    source ./scripts/setPeerConnectionParam.sh 1 2

    peer lifecycle chaincode commit -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com --tls $CORE_PEER_TLS_ENABLED --cafile $ORDERER_CA --channelID $CHANNEL_NAME --name fabcar $PEER_CONN_PARAMS --version ${VERSION} --sequence ${VERSION} --init-required
